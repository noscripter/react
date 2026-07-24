# React 项目关键技术架构与演进计划

> 本文是当前仓库的中文技术基线：解释 React 为什么采用这些架构、各子系统怎样协作、关键边界在哪里，以及哪些内容属于已实现能力、实验能力或演进计划。它不是对外 API 教程，也不是官方未来版本路线图。

九个核心概念以及 Renderer、React DevTools、Profiler 的独立深度说明与关系图，见 [`docs/react-architecture/`](docs/react-architecture/README.md)。

## 1. 文档基线与阅读约定

| 项目 | 基线 |
| --- | --- |
| 仓库 | React monorepo |
| 分支 | docs/architecture |
| 源码提交 | a20b0c759b7927b38125811c10472720d39f9c21 |
| 文档日期 | 2026-07-24 |
| 核心包版本 | react / react-dom 19.3.0；react-reconciler 0.34.0；scheduler 0.28.0 |
| 包管理器 | Yarn 1.22.22 |
| 证据范围 | 当前 checkout 的源码、测试、CI、设计文档和发布脚本；不等同于生产流量或长期 soak 证据 |

本文使用以下状态词，避免把“代码存在”误写成“能力已经上线”：

| 状态 | 含义 |
| --- | --- |
| implemented | 源码中已经实现 |
| component_verified | 相关组件测试或局部基准已通过 |
| system_verified | 真实组合路径的端到端行为已验证 |
| installed | 产物已经安装到目标环境 |
| activated | 目标环境已经启用该能力 |
| soak_verified | 长时间、稳态负载下已验证 |

对于仓库中的技术信号还要再区分：

- **稳定公开面**：稳定发布渠道明确导出的 API。
- **源码启用面**：当前源码或某个 build fork 中启用，不保证所有渠道都可用。
- **实验面**：由 experimental channel、feature flag、unstable API 或实验包承载。
- **计划面**：设计文档、迁移文档或 CI 明确持续推进。
- **线索面**：TODO、测试名和未启用 flag，只能证明问题被记录，不能当作承诺。

## 2. 一页结论

React 的核心不是“把 JSX 转成 DOM”，而是一个**声明式状态到宿主界面的一致性引擎**：

1. JSX 先产生不可直接执行 UI 的 React Element 描述。
2. Reconciler 把描述和已有 Fiber 树比较，生成一棵 work-in-progress Fiber 树。
3. Lane 模型决定不同更新之间的优先级、阻塞、纠缠和重试关系。
4. Scheduler 主要决定并发 JavaScript 工作什么时候执行、什么时候让出主线程；SyncLane 有直接 flush 路径。
5. Render 阶段可中断、可重做且不得产生不可回滚的宿主副作用。
6. Commit 阶段集中、不可随意中断地应用已完成结果；它不是可回滚事务。
7. Fizz 在服务器流式生成 HTML；Hydration 在客户端把 Fiber 接到已有 DOM。
8. Flight 传输 React 模型以及 Client/Server Reference，而不是 HTML。
9. React Compiler 在构建期推导响应式依赖和记忆化边界，减少手工 memo。
10. DevTools、Fast Refresh、测试矩阵、release channel 和 build fork 构成可观测性与兼容性外壳。

核心因果链如下：

~~~mermaid
flowchart LR
    A[组件与 JSX] --> B[React Element 描述]
    B --> C[Reconciler / Fiber]
    U[状态更新] --> L[Lane 优先级模型]
    L --> C
    C <--> S[Scheduler：并发工作时间切片]
    C --> W[work-in-progress 树]
    W --> K[Commit 阶段]
    K --> H[DOM / Native / 自定义宿主]

    SSR[服务器端执行 React 树] --> F[Fizz HTML 流]
    F --> HY[Hydration]
    HY --> C

    RSC[Server Components] --> FL[Flight 模型流]
    FL --> FM[Flight Client 重建模型]
    FM --> B
    FM -. 初始 HTML 组合路径 .-> F

    CMP[React Compiler] -. 构建期优化 .-> A
    DEV[DevTools / Profiler] -. 观察 .-> C
~~~

## 3. 第一性原理：为什么是这套架构

| 约束 | 架构选择 | 因果解释 | 反事实结果 |
| --- | --- | --- | --- |
| 用户代码只应描述“想要什么” | Element 与宿主节点分离 | 描述可被比较、延迟、重放和在不同 renderer 中解释 | JSX 直接改 DOM 会把声明、调度和平台副作用绑定，无法安全中断 |
| 大树更新不能长期独占主线程 | Fiber 把调用栈显式化 | 每个 Fiber 是可恢复工作单元，render 可暂停和重启 | 只用原生递归栈时，一次 render 必须运行到底 |
| 紧急输入不能被低优先级渲染拖住 | Lane 与 Scheduler 分层 | Lane 表达 React 语义优先级；Scheduler 表达主机执行时机 | 只有一个 FIFO 队列会让输入、网络重试和后台更新互相阻塞 |
| 已显示 UI 不应暴露 render 半成品 | render / commit 分离 | render 可丢弃；宿主变化集中到 commit 窗口 | render 途中写宿主环境，取消工作会留下半成品；commit 自身仍不是可回滚事务 |
| 首屏应尽快出现且可渐进交互 | Fizz + Suspense + Hydration | HTML 可流出，边界可独立完成，客户端可选择性接管 | 等整棵树完成再发 HTML 会放大慢数据源的尾延迟 |
| 服务器能力不能把全部代码发给浏览器 | Flight 的模块图分割 | Client/Server Reference 保留组件模型，同时明确执行位置 | 只发 HTML 会丢失后续模型；全部发 JS 又失去服务器计算优势 |
| 自动优化不能改变用户语义 | Compiler 建立在 React Rules 上 | 静态分析需要稳定的纯度、别名和 Hook 约束 | 对任意动态 JavaScript 强行优化会产生不可证明的行为变化 |
| 调试工具不能反过来压垮应用 | DevTools 增量协议与懒检查 | 只发送树操作和按需数据，降低观察者效应 | 每次 commit 传全树和深层 props 会造成明显性能失真 |

## 4. 仓库拓扑与所有权边界

这个仓库实际包含两个相互协作但工程栈不同的项目：

- **React runtime**：除 compiler 目录外的核心库、renderer、服务端协议、DevTools、构建与发布设施。
- **React Compiler**：compiler 目录中的 TypeScript 编译器、Babel/ESLint 集成、运行时，以及正在推进的 Rust 端口。

主要目录：

| 路径 | 职责 |
| --- | --- |
| [packages/react](packages/react) | 公共 React API、Element、Hooks 入口、Context、Transition 等 |
| [packages/react-reconciler](packages/react-reconciler) | Fiber 数据结构、协调、Lane、render/commit、错误与 Suspense |
| [packages/scheduler](packages/scheduler) | 与宿主事件循环协作的任务调度与让出 |
| [packages/react-dom](packages/react-dom) | DOM 客户端、服务端和静态渲染公共入口 |
| [packages/react-dom-bindings](packages/react-dom-bindings) | DOM host config、属性处理、事件系统、Fizz DOM 格式 |
| [packages/react-server](packages/react-server) | Fizz 与 Flight 的服务器核心 |
| [packages/react-client](packages/react-client) | Flight 客户端协议解码核心 |
| packages/react-server-dom-* | Webpack、Turbopack、Parcel、ESM、Unbundled、内部环境的 RSC 适配 |
| [packages/react-native-renderer](packages/react-native-renderer) | React Native renderer |
| [packages/react-noop-renderer](packages/react-noop-renderer) | 不依赖真实宿主的 reconciler 行为测试 renderer |
| [packages/react-test-renderer](packages/react-test-renderer) | 测试 renderer |
| [packages/react-devtools](packages/react-devtools) | DevTools 产品与架构文档 |
| [packages/react-devtools-shared](packages/react-devtools-shared) | backend、frontend、bridge、检查和 profiling 公共实现 |
| [packages/react-refresh](packages/react-refresh) | Fast Refresh runtime 与 bundler 集成 |
| [compiler](compiler) | React Compiler 的 TS 实现、插件、测试、文档和 Rust crates |
| [scripts/rollup](scripts/rollup) | 包图、bundle 类型、fork 和构建 |
| [scripts/jest](scripts/jest) | 多渠道、多模式测试矩阵 |
| [fixtures](fixtures) | DOM、Fizz、Flight、Concurrent、Compiler 等真实集成样例 |
| [.github/workflows](.github/workflows) | runtime、Compiler TS、Compiler Rust 等 CI 证据 |

