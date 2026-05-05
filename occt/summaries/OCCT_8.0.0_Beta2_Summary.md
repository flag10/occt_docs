# Open CASCADE Technology 8.0.0 Beta 2 发布总结

> 本文基于 [GitHub Release V8_0_0_beta2](https://github.com/Open-Cascade-SAS/OCCT/releases/tag/V8_0_0_beta2) 及 Beta 1→Beta 2 提交记录整理。

---

## 概览

2026 年 5 月 3 日，Open Cascade 发布 **Open CASCADE Technology 8.0.0 Beta 2**，作为 Beta 1 的跟进版本。本次没有新增功能，仅聚焦 Beta 阶段发现的两个关键问题修复和 8.0.0 文档体系的全面更新。相对 Beta 1 新增 **6 项变更**，涉及 124 个文件（+4,011 / −6,870 行，多为文档重构）。正式版仍计划于 **2026 年 5 月 7 日** 发布。

**Beta 2 不引入任何 API 变更或功能新增。** Beta 1 迁移指南全文适用。

---

## 核心变更

### 1. STEP/IGES 读写管线线程安全 [#1259](https://github.com/Open-Cascade-SAS/OCCT/pull/1259)

修复 Beta 1 中并发使用 `STEPControl_Writer::Transfer` 时的 `libmalloc` double-free 崩溃，以及并发 STEP/IGES 读取器的间歇性崩溃。核心改动：

- **`STEPControl_Controller.cxx`** 新增静态互斥锁保护共享初始化路径
- **`XSAlgo_ShapeProcessor.cxx`** 将全局 `ShapeProcess` 静态对象替换为线程局部 TLS 存储
- **`LibCtl_Library.gxx`** 注册表访问加锁

**并发使用约定**：
| 场景 | 安全级别 |
|------|----------|
| 每线程一个 STEP 写入器（默认参数） | 安全 |
| 每线程一个 STEP 读取器 | 安全 |
| 每线程一个 IGES 读取器 | **不安全** — 需串行化调用 |

### 2. CPU 网格路径恢复 [#1252](https://github.com/Open-Cascade-SAS/OCCT/pull/1252)

Beta 1 中移除的经典 `Graphic3d_Structure` 网格路径恢复为与 Shader 网格共存的独立后端：

- `V3d_RectangularGrid` / `V3d_CircularGrid` 重新支持传统 CPU 渲染（用于旧版 GL 配置和嵌入式目标）
- Snap 算法不变
- CPU 路径和 Shader 路径**每视图互斥**：激活一个自动擦除另一个
- 新增测试用例：`tests/v3d/grid/circ_cpu`、`rect_cpu`、`mode_switch`

**使用方式**：
```cpp
// CPU 路径（恢复的传统方式）
V3d_View::ActivateGrid(Aspect_GT_Rectangular, ...);
V3d_View::SetGrid (Handle(V3d_RectangularGrid), ...);

// Shader 路径（Beta 1 新增）
V3d_View::GridDisplay (Aspect_GridParams, gp_Ax3);
```

### 3. 文档现代化 [#1256](https://github.com/Open-Cascade-SAS/OCCT/pull/1256)

8.0.0 文档体系全面刷新，适配 GitHub 原生工作流、vcpkg 构建和 C++17 基线：

- **构建指南**重写：以 GitHub + CMake + vcpkg 为主线
- **升级指南**扩展：完整的 7.9→8.0 迁移指引
- **贡献指南**缩减：移除旧版 Mantis/Gitolite/Inspector/DFBrowser 页面
- **Git 指南**精简：面向 GitHub PR 工作流
- 移除约 50 张过期示意图，裁剪大量冗余内容

### 4. 顶层 `samples/` 目录 [#1257](https://github.com/Open-Cascade-SAS/OCCT/pull/1257)

新增 `samples/README.md`，指向外部 samples 仓库和可浏览站点，替代原有内联采样代码的组织方式。

### 5. CI 警告清理 [#1253](https://github.com/Open-Cascade-SAS/OCCT/pull/1253)

在 Beta 1 的 `-Werror` 基线基础上进一步清理：

- OpenGL 相关文件 `NULL` → `nullptr` 统一扫查
- macOS（`.mm` 文件）编译警告修复
- Ubuntu Clang 构建下多处符号警告消除
- 影响文件：`OpenGl_GlFunctions.cxx`、`OpenGl_GraphicDriver.cxx`、`OpenGl_Window.cxx`、`OpenGl_Window_1.mm`、`Cocoa_Window.hxx`

---

## 变更统计

| 提交 | PR | 日期 | 说明 |
|------|-----|------|------|
| `42d9c36` | [#1259](https://github.com/Open-Cascade-SAS/OCCT/pull/1259) | 05-03 | STEP 写入 / STEP&IGES 读取线程安全 |
| `4674df1` | [#1257](https://github.com/Open-Cascade-SAS/OCCT/pull/1257) | 05-03 | 新增 `samples/` 目录 |
| `bec9558` | [#1256](https://github.com/Open-Cascade-SAS/OCCT/pull/1256) | 05-03 | 文档现代化 |
| `076aad3` | [#1253](https://github.com/Open-Cascade-SAS/OCCT/pull/1253) | 05-01 | CI 编译警告修复 |
| `3ad68cb` | [#1252](https://github.com/Open-Cascade-SAS/OCCT/pull/1252) | 05-01 | 恢复 CPU 网格路径 |
| `a70427f` | — | 05-03 | 版本号提升至 8.0.0.beta2 |

---

## 从 Beta 1 迁移到 Beta 2

**Beta 2 无新增 API 变更。** Beta 1 迁移要点全部适用。两个新增注意事项：

### STEP/IGES 并发
- 每线程一个写入器/读取器实例（使用默认参数集）即可安全并发读写 STEP
- IGES 读取暂不支持并发 — 在应用层串行化

### 网格后端
- 已有调用 `V3d_View::SetGrid` / `V3d_Viewer::ActivateGrid` 解析到恢复的 CPU 路径，行为与 7.9 一致
- `V3d_View::GridDisplay(Aspect_GridParams, ...)` 走 Shader 路径
- 同一视图上激活一个后端会擦除另一个

---

## Beta 大事记（回顾）

| 里程碑 | 日期 | 变更数 | 要点 |
|--------|------|--------|------|
| **RC5** | 04-28 | — | 最后一轮候选 |
| **Beta 1** | 04-30 | 41 项 | 功能冻结，NCollection 重构、BRepGraph 整合、Shader 网格、Clang-Tidy 扫荡 |
| **Beta 2** | 05-03 | 6 项 | 线程安全修复、CPU 网格恢复、文档现代化 |
| **8.0.0 Final** | 05-07 (计划) | — | 正式发布 |

---

## 完整变更集

Beta 1→Beta 2 完整 Changelog：  
https://github.com/Open-Cascade-SAS/OCCT/compare/V8_0_0_beta1...V8_0_0_beta2
