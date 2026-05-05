# Open CASCADE Technology 8.0.0 Beta 1 发布总结

> 本文基于 [GitHub Discussion #1249](https://github.com/Open-Cascade-SAS/OCCT/discussions/1249) 及最新提交记录整理。

---

## 概览

2026 年 4 月 30 日，Open Cascade 正式发布 **Open CASCADE Technology 8.0.0 Beta 1**。这是 OCCT 8.0.0 的**功能冻结里程碑**（Feature Freeze），累计自 7.9.0 以来超过 **500 项变更**，其中 Beta 1 相对 RC5 新增 **41 项**。正式版计划于 **2026 年 5 月 7 日** 发布，后续仅做文档完善、Sample 现代化和关键 Bug 修复。

---

## 核心亮点

### 1. NCollection 现代化（重大变更）[#1212](https://github.com/Open-Cascade-SAS/OCCT/pull/1212)

8.0.0 中影响面最广的变更：

- **`Size()` 返回类型从 `int` 迁移到 `size_t`**：所有 Map、Sequence、Array、List 的 `Size()` 现在返回 `size_t`，旧语义由 `Length()` 保留。涉及 TKernel/TKBRep/TKMath/TKBO/TKV3d/TKOpenGl 等数十个模块。
- **新增 `NCollection_LinearVector`**：连续内存的动态数组，基于 `Standard::Reallocate` 增长，对齐 `std::vector` 内存模型。
- **`NCollection_DynamicArray` 重构**：基于 `LinearVector<T*>` 的 2 的幂分块存储，O(1) 索引访问。
- **`NCollection_BasePointerVector` 移除**，`NCollection_Vector` 废弃。
- **`NCollection_IncAllocator` 无锁优化**：`std::shared_mutex` + CAS 原子分配，仅在新区块分配时加锁。

### 2. BRepGraph 编辑器整合（重大变更）[#1212](https://github.com/Open-Cascade-SAS/OCCT/pull/1212)

- `BuilderView` 移除，`EditorView` 成为唯一的修改入口。
- `EditorView` 统一管理结构变更（`Add*`/`Remove*`）和字段级 RAII 修改（`Mut*()`），自动传播 `OwnGen`/`SubtreeGen`。
- 新增 **`BRepGraph_MeshCache` / `BRepGraph_MeshView`** 双层网格存储，分离算法缓存与定义三角剖分。
- 新增 `RefsIterator`、`ReverseIterator` 等泛型迭代器，支持 `uint32_t` ID 体系。

### 3. Shader 无限网格 [#1223](https://github.com/Open-Cascade-SAS/OCCT/pull/1223)

`V3d_RectangularGrid` 和 `V3d_CircularGrid` 统一到单条 Shader 路径：
- **无限范围** + 亚像素抗锯齿（`fwidth`）
- Nyquist 渐隐：网格周期小于 1 像素时自动淡出
- 背景（Sky-Plane）模式，`gl_FragDepth = 1.0`
- 边界裁剪支持（矩形 `SizeX/SizeY`、圆形 `Radius`、弧形范围）
- 稳定参考坐标系，消除大幅偏移时的精度抖动
- 无需修改 `Graphic3d_Camera` API

### 4. BVH 加速多面体干涉检测 [#924](https://github.com/Open-Cascade-SAS/OCCT/pull/924)

新增 `IntPatch_PolyhedronBVH` + `IntPatch_BVHTraversal`，将多面体三角面片对扫描从 O(n²) 替换为 **O(log n) 双重 BVH 遍历**。

### 5. 选择系统 Flip 感知 [#1219](https://github.com/Open-Cascade-SAS/OCCT/pull/1219)

新增 `Graphic3d_Flipper`，使 BVH 包围盒和视锥体重叠检测对齐 `OpenGl_Flipper` 渲染行为，修复 `AIS_TextLabel` 和 `PrsDim_Dimension` 在相机旋转后的拾取失败问题。

### 6. `-Werror` CI 全覆盖 [#1209](https://github.com/Open-Cascade-SAS/OCCT/pull/1209)

Linux/macOS CI 启用 `-Werror -Wzero-as-null-pointer-constant`，锁定零警告基线。

### 7. 两轮大规模 Clang-Tidy 扫荡

- **Round 1** [#1207](https://github.com/Open-Cascade-SAS/OCCT/pull/1207)：`emplace_back`、`empty()`、`= default`
- **Round 2** [#1245](https://github.com/Open-Cascade-SAS/OCCT/pull/1245)：全代码库 `readability-braces-around-statements`、`nullptr`、`override`、`=default/delete`

---

## 建模算法修复

| 问题 | 修复 | PR |
|------|------|-----|
| Shape Healing 共享子形状导致栈溢出/边倍增 | 替换链条叶节点解析 + 循环检测 + DFS in-flight 保护 | [#1227](https://github.com/Open-Cascade-SAS/OCCT/pull/1227) [#1247](https://github.com/Open-Cascade-SAS/OCCT/pull/1247) |
| `ShapeUpgrade_FaceDivide::Perform()` 空上下文崩溃 | 延迟初始化 `ShapeBuild_ReShape` | [#1203](https://github.com/Open-Cascade-SAS/OCCT/pull/1203) |
| `BRepFill_PipeShell::MakeSolid()` 崩溃 | `IsSameOriented()` 加入 Contains 守卫 | [#1224](https://github.com/Open-Cascade-SAS/OCCT/pull/1224) |
| 两圆柱相切线情况 | 回退到隐式-隐式求交 | [#1228](https://github.com/Open-Cascade-SAS/OCCT/pull/1228) |
| 退化薄实体误分类 | 使用父 Solid 而非 Shell 做 SameDomain 面映射 | [#1231](https://github.com/Open-Cascade-SAS/OCCT/pull/1231) |
| `IntWalk_PWalking` 3D 成环检测 | 3D 距离闭合检查 | [#1226](https://github.com/Open-Cascade-SAS/OCCT/pull/1226) |

---

## 可视化修复

| 问题 | 修复 | PR |
|------|------|-----|
| AIS_ColorScale 三角形绕序 | 修复背面剔除下的接缝/顶点丢失 | [#1220](https://github.com/Open-Cascade-SAS/OCCT/pull/1220) [#1225](https://github.com/Open-Cascade-SAS/OCCT/pull/1225) |
| 折线选择 strict-inclusion 失效 | 重写 `OverlapsBox`/`OverlapsPoint` | [#1222](https://github.com/Open-Cascade-SAS/OCCT/pull/1222) |
| `Select3D_SensitivePrimitiveArray` 三角形顶点查询 | 新增 `GetVertex()` 纯访问器 | [#1221](https://github.com/Open-Cascade-SAS/OCCT/pull/1221) |

---

## 数据交换修复

| 问题 | 修复 | PR |
|------|------|-----|
| `XSControl_Reader::OneShape()` 崩溃 | 允许空 Shape 结果，构建复合时跳过 | [#1216](https://github.com/Open-Cascade-SAS/OCCT/pull/1216) |
| `CheckSRRReversesNAUO` 段错误 | 添加空指针检查 | [#1215](https://github.com/Open-Cascade-SAS/OCCT/pull/1215) |
| PMI 测量转换因子 | 支持 `ReprItemAndLengthMeasureWithUnitAndQRI` 等类型 | [#1218](https://github.com/Open-Cascade-SAS/OCCT/pull/1218) |

---

## 测试与 CI

- Windows ARM64 验证工作流发布到 master 分支 [#1213](https://github.com/Open-Cascade-SAS/OCCT/pull/1213)
- 两轮 QADraw → GTest 迁移 [#1235](https://github.com/Open-Cascade-SAS/OCCT/pull/1235) [#1243](https://github.com/Open-Cascade-SAS/OCCT/pull/1243)，移除大量遗留 Tcl 测试脚本
- 测试参考数据刷新适配新 Jenkins 构建站 [#1248](https://github.com/Open-Cascade-SAS/OCCT/pull/1248)

---

## 迁移要点速查

| 变更 | 旧方式 | 新方式 |
|------|--------|--------|
| `Size()` 返回 `size_t` | `aMap.Size()` 给 `int` API | 改用 `aMap.Length()` |
| `NCollection_Vector` 废弃 | `#include <NCollection_Vector.hxx>` | `NCollection_DynamicArray` 或 `NCollection_LinearVector` |
| `BRepGraph_BuilderView` 移除 | `graph.BuilderView()` | `graph.EditorView()` |
| `V3d` 网格默认为无界 | 自动限制到 `0.5 * DefaultViewSize()` | 显式调用 `SetSizeX()`/`SetSizeY()` |
| IVtk 像素容差 | `SetTolerance(float)` | `SetPixelTolerance(int)` |
| `NCollection_BasePointerVector` 移除 | 直接使用 | `NCollection_LinearVector<T*>` |

---

## 预计时间线

- **功能冻结**：2026 年 4 月 30 日（Beta 1）
- **最终发布**：2026 年 5 月 7 日（若 Beta 期间无关键回归）
- Beta 阶段仅接受：文档完善、Sample 现代化、关键 Bug 修复