模块边界上的关键原则是：**Reconciler 不知道 DOM，renderer 不重新实现协调算法，公共 React 包不决定具体宿主提交方式。**

## 5. 公共运行时：Element、组件、Hooks 与 Context

### 5.1 Element 是描述，不是实例

JSX runtime 最终构造普通对象形式的 React Element。核心字段包括：

- $$typeof：识别 React Element 的标记。
- type：字符串宿主类型或组件函数等。
- key：兄弟节点身份。
- props：输入描述；在 React 19 中，ref 的事实来源也是 props.ref。
- 开发构建还会附带 owner/debug 信息；element.ref 仅是过渡兼容 getter，访问时会提示 ref 已改为普通 prop，不能继续把它视为独立核心字段。

入口可见 [ReactJSXElement.js](packages/react/src/jsx/ReactJSXElement.js)。

Element 与 Fiber 不同：

- Element 是某次 render 的不可变输入描述。
- Fiber 是跨 render 存活的内部工作记录。
- DOM/Native 节点是 renderer 管理的宿主实例。

把三者分离后，同一组件模型才能同时支持 DOM、Native、SSR、测试 renderer 和未来宿主。

### 5.2 组件调用与 Hooks Dispatcher

公共 Hook 并不自己保存状态。[ReactHooks.js](packages/react/src/ReactHooks.js) 会读取当前 dispatcher，再转发 useState、useEffect、use、useMemo 等调用。Reconciler 在不同阶段安装不同 dispatcher：

- mount dispatcher：创建 Hook 链和初始状态。
- update dispatcher：按稳定调用顺序读取已有 Hook。
- rerender dispatcher：处理 render phase update。
- context-only dispatcher：禁止不合法的 Hook 调用。

主要实现位于 [ReactFiberHooks.js](packages/react-reconciler/src/ReactFiberHooks.js)。

这解释了“Hook 必须稳定地按顺序调用”：Hook 身份主要来自 Fiber 上链表的位置，而不是变量名。若条件分支改变调用序列，后续状态就无法与上次 render 一一对应。

### 5.3 Context 与 renderer 隔离

[ReactContext.js](packages/react/src/ReactContext.js) 为 primary/secondary renderer 保存独立 current value，并维护 renderer 相关字段。Context 是传播机制，不是通用状态数据库：读者 Fiber 会记录依赖，Provider 值变化再使依赖者进入更新路径。

## 6. Fiber 与 Reconciler

### 6.1 Fiber 数据模型

[ReactInternalTypes.js](packages/react-reconciler/src/ReactInternalTypes.js) 展示了 Fiber 的核心维度：

| 字段组 | 代表字段 | 作用 |
| --- | --- | --- |
| 身份 | tag、key、elementType、type | 判断节点类型与复用 |
| 宿主 | stateNode | 指向 DOM、Native 实例、root 或组件相关对象 |
| 树关系 | return、child、sibling、index | 用显式链表表示可暂停遍历的树 |
| 输入/结果 | pendingProps、memoizedProps、memoizedState | 比较新旧输入和保存已提交状态 |
| 更新 | updateQueue、dependencies | 保存更新、effects 和 Context 等依赖 |
| 副作用 | flags、subtreeFlags、deletions | 汇总 commit 要做的工作 |
| 优先级 | lanes、childLanes | 记录自身和子树待处理 Lane |
| 双缓冲 | alternate | 连接 current 与 work-in-progress Fiber |

Root 还维护 pending、suspended、pinged、warm、expired、entangled lanes，以及 cache、回调和 hydration 状态。

### 6.2 双缓冲

current 树代表屏幕上已经提交的状态；work-in-progress 树承载正在计算的新状态。alternate 把同一逻辑节点的两个版本连接起来。

双缓冲带来的核心保证：

- render 失败或被高优先级工作抢占时，可以丢弃未提交结果。
- current 树仍然对应用户正在看到的完整 UI。
- 未进入 commit 前，work-in-progress 不会替换 current。Mutation 完成后 root.current 会切到 finished tree，再进入 layout；后续 commit error 通过新更新恢复，不会事务式回滚已完成的宿主修改。

反事实是直接原地修改 current 树：一旦 render 中断，内部状态会同时包含新旧两种世界，恢复和错误回退都更困难。

### 6.3 协调与身份

协调算法在**同一父 Fiber 的 child 集合内**比较新 Element 与 current child Fiber：

- 有显式 key 时先按 key 找候选，无 key 时退化为 index，再检查 element type；兼容时才复用 Fiber 和状态。
- 身份不兼容时删除旧节点并创建新节点。
- 列表 key 决定“哪个状态属于哪个逻辑项目”，而不只是消除警告。
- 即使 key/type 相同，移动到另一个父节点也不保证保留状态。
- flags 和 subtreeFlags 将变化压缩为 commit 所需的 effect tree。

主路径分布在 [ReactFiberBeginWork.js](packages/react-reconciler/src/ReactFiberBeginWork.js)、[ReactChildFiber.js](packages/react-reconciler/src/ReactChildFiber.js) 与 [ReactFiberCompleteWork.js](packages/react-reconciler/src/ReactFiberCompleteWork.js)。

## 7. 更新、Lane 与 Scheduler

### 7.1 从 setState 到 Root

典型路径：

~~~mermaid
sequenceDiagram
    participant U as 用户事件/异步源
    participant F as Fiber update queue
    participant L as Lane/Root
    participant RS as Root Scheduler
    participant S as Scheduler
    participant W as Work Loop
    participant C as Commit

    U->>F: enqueue update
    F->>L: 标记 Fiber 与 ancestor lanes
    L->>RS: ensure root scheduled
    alt Sync work
        RS->>W: root microtask 直接 flush，不经过 Scheduler
    else Concurrent work
        RS->>S: 安排对应优先级 callback
        S->>W: 在可执行时间进入 render
        W-->>S: 必要时 yield/continuation
    end
    W->>C: completed tree
    C->>U: 宿主 UI、ref、effects 可见
~~~

并发下，update queue 不能简单“取出后删除”。本轮优先级不足的更新会保留到 base queue；为保持状态计算顺序，必要时还会克隆后续更新，待将来以新的 render lanes 重放。这样高优先级输入可以先提交，同时不会吞掉早先的低优先级状态变换。

### 7.2 Lane 是 React 的语义优先级

[ReactFiberLane.js](packages/react-reconciler/src/ReactFiberLane.js) 用位集合表达并发更新。当前模型包含 Sync、InputContinuous、Default、Gesture、Transition、Retry、SelectiveHydration、Idle、Offscreen、Deferred 等类别。

