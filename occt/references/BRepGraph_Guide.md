# BRepGraph 指南

> BRepGraph 是基于关联表（incidence-table）拓扑后端的门面 API，为 TopoDS/BRep 形状提供稳定的算法接口。
>
> **命名说明**：底层存储命名空间 `BRepGraphInc` 中的 **Inc** 是 **Incidence**（关联/入射）的缩写，指拓扑学中的 incidence table（关联表）数据模型——用关联关系而非包含关系来表达拓扑结构。

---

## 目录

- [1. 架构概览](#1-架构概览)
- [2. 数据模型](#2-数据模型)
  - [2.1 三层分工：TopoDS、BRep_T*、BRepGraph](#21-三层分工topodsbrep_tbrepgrraph)
  - [2.2 定义（Definition）](#22-定义definition)
  - [2.3 引用（Reference）](#23-引用reference)
  - [2.4 表示（Representation）](#24-表示representation)
  - [2.5 经典模型对照：边与 PCurve](#25-经典模型对照边与-pcurve)
- [3. 标识类型](#3-标识类型)
  - [3.1 NodeId](#31-nodeid)
  - [3.2 RefId](#32-refid)
  - [3.3 RepId](#33-repid)
  - [3.4 UID 与 RefUID](#34-uid-与-refuid)
  - [3.5 VersionStamp](#35-versionstamp)
  - [3.6 四种标识总结](#36-四种标识总结)
- [4. 存储后端与反向索引](#4-存储后端与反向索引)
  - [4.1 BRepGraphInc_Storage](#41-brepgraphinc_storage)
  - [4.2 BRepGraphInc_ReverseIndex](#42-brepgraphinc_reverseindex)
  - [4.3 Edge→Faces 的派生原理](#43-edgefaces-的派生原理)
- [5. 公共 API：分组视图](#5-公共-api分组视图)
  - [5.1 TopoView](#51-topoview)
  - [5.2 RefsView](#52-refsview)
  - [5.3 UIDsView](#53-uidsview)
  - [5.4 ShapesView](#54-shapesview)
  - [5.5 CacheView](#55-cacheview)
  - [5.6 BuilderView](#56-builderview)
- [6. 构建与算法](#6-构建与算法)
- [7. 变异追踪](#7-变异追踪)
  - [7.1 MutGuard](#71-mutguard)
  - [7.2 DeferredScope](#72-deferredscope)
  - [7.3 History / HistoryRecord](#73-history--historyrecord)
- [8. 遍历器](#8-遍历器)
- [9. 几何访问：BRepGraph_Tool](#9-几何访问brepgrraph_tool)
- [10. 层与缓存](#10-层与缓存)
- [11. 迭代器与并行策略](#11-迭代器与并行策略)
- [12. 拓扑与装配关系图](#12-拓扑与装配关系图)

---

## 1. 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                      算法层                              │
│  (缝合/修复/压缩/去重/拷贝/变换/验证)                     │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              BRepGraph 门面                              │
│  ┌──────────┬──────────┬──────────┬──────────────┐      │
│  │ TopoView │ RefsView │UIDsView  │ ShapesView   │      │
│  ├──────────┼──────────┼──────────┼──────────────┤      │
│  │CacheView │BuilderView│History  │LayerRegistry │      │
│  └──────────┴──────────┴──────────┴──────────────┘      │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              BRepGraph_Data                              │
│  ┌──────────────────────────────────────────────┐        │
│  │        BRepGraphInc_Storage (唯一真源)        │        │
│  │  ┌─────────────────────────────────────┐     │        │
│  │  │ 定义表 (Def)  │ 引用表 (Ref)  │ 表示表 (Rep) │    │
│  │  └─────────────────────────────────────┘     │        │
│  │  ┌─────────────────────────────────────┐     │        │
│  │  │      BRepGraphInc_ReverseIndex       │     │        │
│  │  │  Edge→Faces/Wires/CoEdges  等16张表  │    │        │
│  │  └─────────────────────────────────────┘     │        │
│  │  UID 向量 │ 原始形状                           │        │
│  └──────────────────────────────────────────────┘        │
│  │ 形状缓存 │ UID 反查 │ 历史日志 │ 延期列表 │       │
└──────────────────────────────────────────────────────────┘
```

**数据流向**:
- 算法 → 门面视图 → Data → Storage（定义/引用/表示）+ ReverseIndex（反向索引）
- 正向遍历: TopoView → Definition/Reference 表 → 子定义/引用向量
- 反向查询: TopoView → ReverseIndex → O(1) 邻接查找
- 变异: BuilderView → markModified → 向上传播 SubtreeGen → 层通知 → 缓存失效
- 重建: ShapesView → Storage → TopoDS_Shape → 缓存填充

---

## 2. 数据模型

### 2.1 三层分工：TopoDS、BRep_T*、BRepGraph

| 层次 | 代码位置 | 作用 |
|------|---------|------|
| **TopoDS** | `TopoDS_Shape.hxx` | 拓扑外壳：`TopoDS_Shape` 引用 `TopoDS_TShape`，携带 `TopLoc_Location` 与 `TopAbs_Orientation`。同一 `TShape` 可在父形状中多次出现（共享边/顶点）。 |
| **BRep_T\*** | `BRep_TVertex/TEdge/TFace.hxx` | 几何载体：在 `TopoDS_T*` 上附加三维点、曲线表示列表、曲面与三角化等。 |
| **BRepGraph** | `BRepGraph.hxx` 等 | 索引化图：按种类扁平存储实体，整数交叉引用 + 反向索引，与 TopoDS 可互转，不替代 TopoDS。 |

BRepGraph 引入的核心改进：
- **CoEdge** 实体将"边在某面上的 PCurve、UV 端点"直接放在 CoEdge 行中，查询时按下标直达，无需在 `TEdge` 的曲线表示链表中匹配 `(Surface, Location)`
- **反向索引** 提供 O(1) 的 Edge→Faces、Face→Shells 等邻接查询，无需遍历全图或构建 `TopTools_IndexedDataMap`

### 2.2 定义（Definition）

`*Def` 结构体按类型存放在扁平向量中，通过 `NodeId` 的 `Index` 字段直接寻址。所有定义继承自 `BaseDef`：

**BaseDef 公共字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `Id` | `BRepGraph_NodeId` | 类型化地址 (Kind + Index) |
| `OwnGen` | `uint32_t` | 自身数据修改计数 |
| `SubtreeGen` | `uint32_t` | 子树修改计数（向上传播） |
| `LastPropWave` | `uint32_t` | 传播波防重入标记 |
| `IsRemoved` | `bool` | 软删除标志 |

**各定义类型的专属字段**:

| 定义类型 | 关键字段 |
|---------|---------|
| `VertexDef` | `Point` (gp_Pnt), `Tolerance` |
| `EdgeDef` | `Curve3DRepId`, `ParamFirst/Last`, `Tolerance`, `IsDegenerate`, `SameParameter`, `SameRange`, `IsClosed`, `StartVertexRefId`, `EndVertexRefId`, `InternalVertexRefIds`, `Polygon3DRepId` |
| `CoEdgeDef` | `EdgeDefId`, `FaceDefId`, `Orientation`, `Curve2DRepId`, `ParamFirst/Last`, `Continuity`, `UV1/UV2`, `SeamPairId`, `SeamContinuity`, `Polygon2DRepId`, `PolygonOnTriRepIds` |
| `WireDef` | `IsClosed`, `CoEdgeRefIds` |
| `FaceDef` | `SurfaceRepId`, `TriangulationRepIds`, `ActiveTriangulationIndex`, `Tolerance`, `NaturalRestriction`, `WireRefIds`, `VertexRefIds` |
| `ShellDef` | `IsClosed`, `FaceRefIds`, `AuxChildRefIds` |
| `SolidDef` | `ShellRefIds`, `AuxChildRefIds` |
| `CompoundDef` | `ChildRefIds` |
| `CompSolidDef` | `SolidRefIds` |
| `ProductDef` | `ShapeRootId`, `RootOrientation`, `RootLocation`, `OccurrenceRefIds` |
| `OccurrenceDef` | `ProductDefId`, `ParentProductDefId`, `ParentOccurrenceDefId`, `Placement` |

**关键设计**:
- **CoEdgeDef** 拥有 `FaceDefId` 和 `EdgeDefId`，是边和面之间的连接点——PCurve、UV 参数、缝对信息都挂在 CoEdge 上
- **EdgeDef** 不直接存储"哪些面引用了我"——这由反向索引 `myEdgeToFaces` 提供
- **FaceDef** 的 `VertexRefIds` 直接存储 INTERNAL/EXCEPTION 顶点引用，无需遍历线

### 2.3 引用（Reference）

`*Ref` 结构体存储在父实体的向量中，承载每出现（per-occurrence）的数据——主要是方向和位置。所有引用继承自 `BaseRef`：

**BaseRef 公共字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `RefId` | `BRepGraph_RefId` | 类型化自地址 (RefKind + Index) |
| `ParentId` | `BRepGraph_NodeId` | 所属父节点 |
| `OwnGen` | `uint32_t` | 引用级修改计数 |
| `IsRemoved` | `bool` | 软删除标志 |

**各引用类型的专属字段**:

| 引用类型 | 关键字段 | 说明 |
|---------|---------|------|
| `ShellRef` | `ShellDefId`, `Orientation`, `LocalLocation` | 面引用壳时的方向和位置 |
| `FaceRef` | `FaceDefId`, `Orientation`, `LocalLocation` | 壳引用面时 |
| `WireRef` | `WireDefId`, `IsOuter`, `Orientation`, `LocalLocation` | 面引用线时，含外环标记 |
| `CoEdgeRef` | `CoEdgeDefId`, `LocalLocation` | **无 Orientation**——方向由 CoEdgeDef 持有 |
| `VertexRef` | `VertexDefId`, `Orientation` (默认 INTERNAL), `LocalLocation` | B-Rep 顶点分类 |
| `SolidRef` | `SolidDefId`, `Orientation`, `LocalLocation` | 复合体引用体时 |
| `ChildRef` | `ChildDefId` (NodeId), `Orientation`, `LocalLocation` | 复合体引用子实体时，异构目标 |
| `OccurrenceRef` | `OccurrenceDefId` | **无 Orientation/LocalLocation**——放置由 OccurrenceDef 持有 |

**关键设计**:
- `CoEdgeRef` 没有 `Orientation`，因为 `CoEdgeDef` 已有 `Orientation` 字段——方向是半边的本质属性而非引用的每出现属性
- `OccurrenceRef` 没有 `LocalLocation`，因为 `OccurrenceDef` 已有 `Placement`——实例的放置是其定义的一部分
- `WireRef` 额外有 `IsOuter` 标记，标识外环线

### 2.4 表示（Representation）

`*Rep` 结构体存储几何和网格数据，通过 `RepId`（`(RepKind, Index)`）寻址，不参与拓扑遍历或反向索引：

| RepKind | C++ 类型 | 说明 |
|---------|---------|------|
| Surface | `Geom_Surface` | 面的曲面 |
| Curve3D | `Geom_Curve` | 边的 3D 曲线 |
| Curve2D | `Geom2d_Curve` | 共边的 PCurve |
| Triangulation | `Poly_Triangulation` | 面的三角化 |
| Polygon3D | `Poly_Polygon3D` | 边的 3D 多边形 |
| Polygon2D | `Poly_Polygon2D` | 共边的 2D 多边形 |
| PolygonOnTri | `Poly_PolygonOnTriangulation` | 三角化上的多边形 |

**关键设计**：表示与拓扑节点解耦。同一 `Geom_Surface` 可被多个面共享（通过 Surface 三角化去重实现），但面的定义（FaceDef）只存储 RepId 索引而非 handle。

### 2.5 经典模型对照：边与 PCurve

**TopoDS/BRep 中**，边与面上 PCurve 的关系：

```cpp
// BRep_TEdge 内部是曲线表示的链表
class BRep_TEdge : public TopoDS_TEdge {
    NCollection_List<Handle(BRep_CurveRepresentation)> myCurves;
};

// 取某边在某面上的 PCurve：遍历链表 + 匹配 (Surface, Location)
Handle(Geom2d_Curve) BRep_Tool::CurveOnSurface(
    const TopoDS_Edge& E, const Handle(Geom_Surface)& S,
    const TopLoc_Location& L, ...) {
    // 循环遍历 TE->Curves()，找到 IsCurveOnSurface(S, L) 的那条
    // 未找到则尝试 CurveOnPlane 投影
}
```

**BRepGraph 中**，CoEdge 直接存储 PCurve 信息：

```
CoEdgeDef[12] = {
    EdgeDefId:    Edge[5]        ← 指向边定义
    FaceDefId:    Face[2]        ← 指向面定义
    Orientation:  FORWARD        ← 方向（在 CoEdge 上，不是 Ref 上）
    Curve2DRepId: RepId(Curve2D, 42)  ← 直接索引到 PCurve 表示
    UV1:          (0.0, 0.0)    ← UV 起始点
    UV2:          (1.0, 1.0)    ← UV 终止点
    SeamPairId:   CoEdgeId(27)  ← 缝对半边
}
```

查询 `BRepGraph_Tool::CoEdge::PCurve(graph, coEdgeId)` 直接按下标访问，无需遍历匹配。

---

## 3. 标识类型

### 3.1 NodeId

轻量类型化节点索引 `(NodeKind, Index)`。默认 Index = -1（无效）。

| Kind 值 | 类型 | 说明 |
|--------|------|------|
| 0 | Solid | 体 |
| 1 | Shell | 壳 |
| 2 | Face | 面 |
| 3 | Wire | 线 |
| 4 | Edge | 边 |
| 5 | Vertex | 顶点 |
| 6 | Compound | 复合体 |
| 7 | CompSolid | 复合体 |
| 8 | CoEdge | 半边 |
| 10 | Product | 产品（零件或装配） |
| 11 | Occurrence | 出现（放置实例） |

**角色**：存储向量下标，O(1) 直接寻址。Compact 后 Index 会重映射，不适合跨操作持久引用。

### 3.2 RefId

轻量类型化引用索引 `(RefKind, Index)`。8 种引用类型：

| RefKind | 父→子 | 承载数据 |
|---------|-------|---------|
| ShellRef | Solid → Shell | Orientation, Location |
| FaceRef | Shell → Face | Orientation, Location |
| WireRef | Face → Wire | Orientation, IsOuter, Location |
| CoEdgeRef | Wire → CoEdge | Location（方向在 CoEdgeDef 上） |
| VertexRef | Edge → Vertex | Orientation (默认 INTERNAL), Location |
| SolidRef | CompSolid → Solid | Orientation, Location |
| ChildRef | Compound → Child | Orientation, Location（异构目标） |
| OccurrenceRef | Product → Occurrence | 无（放置在 OccurrenceDef 上） |

**核心理念**：定义唯一（NodeId 标识），引用多实例（RefId 标识每出现数据）。

### 3.3 RepId

轻量类型化表示索引 `(RepKind, Index)`。7 种几何/网格类型（Surface、Curve3D、Curve2D、Triangulation、Polygon3D、Polygon2D、PolygonOnTri）。不参与拓扑遍历或反向索引。

### 3.4 UID 与 RefUID

持久唯一标识符，用于跨操作追踪和缓存失效。

**UID** `(NodeKind, Counter, Generation)` 和 **RefUID** `(RefKind, Counter, Generation)` 共享全局单调计数器（`myNextUIDCounter`），保证不冲突。

**核心原则**：Counter ≠ Index。Counter 单调递增不回收，Compact 后不变；Index 会重映射。

```
Compact 前:  NodeId{Edge,5} ←→ UID{Edge,Counter:5,Gen:1}
             NodeId{Edge,7} ←→ UID{Edge,Counter:7,Gen:1}  ← 已软删除

Compact 后:  NodeId{Edge,5} ←→ UID{Edge,Counter:5,Gen:2}  ← Index 不变
             NodeId{Edge,6} ←→ UID{Edge,Counter:8,Gen:2}  ← 原[8]重映射到[6]
                              ↑ Counter不变  ↑ Generation变了
```

**序列化契约**：写出 (Kind, Counter, OwnGen)；读入时重建 UID 向量，设置 `myNextUIDCounter = max(所有计数器) + 1`；Generation 重置为 0。

### 3.5 VersionStamp

版本快照 `(Domain, UID/RefUID, OwnGen, Generation)`，用于缓存失效检测：

- **OwnGen**：仅自身数据修改时递增（如 Edge.OwnGen++）
- **SubtreeGen**（节点级缓存）：自身或子树任何修改时递增，向上传播
- **Generation**：每次 Build()/Compact() 时递增

节点缓存用 SubtreeGen 做新鲜度检测（子树变更影响面/壳/体的边界框），引用缓存用 OwnGen（仅引用自身数据变更）。

### 3.6 四种标识总结

| 属性 | NodeId | UID | VersionStamp |
|------|--------|-----|-------------|
| 用途 | 运行时寻址 | 持久身份 | 缓存失效检测 |
| 结构 | `{Kind,Index}` | `{Kind,Counter,Generation}` | `{Domain,UID/RefUID,OwnGen,Generation}` |
| Compact后 | Index重映射 | Counter不变 | OwnGen刷新 |
| 查找速度 | O(1) 向量下标 | O(1) 哈希表 | O(1) 比较 |
| 序列化 | 不直接序列化 | 序列化核心 | 派生计算 |

| 属性 | RefId | RefUID | 与 Node 的关系 |
|------|-------|--------|---------------|
| 用途 | 引用寻址 | 引用持久身份 | 定义 1:N 引用 |
| 结构 | `{Kind,Index}` | `{Kind,Counter,Generation}` | 同一 EdgeDef 可有多个 CoEdgeRef |
| 承载 | 向量下标 | 不变计数 | Orientation/Location |

---

## 4. 存储后端与反向索引

### 4.1 BRepGraphInc_Storage

`BRepGraphInc_Storage` 是图的唯一真源存储，位于 `BRepGraphInc` 命名空间。包含：
- 按类型划分的扁平定义向量（`myVertices`, `myEdges`, `myCoEdges`, ...）
- 按类型划分的引用向量（`myShellRefs`, `myFaceRefs`, ...）
- 按类型划分的表示向量（`mySurfaces`, `myCurves3D`, ...）
- `BRepGraphInc_ReverseIndex myReverseIdx`：反向索引表
- UID 向量、原始形状、驱逐分配器等

通过 `BRepGraph_Data` 的 PIMPL 指针间接访问，算法代码通过视图 API 操作而非直接访问 Storage。

### 4.2 BRepGraphInc_ReverseIndex

维护 **16 个反向索引表**，均在 `Build()` 阶段从正向引用派生构建：

**核心拓扑反向索引（8 个）**：

| 字段 | 源 → 目标 | 含义 |
|------|-----------|------|
| `myEdgeToWires` | Edge → Wire | 边属于哪些线 |
| `myEdgeToFaces` | Edge → Face | 边属于哪些面（从 CoEdgeDef.FaceDefId 派生，去重） |
| `myEdgeToCoEdges` | Edge → CoEdge | 边被哪些半边引用 |
| `myVertexToEdges` | Vertex → Edge | 顶点关联哪些边 |
| `myWireToFaces` | Wire → Face | 线属于哪些面 |
| `myFaceToShells` | Face → Shell | 面属于哪些壳 |
| `myShellToSolids` | Shell → Solid | 壳属于哪些体 |
| `myCoEdgeToWires` | CoEdge → Wire | 半边属于哪些线 |

**装配与 Compound 反向索引（6 个）**：

| 字段 | 源 → 目标 |
|------|-----------|
| `myProductToOccurrences` | Product → Occurrence |
| `myCompoundsOfSolid` | Solid → Compound |
| `myCompSolidsOfSolid` | Solid → CompSolid |
| `myCompoundsOfShell` | Shell → Compound |
| `myCompoundsOfFace` | Face → Compound |
| `myCompoundsOfCompound` | Compound → Compound |

**额外**：`myCompoundsOfCompSolid` (CompSolid → Compound), `myEdgeFaceCount` (O(1) 边界面数缓存)

**为什么反向索引独立于定义？**

| 问题 | 说明 |
|------|------|
| 循环依赖 | EdgeDef 要知道所有 CoEdgeDef，CoEdgeDef 又引用 EdgeDef |
| 共享冲突 | 内置引用列表迫使为每个出现复制一份定义 |
| 修改爆炸 | 添加/删除一个 CoEdge 就要修改 EdgeDef 本体 |
| 单一职责 | 定义应只存本质数据，"谁引用了我"是索引层的关注点 |

**增量维护 API**：`BindEdgeToWire()`, `UnbindEdgeFromWire()`, `BindEdgeToFace()`, `BuildDelta()` 等，支持缝合等算法的增量更新。

**两级访问**：指针返回（`WiresOfEdge()` → `nullptr` 表示空）用于后端；安全引用（`WiresOfEdgeRef()` → 静态空向量）用于公共 API。

### 4.3 Edge→Faces 的派生原理

Edge→Faces 不是直接存储的，而是从 CoEdgeDef.FaceDefId 二级派生：

```
Edge[5]
  → myEdgeToCoEdges: [CoEdge[12], CoEdge[27]]
     → CoEdgeDef[12].FaceDefId = Face[2]
     → CoEdgeDef[27].FaceDefId = Face[7]
  → myEdgeToFaces: [Face[2], Face[7]]    ← 去重后的结果
```

因为 BRepGraph 的数据模型中，**Edge 并不直接知道 Face**——唯一的连接点是 CoEdge。

---

## 5. 公共 API：分组视图

### 5.1 TopoView

只读拓扑定义和邻接查询视图。通过 `BRepGraph::Topo()` 获取。

| 子视图 | 关键查询 |
|--------|---------|
| `FaceOps` | 定义字典、反向壳列表、反面向列表、曲面/三角化表示 ID、外环线 |
| `EdgeOps` | 定义字典、线/共边/面反向列表、3D 曲线表示 ID、边界性、流形性、PCurve |
| `VertexOps` | 定义字典、反向边列表 |
| `WireOps` | 定义字典、反向面列表 |
| `ShellOps` | 定义字典、反向体列表、反向复合体列表 |
| `SolidOps` | 定义字典、反向复合体/复合体列表 |
| `CoEdgeOps` | 定义字典、反向线列表、边定义 ID、面定义 ID、PCurve、缝对 ID |
| `CompoundOps` | 定义字典、父复合体列表 |
| `CompSolidOps` | 定义字典、反向复合体列表 |
| `ProductOps` | 定义、实例列表、形状根、是否装配/零件 |
| `OccurrenceOps` | 定义、产品 ID、父产品、父出现、出现位置 |
| `GenOps` | 通用实体字典、是否删除、总节点数/活动数 |
| `GeometryOps` | 曲面/3D曲线/2D曲线 表示数量及定义 |
| `PolyOps` | 三角化/3D多边形/2D多边形/三角化上多边形 表示数量及定义 |

### 5.2 RefsView

只读引用条目访问视图。通过 `BRepGraph::Refs()` 获取。每个引用类型一个子视图（ShellOps、FaceOps 等），提供 `Nb()`, `NbActive()`, `Entry()`, `IdsOf()` 和通用方法 `ChildNode()`, `IsRemoved()`, `LocalLocation()`, `Orientation()`。

### 5.3 UIDsView

UID/RefUID 双向查找和版本戳查询。`Of()` 双向解析、`StampOf()` 生成版本戳、`IsStale()` 失效检测。

### 5.4 ShapesView

TopoDS_Shape 重构视图。`Shape()` 使用 SubtreeGen 验证的缓存；`FindNode()` 通过 TShape 反查；Product/Occurrence 支持累积放置。

### 5.5 CacheView

非常量瞬态缓存视图。节点级和引用级 `Set/Get/Has/Remove/Invalidate`。支持 `BRepGraph_CacheKind` 描述符键控和 slot 预解析快速路径。

### 5.6 BuilderView

非常量变异视图。节点创建（`AddVertex/AddEdge/.../AddOccurrence`）、链接（`AddFaceToShell/AddShellToSolid`）、表示创建（`CreateCurve2DRep/CreateTriangulationRep/...`）、形状追加（`AppendFlattenedShape/AppendFullShape`）、软删除（`RemoveNode/RemoveSubgraph/RemoveRef/RemoveRep`）、拓扑编辑（`SplitEdge/ReplaceEdgeInWire/...`）、RAII 守卫（`Mut*()` 18种）、批量变异（`BeginDeferredInvalidation/EndDeferredInvalidation`）。

---

## 6. 构建与算法

| 类 | 文件 | 说明 |
|----|------|------|
| `BRepGraph_Builder` | `BRepGraph_Builder.hxx/.cxx` | 静态构建辅助。`Perform()` 从 TopoDS_Shape 填充图；`AppendFlattened/AppendFull` 追加。 |
| `BRepGraph_Compact` | `BRepGraph_Compact.hxx/.cxx` | 图压缩算法。rebuild-and-swap 策略，消除已删除节点，重映射索引为连续。 |
| `BRepGraph_Copy` | `BRepGraph_Copy.hxx/.cxx` | 图到图深拷贝。单遍自底向上，深拷（几何独立）或浅拷（共享几何）。 |
| `BRepGraph_Deduplicate` | `BRepGraph_Deduplicate.hxx/.cxx` | 深度几何去重。GeomHash 规范化相同曲面和曲线，可选合并拓扑定义。 |
| `BRepGraph_Transform` | `BRepGraph_Transform.hxx/.cxx` | 图变换。`Perform()` 变换整个图；`TransformFace()` 变换单个面。 |
| `BRepGraph_Validate` | `BRepGraph_Validate.hxx/.cxx` | 结构不变量检查器。`Lightweight` 模式（快速边界检查）或 `Audit` 模式（完整审计）。 |

---

## 7. 变异追踪

### 7.1 MutGuard\<T\>

RAII 可变守卫。编译期分派：`BaseDef` → `markModified()`，`BaseRef` → `markRefModified()`，`BaseRep` → `markRepModified()`。析构时触发通知（仅一次）。

```cpp
{
  auto guard = graph.Builder().MutEdge(BRepGraph_EdgeId(42));
  guard->Tolerance = 0.5;
  guard->SameParameter = true;
} // markModified 在此调用
```

### 7.2 DeferredScope

RAII 批量变异守卫。构造时激活延期模式，析构时刷出 SubtreeGen 传播并调用 `CommitMutation()`。可重入（内部守卫为空操作）。

### 7.3 History / HistoryRecord

追加式历史子系统。`Record()` 记录替换映射，`FindOriginal()` 向上追溯，`FindDerived()` 向前查找。`HistoryRecord` 包含操作名、序列号和 NodeId→NodeId 映射。

---

## 8. 遍历器

| 遍历器 | 方向 | 说明 |
|--------|------|------|
| `BRepGraph_ChildExplorer` | 向下 | 栈式层次遍历器，带内联 Location/Orientation 累积。Recursive 或 DirectChildren 模式。 |
| `BRepGraph_ParentExplorer` | 向上 | 出现感知父节点遍历器。同一定义经不同出现路径产生独立累积变换。 |
| `BRepGraph_RelatedIterator` | 单层 | 语义关联遍历器，带 RelationKind 标记（BoundaryEdge、AdjacentFace、OuterWire 等）。 |
| `BRepGraph_WireExplorer` | 连接顺序 | 按顶点邻接重排 CoEdge，等价于图层面的 BRepTools_WireExplorer。 |

---

## 9. 几何访问：BRepGraph_Tool

居中式几何访问 API，图层面的 `BRep_Tool` 等价物。引用 Location 自动应用于 3D 几何。

| 嵌套类 | 关键方法 |
|--------|---------|
| `Vertex` | `Pnt`, `Tolerance`, `Parameter`, `Parameters`, `PCurveParameter` |
| `Edge` | `Tolerance`, `Degenerated`, `SameParameter`, `Range`, `Curve/CurveAdaptor`, `Polygon3D`, `Continuity/MaxContinuity`, `IsClosedOnFace`, `StartVertex/EndVertex` |
| `CoEdge` | `HasPCurve`, `PCurve/PCurveAdaptor`, `UVPoints`, `Range`, `HasPolygonOnSurface/PolygonOnSurface` |
| `Face` | `Tolerance`, `NaturalRestriction`, `HasSurface/Surface/SurfaceAdaptor`, `HasTriangulation/Triangulation`, `OuterWire` |
| `Wire` | `IsClosed`, `NbCoEdges` |

---

## 10. 层与缓存

| 类 | 说明 |
|----|------|
| `BRepGraph_Layer` | 抽象层基类。提供生命周期回调（`OnNodeRemoved`, `OnNodeModified`, `OnRefModified` 等）和订阅掩码。持久存活。 |
| `BRepGraph_LayerRegistry` | GUID 键控的运行时层注册表。O(1) 注册/查找/注销/分派。 |
| `BRepGraph_ParamLayer` | 参数层。存储顶点在线/面/PCurve 上的参数绑定，含反向索引。 |
| `BRepGraph_RegularityLayer` | 连续性层。存储边在相邻面对之间的连续性记录。 |
| `BRepGraph_TransientCache` | 节点级瞬态缓存。`(NodeId, CacheKind slot) → CacheValue + SubtreeGen`。Reserve 后无锁并行。 |
| `BRepGraph_RefTransientCache` | 引用级瞬态缓存。`(RefId, CacheKind slot) → CacheValue + OwnGen`。 |
| `BRepGraph_CacheKind` | 缓存种类描述符（GUID + 节点类型掩码）。 |
| `BRepGraph_CacheKindRegistry` | 进程全局缓存种类注册表。 |
| `BRepGraph_CacheValue` / `TypedCacheValue<T>` | 抽象/模板化缓存值。`Get(computer)` 双检锁懒计算。 |

---

## 11. 迭代器与并行策略

| 组件 | 说明 |
|------|------|
| `BRepGraph_DefsIterator.hxx` | 定义向量快速迭代器模板（DefsWireOfFace、DefsEdgeOfWire 等） |
| `BRepGraph_RefsIterator.hxx` | 引用向量快速迭代器模板（RefsShellOfSolid、RefsFaceOfShell 等） |
| `BRepGraph_CacheKindIterator<TKeyId>` | 零分配缓存种类迭代器 |
| `BRepGraph_LayerIterator` | 层注册表迭代器 |
| `BRepGraph_ParallelPolicy` | 工作负载感知并行策略。`ShouldRun()` 在 `totalWorkUnits > workers²` 时才并行。 |

---

## 12. 拓扑与装配关系图

### 12.1 B-Rep 主链

```
Compound ──ChildRef──→ Compound/Solid/Shell/Face/...
CompSolid ──SolidRef──→ Solid
  Solid ──ShellRef──→ Shell
    Shell ──FaceRef──→ Face
      Face ──WireRef──→ Wire
        Wire ──CoEdgeRef──→ CoEdge
          CoEdge ──EdgeDefId──→ Edge
            Edge ──VertexRef──→ Vertex
```

### 12.2 装配链

```
Product(assembly) ──OccurrenceRef──→ Occurrence
  Occurrence ──ProductDefId──→ Product(shape)
  Occurrence ──Placement──→ gp_Trsf (局部放置)
```

### 12.3 反向索引（从子到父）

```
Vertex → Edges            (myVertexToEdges)
Edge   → Wires, CoEdges, Faces  (myEdgeToWires, myEdgeToCoEdges, myEdgeToFaces)
CoEdge → Wires            (myCoEdgeToWires)
Wire   → Faces            (myWireToFaces)
Face   → Shells           (myFaceToShells)
Shell  → Solids           (myShellToSolids)
Solid  → Compounds, CompSolids  (myCompoundsOfSolid, myCompSolidsOfSolid)
Product → Occurrences     (myProductToOccurrences)
```

### 12.4 表示链（定义到几何）

```
Face   ──SurfaceRepId──→ Geom_Surface
Edge   ──Curve3DRepId──→ Geom_Curve
CoEdge ──Curve2DRepId──→ Geom2d_Curve   (PCurve)
Face   ──TriangulationRepId──→ Poly_Triangulation
Edge   ──Polygon3DRepId──→ Poly_Polygon3D
CoEdge ──Polygon2DRepId──→ Poly_Polygon2D
CoEdge ──PolygonOnTriRepId──→ Poly_PolygonOnTriangulation
```