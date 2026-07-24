# React 核心概念文档

这组文档从当前仓库源码出发，分别解释 React 核心概念的定位、职责、工作机制与边界。它们不是用户 API 入门教程，也不把某个框架对 React 的封装当成 React 本身。

## 阅读路径

第一次阅读建议先看[关系总览](00-relationships.md)，再按三层模型阅读：

1. 语法与构建期
   - [JSX](01-jsx.md)
   - [React Compiler](02-react-compiler.md)
2. 客户端协调运行时
   - [Fiber](03-fiber.md)
   - [Reconciler](04-reconciler.md)
   - [Scheduler](05-scheduler.md)
   - [Commit 阶段](06-commit-phase.md)
   - [Lane](07-lane.md)
3. 客户端与服务器执行面
   - [React Client](08-react-client.md)
   - [React Server](09-react-server.md)
4. 宿主与可观测性
   - [Renderer](10-renderer.md)
   - [React DevTools](11-react-devtools.md)
   - [Profiler](12-profiler.md)

如果需要完整的仓库拓扑、Hydration、Fizz、Flight、DevTools、构建发布与演进计划，请继续阅读根目录的[《React 项目关键技术架构与演进计划》](../../REACT_TECHNICAL_ARCHITECTURE_ZH.md)。

## 文档基线

| 项目 | 基线 |
| --- | --- |
| 源码 checkout | `a20b0c759b7927b38125811c10472720d39f9c21` |
| 分支 | `docs/architecture` |
| 文档日期 | 2026-07-24 |
| 核心版本 | `react` 19.3.0、`react-reconciler` 0.34.0、`scheduler` 0.28.0 |
| 证据范围 | 当前 checkout 的实现与测试；不是浏览器端到端、生产或 soak 证据 |

源码继续演进后，应优先核对每篇文档末尾的“源码导航”，而不是只修改版本号。

## 统一词汇

| 词汇 | 本文中的含义 |
| --- | --- |
| Element | JSX runtime 创建的 React 描述对象 |
| Fiber | Reconciler 使用的可恢复工作单元和树节点 |
| host | DOM、React Native 或自定义 renderer 所操作的目标环境 |
| renderer | 将 Reconciler 接到具体 host 的实现，例如 React DOM |
| render 阶段 | 计算 work-in-progress Fiber 树，可中断或重做 |
| commit 阶段 | 把已经完成的结果应用到 host，并运行相关 effect |
| Flight | React Server Components 使用的模型传输协议 |
| Fizz | React 的流式 HTML 服务端 renderer |

## 重要边界

- JSX 不创建 DOM；它通常先被转换为 JSX runtime 调用，再创建 Element。
- React Compiler 不替代 JSX transform、Reconciler 或 JavaScript 引擎。
- Fiber 是数据模型；Reconciler 是使用这套数据模型执行协调的算法与运行时。
- Lane 表达 React 工作的语义优先级；Scheduler 管理 JavaScript 回调何时获得执行时间。
- 并非所有工作都经过 Scheduler；同步 Lane 可以直接 flush。
- Commit 不是名为 “commiter” 的独立包，也不是可回滚数据库事务。
- `react-client` 与 `react-server` 包主要承载 Flight/Fizz 内核；“Client Component”和“Server Component”是执行与模块图边界，不能只按包名理解。
- Renderer 是 Reconciler 与具体 host 之间的能力适配层；React DOM 只是其中一种 renderer。
- DevTools 通过全局 hook、backend、bridge 和 frontend 观察 renderer；Profiler 的计时数据来自运行时 instrumentation（插桩）。