Lane 不只是“数字越小越快”，还编码：

- 哪些更新可以一起 render。
- 哪些 lane 被 suspend。
- promise resolve 后哪些 lane 被 ping。
- 哪些 transition 被 entangle，避免逻辑上相关的状态撕裂。
- 哪些工作过期，需要提升处理紧迫度。

### 7.3 Scheduler 是宿主时间管理

[Scheduler.js](packages/scheduler/src/forks/Scheduler.js) 使用 task queue 与 timer queue，按优先级和 expiration time 执行 callback；任务可返回 continuation，并在宿主要求时 yield。

这里有一个重要例外：SyncLane 在 root microtask 末尾直接 flush，并不经过 Scheduler callback；Scheduler 主要承载并发工作。把所有更新都画成 Scheduler task 会掩盖同步更新的真实延迟路径。

Lane 与 Scheduler 必须分开理解：

| 层 | 回答的问题 |
| --- | --- |
| Lane | “这批 React 更新与其他更新是什么关系，应该先算哪一个？” |
| Scheduler | “这段 JavaScript 现在能运行多久，何时暂停并继续？” |

如果把两层合并，React 的 Suspense/transition 语义会被迫绑定某个宿主事件循环；如果只保留 Lane 而没有让出机制，低优先级 render 仍可能长时间阻塞浏览器。

### 7.4 render 和 commit

[ReactFiberWorkLoop.js](packages/react-reconciler/src/ReactFiberWorkLoop.js) 驱动同步或并发 render：

1. beginWork 执行组件并进入子树。
2. completeWork 从叶子向上完成宿主准备和 flags 汇总。
3. render 可能 yield、重启、因错误或 thenable 转向另一条路径。
4. 完整 work-in-progress 树进入 commit。

Commit 的语义阶段是：

1. **before mutation**：读取即将变化前的宿主快照等。
2. **mutation**：插入、删除、更新宿主节点。
3. **layout**：ref、layout effect 和同步读取布局的生命周期。
4. **passive**：在后续时机清理并运行 passive effect。

关键不变量：

- render 阶段应保持纯净；它可能被执行多次而不 commit。
- commit 对一棵完成树执行，不能像 render 一样任意重放。
- layout effect 可阻塞绘制，应保持短小。
- passive effect 不属于 mutation/layout 的集中提交窗口。

### 7.5 错误、Thenable 与恢复

Render 抛出的值先按性质分流：

- 可等待 thenable 进入 Suspense 捕获、ping 与 retry 路径。
- 普通 render error 向上寻找 class error boundary；没有边界时由 root 处理。
- 并发 render 失败可转入同步恢复尝试。
- hydration error 可在相应 boundary 或 root 放弃 hydration，切换到客户端 render。

Root 支持 caught、uncaught 与 recoverable error 回调。Commit 阶段已经开始修改宿主树，不能像 render 一样整体回滚；commit error 会向最近 error boundary/root 安排后续捕获更新。因而“render 可丢弃”不能外推为“任何 React 工作都可无成本回滚”。

核心路径见 [ReactFiberThrow.js](packages/react-reconciler/src/ReactFiberThrow.js)、[ReactFiberWorkLoop.js](packages/react-reconciler/src/ReactFiberWorkLoop.js) 与 [ReactFiberRoot.js](packages/react-reconciler/src/ReactFiberRoot.js)。

## 8. Renderer 抽象

[react-reconciler README](packages/react-reconciler/README.md) 和 host config 定义 reconciler 与平台的契约。renderer 声明自己支持哪种提交模型：

- **Mutation**：原地创建、插入、更新和删除宿主实例；DOM 采用此模式。
- **Persistence**：克隆子树并一次替换 child set；部分 Native 路径使用此模式。
- **Hydration**：识别并接管既有宿主树。

host config 还负责：

- 实例与文本创建。
- props diff 与提交。
- 子节点操作。
- 宿主上下文与资源。
- 事件优先级、超时和微任务能力。
- Suspense/hydration marker 的平台解释。

自定义 renderer API 在仓库中仍被标记为实验接口。其兼容风险高于 react-dom 公共 API，升级时应以具体 reconciler 版本和 host config 类型为准。

## 9. React DOM 与事件系统

### 9.1 Root

[ReactDOMRoot.js](packages/react-dom/src/client/ReactDOMRoot.js) 的 createRoot 创建普通 container，hydrateRoot 创建 hydration container；二者最终都把工作交给 reconciler，但初始宿主状态不同。

Root 是调度、错误回调、hydration 和事件边界，不等同于一个 DOM 节点的简单包装。

### 9.2 DOM host config

[ReactFiberConfigDOM.js](packages/react-dom-bindings/src/client/ReactFiberConfigDOM.js) 实现 DOM 实例创建、属性更新、文本、资源、hydration 匹配和 Suspense marker 解释。[ReactDOMComponentTree.js](packages/react-dom-bindings/src/client/ReactDOMComponentTree.js) 维护 DOM 节点到 Fiber、当前 props 等内部映射。

### 9.3 委托事件与优先级

[DOMPluginEventSystem.js](packages/react-dom-bindings/src/events/DOMPluginEventSystem.js) 在 root/container 级注册支持的 native event，并通过插件系统提取合成事件。主要插件覆盖 Simple、EnterLeave、Change、Select、BeforeInput、FormAction、ScrollEnd 等语义。

[ReactDOMEventListener.js](packages/react-dom-bindings/src/events/ReactDOMEventListener.js) 把离散、连续和默认 DOM 事件映射到不同 React update priority。

因此，hydration 不是“为每个服务器 DOM 节点重新绑一个 click handler”。React DOM 主要使用 root 委托；hydration 真正完成的是 Fiber、props、refs、effects 与既有 DOM 的对应关系。若目标 boundary 尚未 hydrate，事件系统还可能阻塞并触发选择性 hydration。

## 10. 并发 UI、Suspense 与 Transition

### 10.1 Suspense 的最小语义

组件读取尚未就绪的异步资源时，render 可遇到 thenable。Reconciler 将相关 boundary 标记为 suspended，选择保留内容、显示 fallback 或延迟切换；thenable settle 后 ping 对应 root/lane 并重试。

Suspense 的价值不是“显示 loading”本身，而是建立一个**可调度的一致性边界**：

- boundary 内部可以失败、延迟或重试。
- 父级和兄弟不必被一个慢依赖整体阻塞。
- SSR、hydration 和客户端 render 可以共享边界语义。

### 10.2 Transition

Transition 将一组更新标注为可延迟工作，使紧急输入和已显示内容保持响应。它不让 JavaScript 自动并行，也不保证网络请求更快；它改变的是更新的调度与可见性策略。

### 10.3 Cache 与 use

react-server 条件入口和客户端/服务端 dispatcher 提供 cache、cacheSignal、use 等能力。它们依赖具体 render/request 上下文，不能把“render 期间缓存”误当成跨用户、跨请求的永久业务缓存。

### 10.4 Activity、ViewTransition 与 Gesture

当前 [ReactClient.js](packages/react/src/ReactClient.js) 和 [ReactFeatureFlags.js](packages/shared/ReactFeatureFlags.js) 可看到 Activity、ViewTransition、transition type、gesture transition 及相关 flags。这里必须按渠道解释：

- Activity 隐藏子树时会保留 Fiber 与组件状态，但隐藏宿主内容，并断开 layout/passive effects；恢复 visible 后再重新连接。因此 hidden 不等于 unmount，也不等于释放全部内存或外部资源。
- ViewTransition 将 React commit 与宿主 document.startViewTransition 协调；其端到端效果同时依赖浏览器 API、CSS、可访问性与失败回退。
- 源码中存在或默认 source flag 为 true，只能证明当前源码路径。
- WWW、Native、stable、experimental build 可能通过 fork 使用不同 flag。
- unstable 或 experimental 名称不是稳定兼容承诺。
- 浏览器 View Transition 能力还受宿主支持影响。

当前 HEAD 的 [stable source entry](packages/react/index.stable.js) 已包含 Activity、ViewTransition 和 addTransitionType，但仓库内入口状态不能单独证明对应版本已经公开发布到 npm；发布结论仍应核对目标 tarball 与 release metadata。

## 11. Fizz：流式服务端 HTML

### 11.1 核心模型

Fizz 在 [ReactFizzServer.js](packages/react-server/src/ReactFizzServer.js) 中以 Request、Task、Segment 和 SuspenseBoundary 组织工作：

| 对象 | 职责 |
| --- | --- |
| Request | 一次服务器 render 的全局状态、destination、队列、回调和 abort 集合 |
| Task | 一段待执行组件工作及其上下文 |
| Segment | 可独立完成和 flush 的输出片段 |
| SuspenseBoundary | 内容、fallback、完成状态和客户端指令边界 |

Segment 可处于 pending、completed、flushed、aborted、errored、postponed 等状态。Request 维护 completed root、completed boundary、partial boundary 等队列，以便在数据和 destination 可用时渐进输出。

### 11.2 流程

~~~mermaid
flowchart TD
    R[createRequest / createPrerenderRequest] --> T[创建 root task]
    T --> W[startWork]
    W --> X[执行组件]
    X -->|完成| S[completed segment]
    X -->|遇到 Suspense| B[boundary: content + fallback tasks]
    X -->|错误| E[boundary fallback 或 fatal error]
    S --> Q[completed queues]
    B --> Q
    Q --> F[startFlowing / flush]
    F -->|destination 接受| O[HTML + boundary instructions]
    F -->|背压| P[stopFlowing，保留 render 状态]
    P --> F
    R -->|abort| A[终止未完成任务并按语义降级]
~~~

Node 入口在 [ReactDOMFizzServerNode.js](packages/react-dom/src/server/ReactDOMFizzServerNode.js)，renderToPipeableStream 通过 stream drain 处理背压。

重要边界：

- stopFlowing 是输出背压，不等于 abort；render 工作可继续积累。
- Node writable 会根据 write 返回值暂停，并在 drain 后恢复；当前 WHATWG/Browser stream config 不提供完全相同的逐 write desiredSize 反馈，不能把 Node 背压结论直接外推到所有 Web Stream 环境。
- abort 是停止未完成服务器工作，并根据边界状态发出客户端接管指令。
- onShellReady 与 onAllReady 对应不同完成层级。
- renderToString 等 legacy API 不能等待 Suspense，不能替代流式路径。
- 源码中的 progressive chunk heuristic 是实现启发式，不是端到端 SLO。

renderToString 与 renderToStaticMarkup 仍复用 Fizz 核心，只是把 destination 变为字符串累加器、同步执行并立即终止未完成的 Suspense 工作。限制来自“同步返回完整字符串”的 API 契约，而不是另一套更强的 SSR 引擎。

### 11.3 Prerender 与 Resume

Fizz 包含 prerender、postponed state、resume 与静态 API 的组合，用于把已知静态工作和后续动态工作拆分。它们的正确性不仅取决于 React 核心，还取决于框架如何保存 postponed state、绑定请求数据、部署匹配产物并清理过期状态。

PostponedState 还必须延续 nextSegmentId、resumableState、format context、replay nodes/slots 等协议状态。prelude 与 resume 快照不可被应用任意修改，也不可跨不兼容的 React/host config 产物混用；重复分配 B:/S: ID 会让后续完成指令命中错误的 Suspense boundary。

## 12. Hydration 与 Dehydration

### 12.1 精确定义

Hydration 是首次客户端 render 时**接到已有宿主树，而不是重新创建它**。对 DOM 来说，它复用服务器 HTML，并建立 Fiber/DOM、props、ref、effect 和事件可达关系。

Dehydrated 在 reconciler/DOM 语境中表示：**服务器 DOM 已存在，但某个 root 或 Suspense boundary 尚未被客户端 Fiber 完整接管。**

这不是“DOM 里没有水分”或“把状态删除”，也不代表服务器内容一定未完成。

### 12.2 Root 与 Boundary 是两种状态

- Root 通过 RootState.isDehydrated 表示最外层 hydration shell 尚未接管。
- Suspense 通过 memoizedState.dehydrated 保存 DOM SuspenseInstance，表示 boundary 等待 hydration。
- 普通客户端 suspension 的 memoizedState 可以非空，但 dehydrated 字段为空；两者不可混淆。

对应实现主要在 [ReactFiberRoot.js](packages/react-reconciler/src/ReactFiberRoot.js)、[ReactFiberSuspenseComponent.js](packages/react-reconciler/src/ReactFiberSuspenseComponent.js) 与 [ReactFiberHydrationContext.js](packages/react-reconciler/src/ReactFiberHydrationContext.js)。

### 12.3 DehydratedFragment

DehydratedFragment 是内部 Fiber tag，不是 JSX Fragment，也不是 DOM DocumentFragment。它作为 Suspense Fiber 的临时 child：

~~~text
SuspenseComponent Fiber
└── DehydratedFragment Fiber
    ├── stateNode = server boundary start marker
    └── child = null
~~~

它用一个 opaque sentinel 代表整段服务器 DOM range，避免在尚未进入 boundary 时为每个 DOM 节点创建 child Fiber。若 React 放弃 hydration，commit 可以从 marker 范围删除整段 DOM。

### 12.4 DOM marker 与渐进接管

Fizz 使用 comment marker 编码 boundary：

| Marker | 服务器语义 |
| --- | --- |
| &lt;!--$--&gt; | completed boundary |
| &lt;!--$?--&gt; | pending boundary |
| &lt;!--$!--&gt; | 客户端渲染 / 永久 fallback 路径 |
| &lt;!--/$--&gt; | boundary end |

客户端 host config 识别 marker；流式指令可把 pending boundary 更新为 completed 并唤醒 React retry。

晚到的 segment 通常先进入隐藏容器，再由 Fizz instruction 移到 placeholder/boundary。若 CSP 禁止内联脚本，框架必须正确配置 nonce 或匹配的 external runtime；否则字节虽然到达浏览器，DOM 仍可能停在 fallback。external runtime 本身仍受实验 flag 约束。

### 12.5 选择性 Hydration 与事件重放

[ReactDOMEventReplaying.js](packages/react-dom-bindings/src/events/ReactDOMEventReplaying.js) 同时承载显式 hydration target 和一组连续事件的重放状态，但离散与连续事件不能混成一种机制：

- click 等离散 capture 事件会同步尝试 hydrate 被阻塞的 boundary；成功时原始事件继续分发，仍被阻塞时停止传播，并不存在通用的“稍后重放 click”队列。
- focusin、dragenter、mouseover、pointerover、gotpointercapture 等受支持的连续事件可以排队，解除阻塞后克隆并重放。
- 显式 hydration target 会按事件优先级排序，推动 selective hydration。

所以“HTML 已可见、ref 仍为空”与“交互已正常分发”是三个不同状态；验证 selective hydration 必须分别观察 boundary 接管、原始离散事件和连续事件队列。

### 12.6 失败与恢复

Hydration 要求客户端首棵树与服务器结构满足匹配契约。结构不匹配、过早 root update 或 boundary 错误可能导致：

- 局部 boundary 放弃 hydration，改用客户端 render。
- root 级失败时清理/替换更大范围。
- 通过 recoverable error 路径报告。

“浏览器最后看起来一样”不代表 hydration 正确：重复创建、丢失输入状态、事件时序和性能成本都可能不同。

### 12.7 DevTools 中同名但无关的 dehydration

[DevTools hydration.js](packages/react-devtools-shared/src/hydration.js) 的 dehydrate/hydrate 是 bridge 序列化优化：深层 props/state 先只传 preview、type、size 等 metadata，用户展开时再按路径请求真实值。

它与 DOM/Fiber hydration 没有运行时关系，只复用了“先移除部分信息、稍后补回”的比喻。

## 13. Flight 与 React Server Components

### 13.1 Flight 传输什么

Fizz 输出 HTML；Flight 输出可被 React 客户端重建的模型流。它支持 React 元素、引用、Promise、可迭代对象等协议值，并以 chunk/row 的方式渐进解析。

两者是独立协议。Fizz 的输入是 ReactNode，不是 Flight 字节；典型的初始 RSC SSR 组合会先由服务端 Flight Client 把 Flight stream 重建为 React 模型，再把该模型交给 Fizz 输出 HTML。框架也可以在浏览器侧把后续 Flight 模型交给 client renderer。

服务器核心为 [ReactFlightServer.js](packages/react-server/src/ReactFlightServer.js)，客户端核心为 [ReactFlightClient.js](packages/react-client/src/ReactFlightClient.js)。

~~~mermaid
flowchart LR
    SC[Server Component graph] --> FS[Flight Server]
    MAN[Bundler manifest] --> FS
    FS --> P[Flight rows/chunks]
    P --> FC[Flight Client]
    FC --> M[React model / lazy chunks]
    M --> RC[Client renderer]
    M -. 初始 SSR 组合时可在服务端交给 .-> FZ[Fizz HTML renderer]

    CR[Client Reference] --> MAN
    SR[Server Reference] --> MAN
    UI[用户调用 Server Action] --> FW[框架 endpoint / callServer]
    FW --> RP[Reply 解码与 manifest 解析]
    RP --> AZ[框架认证、授权与业务校验]
    AZ --> INV[框架调用已绑定 callable]
    INV -. 可选：结果作为新模型 .-> FS
~~~

### 13.2 Client Reference

use client 边界经 bundler 转换为 Client Reference。Flight 不执行浏览器模块，而是发送模块身份和导出信息；客户端 bundler runtime 再装载对应代码。

### 13.3 Server Reference 与 Action

use server 函数被登记为 Server Reference。客户端传回引用 ID 和序列化参数；React Reply/Action decoder 通过 manifest 解析并返回绑定后的 callable，但不会替框架执行它。框架在认证、授权和业务校验后负责实际调用，并决定是否把结果作为新的 Flight 模型返回。

这只是传输与定位机制，不自动提供：

- 用户认证。
- 对业务对象的授权。
- CSRF 防护。
- 幂等性。
- 限流、审计或事务。

框架和应用必须在每次 action 调用处重新验证身份与权限，不能因为函数“只存在服务器”就信任来自客户端的参数。

React Reply 解码路径虽然包含 proto、嵌套结构、BigInt 和 bound argument 等局部保护，但总请求体、上传文件、速率、deadline 和业务资源消耗仍属于 HTTP/框架层责任。

### 13.4 Bundler 与框架边界

仓库包含 Webpack、Turbopack、Parcel、ESM、Unbundled 和内部适配，说明 Flight 核心刻意不绑定单一 bundler。框架负责：

- 生成并版本化 client/server manifest。
- 路由、HTTP transport 与缓存策略。
- HTML 和 Flight payload 的协调。
- 部署时确保 server/client 产物匹配。
- 请求隔离、错误呈现和安全策略。

RSC/Flight README 明确把相关接口描述为实验性。包可以被发布或列入版本表，并不自动等于所有协议/API 已获得稳定兼容保证。

### 13.5 Taint 与序列化安全

实验 taint API 可阻止特定对象或值被序列化到 Client Component 或 action closure。它是纵深防御，不是访问控制；真正的 secret 仍不应进入可能返回客户端的模型路径。

生产 Flight 错误通常向客户端暴露通用错误与 digest，而不是服务器原始消息；框架需要用 digest 关联服务端日志，同时保证日志和回调本身不泄漏敏感信息。

## 14. React Compiler

### 14.1 目标与非目标

[Compiler README](compiler/README.md) 和 [DESIGN_GOALS.md](compiler/docs/DESIGN_GOALS.md) 的核心目标是让 React 应用默认更快：自动推导可以安全复用的值和 UI 子树，减少手工 useMemo/useCallback/memo，同时维持熟悉的声明式编程模型。

主要目标：

- 限制因父组件更新导致的无效重算和重渲染。
- 尽量不增加启动成本和产物体积。
- 不要求用户为每个优化点写注解。
- 让编译结果仍可调试。

主要非目标：

- 不保证零重算。
- 不为违反 React Rules 的程序兜底。
- 不覆盖所有动态 JavaScript，例如不可分析的 eval。
- 不以 legacy class component 作为主要优化对象。

### 14.2 TypeScript 编译管线

[Pipeline.ts](compiler/packages/babel-plugin-react-compiler/src/Entrypoint/Pipeline.ts) 展示了从 Babel AST 到优化代码的大致过程：

~~~mermaid
flowchart TD
    AST[Babel AST] --> HIR[Lower 到 HIR / CFG]
    HIR --> V[语义与 React Rules 验证]
    V --> SSA[SSA、Phi 与常量传播]
    SSA --> TY[类型与函数分析]
    TY --> EF[Mutation / Alias / Effect 分析]
    EF --> DCE[Dead Code Elimination]
    DCE --> RP[Reactive places]
    RP --> RS[Reactive scopes 与 dependencies]
    RS --> OPT[Scope prune / merge / flatten / optimize]
    OPT --> RF[ReactiveFunction]
    RF --> CG[Codegen 回 Babel AST]
    CG --> OUT[使用 compiler runtime memo cache 的代码]
~~~

HIR 保留 JavaScript 主要语义，但把控制流显式化为 basic block；SSA、别名/副作用分析和 reactive scope 推导让编译器能回答“哪些值变化会使这段计算失效”。

### 14.3 指令、gating 与失败边界

[Program.ts](compiler/packages/babel-plugin-react-compiler/src/Entrypoint/Program.ts) 处理 use memo/use forget、use no memo 和 gating 等模式。Babel plugin、ESLint plugin 和 compiler runtime 是一个整体：

- Babel plugin 执行转换。
- ESLint/验证路径尽早暴露违反规则或妨碍编译的问题。
- react/compiler-runtime 提供编译产物所需 memo cache。
- gating 允许框架或应用控制优化激活范围。

编译器正确性的首要门槛是语义等价，而不是命中率。一次错误复用比少一次优化严重得多。

## 15. Rust Compiler Port：当前最明确的关键演进计划

根目录工程说明把 Rust Compiler Port 标为 active work；[compiler Rust CI](.github/workflows/compiler_rust.yml) 持续执行 cargo check/build、AST、rust-port 与 snapshot 相关任务。计划文档位于 [compiler/docs/rust-port](compiler/docs/rust-port)，代码位于 [compiler/crates](compiler/crates)。

### 15.1 目标架构

[rust-port-architecture.md](compiler/docs/rust-port/rust-port-architecture.md) 给出的方向包括：

- arena 与强类型 ID：IdentifierId、ScopeId、FunctionId、TypeId，减少复制和生命周期耦合。
- flat instruction table 与 InstructionId，区分存储身份和 EvaluationOrder。
- Environment 与 HIR 分离传递。
- 有序 map/set，保证确定性输出和 snapshot 稳定。
- side map 保存分析结果，避免在多个 pass 中不断膨胀核心节点。
- 为适应 borrow checker，使用 collect-then-apply 的两阶段变换。
- 让 Rust pass 与 TS pass 在结构上保持可对照，而不是重新发明另一套算法。
- 用 Result 与诊断累积表达预期错误，避免把用户代码问题当进程 panic。

### 15.2 多前端边界

计划采用“复杂逻辑在 Rust 实现一次，工具侧保持薄适配”：

~~~mermaid
flowchart LR
    B[Babel AST] --> BS[薄 Babel shim]
    S[SWC AST] --> SS[薄 SWC shim]
    O[OXC AST] --> OS[薄 OXC shim]
    BS --> JSON[规范化输入 / options / scope]
    SS --> JSON
    OS --> JSON
    JSON --> RC[Rust Compiler Core]
    RC --> RES[转换结果 + diagnostics]
    RES --> BS
    RES --> SS
    RES --> OS
~~~

收益假设主要是：

- 避免 Babel、SWC、OXC 各维护一套复杂 compiler pass。
- 强类型 ID、arena 和显式所有权降低某些结构性错误。
- native 边界可为性能提供机会。

第三点在没有同路径基准前只能是待验证假设，不能从“Rust”语言选择直接推出端到端更快。

### 15.3 当前状态与风险

| 主题 | 当前证据 | 主要风险 | 达标证据 |
| --- | --- | --- | --- |
| Babel AST / scope 基础 | 编号计划文档记录若干阶段 complete | 文档可能落后于代码；边角语法差异 | 同一 corpus 的 TS/Rust snapshot parity |
| HIR 与 pass 移植 | crates 已按 lowering、HIR、SSA、validation、optimization 等拆分 | pass 顺序或隐含不变量漂移 | 按 pass 对照、诊断与 codegen 等价 |
| Babel plugin native 接入 | Rust native package、shim 与 CI 路径存在 | N-API 生命周期、错误映射、fallback | 构建产物测试与真实 Babel 集成 |
| SWC/OXC | 计划和 TODO 中有接口/兼容缺口 | AST 语义、scope、codegen 差异 | 各前端同语义 fixture 与差异清零 |
| Unicode/codegen | TODO 记录 WTF-8、lone surrogate、precedence 等边角 | 输出语义或源码位置错误 | 语法 corpus、round-trip 与运行结果 |
| 性能 | 架构具备潜在机会 | 跨 N-API 序列化可能抵消收益 | 冷/热构建、CPU、RSS、产物体积端到端基准 |

计划文档中的 complete 表示该文档记录的工作项状态，不应自动升级为 system_verified。发布前还需以当前 CI、真实插件路径和兼容 corpus 重新确认。

## 16. DevTools、Profiler 与 Fast Refresh

### 16.1 DevTools 架构

[DevTools OVERVIEW](packages/react-devtools/OVERVIEW.md) 描述三层：

~~~mermaid
flowchart LR
    R[React Renderer] -->|inject / commit / unmount| H[Global Hook]
    H --> B[DevTools Backend]
    B <-->|Bridge / postMessage| F[DevTools Frontend]
    F --> UI[Components / Profiler UI]
~~~

__REACT_DEVTOOLS_GLOBAL_HOOK__ 是 renderer 与工具的事实协议面。Backend 与应用/renderer 同上下文运行，Frontend 通过 bridge 接收操作。

为降低观察者效应：

- commit 发送 add/remove/reorder 等紧凑 operation，而不是整棵树。
- frontend 根据 operation 维护镜像树并采用 windowing。
- props/state 按需检查和轮询。
- 深层数据通过 DevTools dehydration 先传 metadata。
- profiling 数据尽量留在 backend，用户查看时再取。

### 16.2 Profiling

Profiler 包含组件 commit/duration 视角以及 timeline 视角。构建还有 profiling bundle 类型；DEV、PROD、PROFILING 不应混成一组性能数字。任何基准都要记录 bundle、渠道、浏览器/Node、硬件、采样开销和 warm-up。

### 16.3 Fast Refresh

[react-refresh README](packages/react-refresh/README.md) 定义 bundler 集成边界。Runtime 通过 register、family 和 Hook signature 识别模块更新；Reconciler 注入 refresh handler：

- 类型 family 兼容且 Hook signature 可保留时，尽量保存状态。
- 签名不兼容或边界失效时 remount。
- bundler 负责模块更新发现和注入，React runtime 不负责 HMR transport。

“刷新后保留了状态”是开发体验策略，不是业务持久化保证。

## 17. Build、Feature Flag 与发布渠道

### 17.1 条件导出

react 包提供 client 默认入口、react-server 条件入口、JSX runtime 与 compiler-runtime。react-dom 根据 client/server/static/profiling 和 Node、browser、edge、Bun、Deno、workerd 等运行时暴露不同入口。

条件导出是架构隔离的一部分：服务端组件环境不应意外获得只属于客户端的能力。

### 17.2 Bundle 类型与 fork

[bundles.js](scripts/rollup/bundles.js) 定义 ISOMORPHIC、RENDERER、RENDERER_UTILS、RECONCILER 等模块类型，以及 NODE、ESM、WWW、RN、DEV、PROD、PROFILING 等产物组合。

[forks.js](scripts/rollup/forks.js) 在构建期按 renderer、环境和 release channel 替换模块；[inlinedHostConfigs.js](scripts/shared/inlinedHostConfigs.js) 维护各 renderer 的 host config 入口。

因此：

- 不能只看 packages/shared/ReactFeatureFlags.js 就断言所有发布渠道行为相同。
- WWW、RN、test renderer 可有专用 flag fork。
- 某段源码可达不代表目标 npm bundle 包含它。
- 默认本地 build channel 与 stable 发布渠道也不能混为一谈。

### 17.3 版本源

[ReactVersions.js](ReactVersions.js) 是发布脚本的重要版本清单；packages 内 package.json 与 [ReactVersion.js](packages/shared/ReactVersion.js) 也参与版本/运行时标识。版本提升需要验证这些来源和实际构建产物一致。

### 17.4 发布渠道

[release README](scripts/release/README.md) 描述：

- canary / experimental 由 CI 构建和发布预发布产物。
- stable 是从已经测试的 canary 候选中人工推进。
- feature flag、fork 和 channel 用于控制兼容面与爆炸半径。

一个 commit 在 main 上通过测试，不等于已经 installed 或 activated 到任何用户环境。

## 18. 测试与证据体系

### 18.1 Runtime

根脚本通过 [Jest CLI](scripts/jest/jest-cli.js) 选择 stable、experimental、WWW classic/modern、xplat，DEV/PROD、persistence 和 built artifact 等矩阵。直接运行一个默认 Jest 配置不能覆盖发布差异。

不传 release channel 时，仓库 Jest CLI 默认选择 experimental；单独执行 yarn test 的通过结果不能自动代表 stable。根 yarn build 则通过 build-all-release-channels 同时组织 stable 与 experimental 产物。

[runtime CI](.github/workflows/runtime_build_and_test.yml) 对矩阵分片，并组合 lint、Flow、build 和测试任务。

### 18.2 Compiler

- TypeScript workspace 有自己的 Jest、lint 和类型检查。
- Rust workspace 通过 Cargo、AST、snapshot 和 port parity 路径。
- Babel/SWC/OXC 接入最终还需要真实 bundler/插件端到端 fixture。

### 18.3 renderer 与 fixture

- noop renderer 用于精确测试 reconciler 调度、flags 和宿主操作。
- test renderer 提供非 DOM 测试面。
- DOM/Fizz/Flight fixture 用于更接近真实集成的验证。
- selective hydration、partial hydration、event replay 等内部测试为边界行为提供 component evidence。

### 18.4 证据分级

| 等级 | 能支持的结论 | 不能外推的结论 |
| --- | --- | --- |
| proxy | microbenchmark、静态计数、模拟队列 | 真实用户端到端延迟 |
| component | 单包测试、noop renderer、局部协议 fixture | 多 renderer、网络、bundler 组合行为 |
| end_to_end | 真实构建 + 浏览器/Node + 完整请求路径 | 长时间稳态与生产分布 |
| soak | 长时间稳定负载、资源曲线与故障恢复 | 未覆盖环境的生产普适性 |
| production | 真实发布流量、版本和观测窗口 | 未来版本或不同配置 |

性能与可靠性结论必须带上这个范围。源码中的调用次数假设、chunk heuristic 或局部 benchmark，不能循环代回模型后自证系统收益。

## 19. 关键技术计划与演进雷达

本节只陈述当前仓库可证明的方向，不把 issue、TODO 或 feature flag 推断成承诺。

| 方向 | 状态分类 | 当前证据 | 下一层关键门槛 |
| --- | --- | --- | --- |
| React Compiler 自动 memo | 已实现主架构，持续演进 | TS pipeline、Babel/ESLint/runtime、CI | 更多真实 corpus 的语义等价、收益和诊断质量 |
| Compiler Rust Port | **明确 active plan** | 根工程说明、编号计划、crates、Rust CI | TS/Rust parity、完整插件路径、多前端、端到端性能 |
| Fizz 流式 SSR / prerender / resume | 已实现能力与持续演进面 | Request/Task/Segment、静态与 resume API、测试 | 框架级产物生命周期、部署配对、背压/abort E2E |
| Flight / RSC 多 bundler | 实验架构与集成面 | server/client core、多 bundler 包、协议测试 | 跨版本协议、manifest 配对、安全和框架 E2E |
| Activity / ViewTransition / Gesture | source-enabled 与实验能力混合 | 公共源码导出、feature flags、专用 build/test | 渠道稳定性、宿主兼容、可访问性、回退 |
| 选择性 Hydration 与事件重放 | 已实现核心能力 | hydration context、event replay、内部测试 | 真实流式网络、复杂输入/第三方 DOM 的 E2E |
| 多 renderer 一致性 | 长期架构责任 | host config、fork、DOM/Native/noop/test | 每次共享算法变更的 renderer 矩阵 |
| DevTools 大树可扩展性 | 已实现架构、持续优化面 | 增量 operation、windowing、lazy inspect | 大树/高频 commit 下 observer overhead |
| 安全边界 | 持续责任 | URL sanitization、Trusted Types 路径、taint、action manifest | 框架 authz/CSRF、部署配置、生产攻击面验证 |

## 20. 跨子系统架构不变量

### 20.1 正确性

- Mutation 完成后 root.current 指向 finished tree；layout/commit error 通过后续更新恢复，而不是回滚宿主 mutation。
- render 可重入、可中断，不得留下不可撤销宿主副作用。
- 状态身份在同一父 Fiber 的 child 集合内由 key/index 候选与 element type 共同决定，不能用索引稳定性假设掩盖列表重排，也不能假定跨父节点移动会保留状态。
- Hook 调用序列在同一组件的 render 间必须一致。
- hydration 不能静默接受改变语义的结构差异。
- Flight/Compiler 的优化或序列化不能改变可观察 JavaScript/React 语义。

### 20.2 性能

- Lane 选择与 Scheduler 让出要一起测，单测其中一个不足以证明交互延迟。
- render 吞吐、commit 阻塞、passive effect、DOM layout 是不同成本中心。
- SSR 要同时测 shell、all-ready、首字节、流式字节、hydration 和可交互时间。
- Compiler 要同时测编译冷/热路径、运行时 render、CPU、RSS 和 bundle size。
- DevTools/Profiler 必须记录是否开启及其观察开销。

### 20.3 兼容性

- feature flag、conditional export、build fork、renderer 和 release channel 都是兼容维度。
- 默认值、错误文本、输出格式、文件路径、网络协议、hook 和恢复语义变化都按潜在 breaking change 审计。
- “当前 fixture 没受影响”不能替代一般兼容性结论。
- Flight manifest、Fizz postponed state 和 client/server bundle 必须有版本配对策略。

### 20.4 安全

- dangerouslySetInnerHTML 是显式信任边界。
- URL sanitization 和 Trusted Types 是纵深防御，不替代业务输入治理。
- Server Action 参数永远来自不可信客户端。
- Flight payload 与 manifest ID 不能被当作授权证明。
- 错误栈、debug 信息和服务器模型不得泄漏 secret。

### 20.5 可恢复性

- render error、thenable、hydration mismatch、stream abort、destination backpressure 是不同状态。
- fallback、client render、retry 和 fatal error 必须有可区分的观测信号。
- retry 要有上限、取消与过期语义，不能靠无限重试掩盖失败。

## 21. 系统级变更的六维发布契约

涉及 scheduler、watcher、daemon、并发、缓存、轮询、重试、持久化、默认值或性能的改动，在实现前应建立下表。没有实测数据时填写“待测”，不能编造基线。

| 维度 | 基线 | 目标 | 测量路径 | 发布阈值 | 证据位置 |
| --- | --- | --- | --- | --- | --- |
| 正确性 | 当前版本的语义/失败率 | 新旧等价或明确新契约 | 单测 + 跨包 E2E + 故障注入 | P0/P1 = 0；既定 invariant 全通过 | 测试日志、trace、diff |
| 端到端延迟 | 真实路径 p50/p95/p99 | 按场景定义，不用局部 proxy 替代 | 输入/请求起点到 paint/stream/action 完成 | 各 percentile 不越预算 | benchmark 原始数据与环境 |
| 稳态负载 | CPU/RSS/磁盘/网络/任务数 | 在目标吞吐下不持续增长 | 插桩真实执行路径并长时采样 | 无无界队列；资源在预算内 | profile、heap、I/O、soak 曲线 |
| 兼容性 | 各 channel/renderer/config 行为 | 支持矩阵内无意外 breaking change | stable/experimental/WWW/RN、旧产物/新产物交叉 | 兼容矩阵全部通过或有迁移说明 | 矩阵结果、迁移文档 |
| 回滚 | 当前回滚耗时与状态格式 | 可在目标时间内恢复 | 关闭 flag、回退 bundle、恢复协议/缓存演练 | 回滚后无孤儿状态或数据损坏 | 演练日志、runbook |
| 产物生命周期与清理 | 当前文件/cache/stream/task | 创建、引用、过期、清理都有 owner | 正常、abort、crash、重启、版本切换 | 无泄漏；旧版本可安全清理 | 目录/队列审计与 soak |

还必须测试组合时序，而不是只测试单个开关：

- Lane + Scheduler callback + browser yield。
- Suspense retry + timeout + transition entanglement。
- Fizz render + stream backpressure + abort。
- Hydration + 离散事件 + event replay + mismatch。
- Flight action + network retry + server idempotency。
- Compiler cache + 并发构建 + 进程失败 + 产物清理。

发布前应让独立审查者只基于原始 diff、配置、日志、trace 和基准做 ship-blocker 盲审，并关闭 P0/P1。组件测试通过只能标 component_verified；只有完整路径证据才能标 system_verified。

## 22. 典型故障模型与定位顺序

### 22.1 “页面卡顿”

不要先假定 Scheduler 有 bug。按因果链分层：

1. native event 到 React update 的时间。
2. update lane 是否符合意图。
3. root callback 是否及时运行、是否 yield。
4. render 工作量与重复 render 原因。
5. commit mutation/layout effect 是否阻塞。
6. 浏览器 style/layout/paint。
7. passive effect 或第三方脚本。

### 22.2 “Hydration mismatch”

依次确认：

1. 服务端和客户端组件输入、环境分支、随机数/时间是否一致。
2. 浏览器是否修正了非法 HTML 嵌套。
3. 第三方脚本/扩展是否在 hydration 前改 DOM。
4. marker 与流式指令是否完整到达。
5. 是 boundary 局部 fallback，还是 root 全量 client render。
6. recoverable error 和事件重放是否符合预期。

### 22.3 “Server Action 不安全”

检查的不是 use server 字符串本身，而是：

1. ID 是否只能从当前部署 manifest 解析。
2. 每次调用是否重新认证和授权。
3. 参数是否校验，错误是否泄密。
4. 是否有 CSRF、重放、幂等与限流策略。
5. action 结果是否可能把 secret 序列化回客户端。

### 22.4 “Compiler 优化没有收益”

分离四类原因：

- 函数因规则违规或 unsupported syntax 未编译。
- 编译成功但 reactive dependency 过宽。
- render 省下了，DOM/layout 或网络仍是主瓶颈。
- 编译/N-API 成本抵消运行时收益。

必须比较同 commit、同 bundler、同模式的编译前后真实路径。

## 23. 开发与验证导航

常用根命令：

~~~bash
# 默认测试入口；支持通过参数选择项目、模式和测试文件
yarn test

# 发布渠道矩阵
yarn test-stable
yarn test-www

# 静态检查
yarn flow
yarn lint
yarn prettier-check
yarn version-check

# 全 release channel 构建，成本较高
yarn build

# 查看 feature flag 组合
yarn flags
~~~

建议阅读路径：

1. 从 [ReactClient.js](packages/react/src/ReactClient.js) 看公共入口。
2. 从 [ReactFiberWorkLoop.js](packages/react-reconciler/src/ReactFiberWorkLoop.js) 看 render/commit 主循环。
3. 用 [ReactFiberLane.js](packages/react-reconciler/src/ReactFiberLane.js) 理解优先级。
4. 用某个 host config 把抽象映射到 DOM 或 Native。
5. 服务端分别沿 ReactFizzServer 与 ReactFlightServer/Client 阅读。
6. Compiler 沿 Pipeline.ts 的 pass 顺序阅读。
7. 最后用 bundles、forks、CI 和 release 脚本确认“哪种构建真正包含什么”。

修改后的最小验证原则：

- 公共 runtime 变更：目标单测 + stable/experimental 对应矩阵。
- reconciler 变更：noop/persistence + DOM + 至少一个相关 server/hydration 路径。
- host config 变更：对应 renderer 测试，不以 DOM 成功外推 Native。
- Fizz/Flight：协议两端、built artifact、abort/error/backpressure。
- Compiler：snapshot/fixture + 运行语义 + 插件真实入口；Rust 变更再做 TS parity。
- build/release：实际 bundle 内容、条件导出、版本和 tarball，而不只测源码。

## 24. 术语表

| 术语 | 本文含义 |
| --- | --- |
| Element | JSX 产生的 UI 描述对象 |
| Fiber | 可恢复的组件/宿主工作单元 |
| Reconciler | 比较描述并计算宿主变化的核心 |
| Renderer | 把 reconciler 操作映射到具体宿主 |
| Lane | React 更新的语义优先级与关系位集合 |
| Scheduler | 控制并发 callback 的执行时机与让出主线程；SyncLane 可绕过它 |
| current tree | mutation 阶段后已切换、当前最近提交的 Fiber 树；后续 commit error 不会回滚它 |
| work-in-progress | 当前正在计算、尚未提交的 Fiber 树 |
| Suspense | 异步 render 的一致性与调度边界 |
| Fizz | React 的流式 HTML 服务端 renderer |
| Flight | Server Component 模型传输协议 |
| Hydration | 客户端 Fiber 接管已有服务器宿主树 |
| Dehydrated boundary | DOM 已有但客户端尚未接管的 Suspense boundary |
| DehydratedFragment | 代表服务器 DOM range 的内部 opaque Fiber |
| Client Reference | Flight 中指向客户端模块/导出的引用 |
| Server Reference | 可由客户端协议调用的服务器入口引用 |
| HIR | Compiler 的高层中间表示和控制流图 |
| Reactive Scope | Compiler 推导出的可一起缓存及失效的计算区域 |
| Build fork | 按渠道/renderer/环境替换实现模块 |

## 25. 关键源码索引

### Runtime 与协调

- [React public client entry](packages/react/src/ReactClient.js)
- [React server conditional entry](packages/react/src/ReactServer.js)
- [Element construction](packages/react/src/jsx/ReactJSXElement.js)
- [Public Hook dispatch](packages/react/src/ReactHooks.js)
- [Fiber internal types](packages/react-reconciler/src/ReactInternalTypes.js)
- [Fiber work loop](packages/react-reconciler/src/ReactFiberWorkLoop.js)
- [Fiber lanes](packages/react-reconciler/src/ReactFiberLane.js)
- [Hooks implementation](packages/react-reconciler/src/ReactFiberHooks.js)
- [Hydration context](packages/react-reconciler/src/ReactFiberHydrationContext.js)
- [Scheduler](packages/scheduler/src/forks/Scheduler.js)

### DOM 与服务端

- [DOM root](packages/react-dom/src/client/ReactDOMRoot.js)
- [DOM host config](packages/react-dom-bindings/src/client/ReactFiberConfigDOM.js)
- [DOM event plugin system](packages/react-dom-bindings/src/events/DOMPluginEventSystem.js)
- [DOM event replay](packages/react-dom-bindings/src/events/ReactDOMEventReplaying.js)
- [Fizz server](packages/react-server/src/ReactFizzServer.js)
- [Flight server](packages/react-server/src/ReactFlightServer.js)
- [Flight client](packages/react-client/src/ReactFlightClient.js)
- [React server architecture note](packages/react-server/README.md)

### Compiler、工具与发布

- [Compiler pipeline](compiler/packages/babel-plugin-react-compiler/src/Entrypoint/Pipeline.ts)
- [Compiler program/options](compiler/packages/babel-plugin-react-compiler/src/Entrypoint/Program.ts)
- [Compiler design goals](compiler/docs/DESIGN_GOALS.md)
- [Rust port architecture](compiler/docs/rust-port/rust-port-architecture.md)
- [DevTools architecture](packages/react-devtools/OVERVIEW.md)
- [Bundle definitions](scripts/rollup/bundles.js)
- [Build forks](scripts/rollup/forks.js)
- [Renderer host config inventory](scripts/shared/inlinedHostConfigs.js)
- [Runtime CI](.github/workflows/runtime_build_and_test.yml)
- [Compiler Rust CI](.github/workflows/compiler_rust.yml)
- [Release process](scripts/release/README.md)

---

维护本文时，应同时更新提交基线、证据范围和状态分类。新增“路线图”结论至少需要活跃计划文档、实现或 CI 三者中的明确证据；单独的 TODO、flag 名称或测试占位不得升级为正式承诺。
