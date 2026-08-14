# 面向 AI 工程师和软件开发者的 Cordis 报告

## 摘要

Cordis 是一个插件运行时. 它让应用可以在进程继续运行时添加, 删除, 替换和重新配置插件. 它主要解决共享同一进程的插件如何管理依赖, 资源和生命周期的问题.

核心工作方式如下:

1. 插件声明自己需要哪些服务.
2. 插件为创建的每个资源登记清理操作.
3. 只有依赖齐全时, Cordis 才激活插件.
4. 如果依赖消失或被替换, Cordis 按安全顺序停用受影响的插件.
5. 卸载或启动失败时, Cordis 按相反顺序运行已经记录的清理操作.

对于 LLM Agent 平台, 工具, Memory Backend, Model Provider, Policy Module, Planner 和外部集成都可以成为受管理的运行时组件. Agent Host 可以替换模型供应商, 禁用故障工具, 重载生成代码, 或切换租户专用服务, 而不必重启整个进程.

Cordis 不能让任意代码自动变得可逆或安全. 清理逻辑仍然必须正确. 发送消息, 扣款等外部行为通常无法撤销. 不可信插件仍然需要进程或容器级隔离. 最准确的定位是: Cordis 是进程内组件的生命周期和依赖控制平面.

## 背景

### 普通插件系统为什么会变复杂

很多插件系统最初只有一个简单接口:

- 加载模块.
- 把应用对象传给模块.
- 让模块注册 Handler.
- 关闭时可选调用一个 Shutdown Hook.

只要应用不需要在线变更, 这种方式通常够用. 一旦要求动态卸载或热更新, 问题就会出现. 一个插件可能已经注册 Event Listener, Route, Timer, Background Job, Tool, Database Connection, Middleware, Child Plugin, 或被其他插件使用的 Service.

安全删除插件需要回答:

- 哪些资源属于这个插件?
- 应该按照什么顺序释放?
- 哪些插件依赖它?
- 依赖方清理时还能否继续使用 Provider?
- 如果启动执行到一半失败会怎样?
- 异步启动期间配置改变会怎样?
- 热更新是否会留下重复 Handler 或过期 Service Reference?

传统 Lifecycle Callback 要求插件作者手动记住所有细节. 重启进程虽然能回收部分资源, 但也会丢失 Cache, Session, Connection 和正在执行的任务. 对长时间运行的 Agent Host 来说, 代价尤其明显.

### 为什么 LLM Agent 需要这种能力

生产级 Agent 系统通常不只是一次模型调用. 一个运行时可能包含:

- Model Provider 和 Embedding Provider.
- Tool Registry 和 MCP Client.
- Retrieval 和 Memory Service.
- Planner, Router 和 Evaluator.
- Authorization 和 Policy Component.
- Tracing, Billing 和 Rate Limiting.
- 用户或租户专用扩展.
- 模型生成的代码和实验性行为.

这些组件变化速度不同. Tool 可能临时不可用. Credential 可能轮换. Provider 可能被替换. 新生成的组件可能在激活过程中失败. 管理员可能只想禁用一个集成, 而不是中断所有正在运行的 Agent.

Cordis 解决的就是这些变化带来的进程内生命周期问题.

## 1. Cordis 做什么

Cordis 把通常分散实现的 4 个职责合并到一个运行时中.

### 1.1 跟踪插件拥有的 Effect

插件改变运行中应用时就产生了 Effect. 常见例子包括:

- 注册 Event Listener.
- 添加 Route 或 Command.
- 启动 Timer 或 Worker.
- 发布 Service.
- 打开一个具有 Close 操作的资源.
- 加载 Child Plugin.

在 Cordis 中, 插件通过 `ctx.effect` 登记变化, 并返回或 yield 一个 Disposer. Cordis 保存 Disposer, 以后按照 Last In First Out 顺序运行.

概念示例:

~~~ts
ctx.effect(() => {
  const timer = setInterval(runHealthCheck, 10_000)
  return () => clearInterval(timer)
})
~~~

这种方式比把 Setup 和 Teardown 分散到两个 Callback 中更可靠. 清理逻辑紧邻它所撤销的操作, 而且清理栈由 Cordis 统一管理.

### 1.2 管理 Service Dependency

插件声明自己需要哪些 Service, 以及提供哪些 Service. Consumer 在所有依赖存在之前保持 Inactive. 如果 Provider 被删除或替换, Cordis 找到绑定到它的 Consumer, 让这些 Consumer 停用, 并在条件满足时重新激活.

这是一种动态 Dependency Injection. 它和普通 DI Container 的关键差别是: 依赖解析不只发生在应用启动时. Cordis 在整个进程生命周期中持续管理 Service Availability.

### 1.3 协调 Lifecycle 顺序

Cordis 把每个插件实例表示为 Fiber. Fiber 可以处于 Inactive, Loading, Active 或 Unloading 状态. 它记录:

- 插件实例和 Parent.
- Required Service 和 Provided Service.
- 当前激活实际选中的 Provider.
- 已累积的 Cleanup Operation.
- Lifecycle State 和 Failure Information.

删除 Provider 时采用受保护的顺序:

1. Provider 先停止接受新的 Consumer.
2. 通知已经绑定的 Consumer, 其依赖即将离开.
3. Consumer 开始卸载, 但清理期间仍然可以使用旧 Provider.
4. Provider 等待这些 Consumer 完成清理.
5. Provider 最后清理自己的资源.

这个顺序避免常见的 Shutdown Bug. 例如 Database, Transport 或 Model Client 已经被销毁, 但它们的 Consumer 仍然需要使用它们完成清理.

### 1.4 让期望配置和运行状态保持一致

Cordis Loader 把配置视为 Desired State. 它比较期望 Plugin Tree 和当前运行的 Tree, 然后应用最小范围的变化:

- 创建或删除插件.
- 启用或禁用插件.
- 更新配置.
- 修改 Isolation 或 Interception.
- 模块变化时重建 Component.

Hot Module Replacement 使用同一套 Lifecycle System. Cordis 找出受影响的插件实例, 卸载它们的 Effect, 让相关模块 Cache 失效, 然后创建替代实例. 如果新版本 Import 失败, 它可以恢复旧模块状态并重建原来的实例.

## 2. Cordis Plugin 和 Hook 有什么不同

Hook 和 Cordis Plugin 解决的问题不同.

| 关注点 | Hook 或 Callback | Cordis Plugin |
|---|---|---|
| 目标 | 扩展一个 Event 或 Phase | 拥有一个运行时组件及其完整 Lifecycle |
| 调用方式 | Host 调用已注册函数 | 依赖满足时由 Runtime 激活 Component |
| 依赖 | 通常隐式存在或手动查询 | 显式声明并持续监控 |
| 清理 | 单独编写 Unsubscribe 或 Shutdown | 自动记录和组合 Disposer |
| 部分启动失败 | 通常手动处理 | 可以回滚已经完成的 Effect |
| Provider 替换 | Hook 往往保留过期引用 | Consumer 卸载后重新解析 |
| Hot Reload | 由应用单独实现 | 复用同一套受管理 Lifecycle |
| Child Component | 通常只是约定 | 创建 Child Plugin 本身也是 Tracked Effect |
| 安全边界 | 没有 | 有 Lifecycle Confinement, 但不是 Security Sandbox |

Hook 是 Extension Point. Cordis Plugin 是一个拥有 State, Dependency, Effect 和 Recovery 的运行时单元. 一个 Cordis Plugin 可以注册很多 Hook, 但 Plugin Lifecycle 决定这些 Hook 何时存在, 并确保它们的 Disposer 参与卸载.

例如, `onMessage` Hook 只回答 "收到消息时运行什么代码?" Cordis 还回答:

- 谁拥有这次注册?
- 安装前必须有哪些 Service?
- 如何删除它?
- 安装到一半停止怎么办?
- Model Provider 或 Database Provider 被替换怎么办?

### 2.1 和常见 Plugin 及 Runtime System 的对比

论文使用 VS Code 作为最详细的动机案例. 它还把 IntelliJ, Eclipse, OSGi, Dependency Injection, React Effect, HMR 和 Reactive Framework 作为相邻方案进行讨论. 下表把这些内容转换成工程师更容易使用的比较.

| System 或 Model | 擅长的事情 | Runtime Change 时会怎样 | 和 Cordis 的主要差别 |
|---|---|---|---|
| VS Code Extension | 为 Command, View 和 Language Feature 提供丰富且稳定的 Host Defined Extension Point | Executable Extension 共享一个 Extension Host. 根据论文, Disable 或 Uninstall 一个插件通常要求重启 Extension Host. `deactivate` 是 Graceful Shutdown Callback, 不是单插件在线卸载机制 | Cordis 把每个 Plugin 作为可独立卸载的 Fiber, 并把 Setup 和累积 Cleanup 关联起来 |
| IntelliJ Plugin | 成熟的 IDE Extension 生态和显式 Plugin Lifecycle Convention | 论文把 IntelliJ 归入依赖 Developer Authored Unload Callback 的系统 | Cordis 仍要求开发者为 Atomic Effect 提供 Disposer, 但会为整个 Plugin 自动累积和执行 Composite Cleanup |
| Eclipse Extension Point | 结构化 Host Extension Point 和大型模块化生态 | 论文把 Eclipse 归入 Callback Based Cleanup. Extension Author 仍然负责完整 Teardown | Cordis 把 Effect Ownership 和 Recovery 变成 Runtime Component Model 的一部分 |
| OSGi Declarative Services | 声明 Provided Service 和 Required Service, 并根据 Service Availability 激活 | Service 出现和消失时, Component 可以自动响应 | 这是最接近 Cordis Spatial Model 的先例. Cordis 额外提供累积 Effect Recovery 和可等待的 Guarded Async Unload |
| iPOJO 和 Gravity | 使用 Provide 和 Require Model 对变化中的 Service 进行自主适配 | Runtime Availability 驱动 Component Activation 和 Deactivation | 和 OSGi DS 类似, Recovery 依赖手写且同步的 Deactivation Callback. Cordis 可以等待 Teardown, 并在 Consumer Drain 期间保留旧 Provider |
| Spring, Guice, Angular DI, Inversify | 启动时进行强类型 Dependency Wiring, 部分系统支持 Scope 或 Hierarchical Injector | Provider 消失或 Identity 改变时, 现有 Consumer 通常不会自动停用和重建 | Cordis 持续重新计算 Dependency Satisfaction, 并把结果连接到 Lifecycle Transition |
| React `useEffect` | 把 Effect 和 Cleanup 放在一起 | Re Execution 和 Unmount 前运行 Cleanup | 它受 React Hook Rule 和 UI Lifecycle 限制, 不提供通用 Cross Plugin Service Graph, 也不能自由组合 Async Effect Iterator |
| webpack 或 Vite HMR | 快速替换 Module, 并提供可选 State Handoff | Module 使用显式 Acceptance 和 Migration API 保留状态 | Cordis 先卸载 Tracked Effect, 再从干净 Lifecycle Boundary 应用新 Component. 只有移入 Longer Lived Service 的状态会保留 |
| FRP 和 Signal | 细粒度 Value Propagation 和 Derived State Update | Value 和 Computation 在 Reactive Scheduler 下更新 | Cordis 在 Component Granularity 上响应变化, 并协调异步 Activate 和 Unload |
| Process 和 Container | 强隔离和可靠的粗粒度资源回收 | 替换完整 Process 或 Service Instance | Cordis 只改变同一进程内较小的区域, 并保留无关 Process Local State. 但 Cordis 不提供 Security Sandbox |

论文给出了一组在 2026-06-09 收集的 VS Code 生态数据. 安装量最高的 100 个 Extension 中, 87 个包含 Executable Code, 因此移除时需要重启 Extension Host. 只有 7 个声明了对非内置 Extension 的依赖. 作者用这组数据说明两个问题: Unload Boundary 过于粗糙, 并且生态主要围绕 Host Provided Extension Point 构建, 而不是 Typed Plugin To Plugin Composition.

重点不是 Cordis 替代上表所有系统. 它把通常分散存在的能力组合在一起:

- IDE Extension Point 定义 Plugin 如何贡献功能.
- OSGi Style Service 定义 Component 如何发现彼此.
- React Style Effect 把 Setup 和 Cleanup 配对.
- HMR 替换代码.
- Process Supervisor 提供 Fault Boundary.

Cordis 在同一个 In Process Component Boundary 中组合 Service Availability, Effect Ownership, Cleanup Composition, Dependency Aware Ordering 和 Code Replacement.

### 2.2 用不同系统处理同一次变更

假设运行中的 Research Agent 依赖一个 Memory Provider, 现在必须替换这个 Provider.

| Approach | 常见工程处理方式 |
|---|---|
| Callback Based IDE Plugin | Plugin Author 手动注销所有资源. Host 可能要求重启 Extension Host 或整个应用 |
| Startup Time DI Container | 创建新 Container 或重启相关 Application Scope. 现有 Consumer 通常继续持有旧 Reference |
| OSGi DS 或 iPOJO | Service Withdrawal 可以停用 Consumer, 但 Teardown 仍然由 Callback 驱动, 并且论文描述其为同步机制 |
| Module HMR | 重新加载变化的 Module, 并使用 Framework Specific Acceptance 或 State Migration Code |
| Process 或 Container Replacement | 重启整个 Runtime Unit, 然后重新构建它的状态 |
| Cordis | 撤回 Binding, Drain 已绑定的 Consumer, 运行它们累积的 Disposer, 释放旧 Provider, 解析新 Provider, 并且只重新激活受影响区域 |

~~~mermaid
flowchart LR
    A["请求替换 Memory Provider"] --> B["Cordis 停止发布旧 Provider"]
    B --> C["已绑定的 Agent Plugin 变成 Stale"]
    C --> D["Agent 卸载 Tracked Effect"]
    D --> E["旧 Provider 等待 Consumer Drain"]
    E --> F["旧 Provider 释放资源"]
    F --> G["新 Provider 变为 Available"]
    G --> H["只重新激活受影响的 Agent"]
    U["无关 Plugin"] --> V["继续保持 Active"]
~~~

## 3. Lifecycle 在实际系统中如何运行

### 3.1 正常激活

假设一个 Agent Plugin 依赖 `model` 和 `memory`, 并提供 `researchAgent`.

1. Fiber 被创建, 但保持 Inactive.
2. Cordis 等待两个 Required Service 都可用.
3. Cordis 记录本次激活选中的具体 Model Fiber 和 Memory Fiber.
4. 插件开始安装 Effect.
5. 每个成功步骤都贡献一个 Disposer.
6. 启动完成后, `researchAgent` 才对 Consumer 可用.

Provider Identity 很重要. 即使替代 Provider 返回看起来相同的对象, 它仍然是一个新 Provider, Consumer 可能需要重建内部状态.

### 3.2 替换 Provider

如果 Memory Provider 被替换:

1. 旧 Memory Provider 停止接受新依赖方.
2. Agent Plugin 因 Committed Provider 改变而变成 Stale.
3. Agent 卸载 Handler, Job 和它发布的 Service.
4. 旧 Memory Provider 等待 Agent 完成卸载.
5. 旧 Provider 释放资源.
6. 新 Provider 以及其他依赖全部可用后, Agent 再次激活.

只有受影响的 Dependency Region 需要变化. 无关插件继续保持 Active.

### 3.3 部分失败和取消

Cordis 通过 Effect Iterator 支持多步骤异步激活. 插件可以在每个步骤完成后 yield 一个 Disposer. 如果第 4 步失败, Cordis 可以依次撤销第 3, 2, 1 步.

如果一个异步步骤已经开始, 但执行期间依赖发生变化, Cordis 会让当前步骤完成, 记录它的 Cleanup, 然后立即卸载这次已经过期的激活. 它不会把过期插件短暂暴露为 Active.

这对初始化较慢的 Agent Component 很有价值. 例如建立 MCP Connection, 预热 Model Client, 加载 Index, 或启动 Sandbox.

同一行为也可以表示为 State Machine:

~~~mermaid
stateDiagram-v2
    [*] --> Inactive
    Inactive --> Loading: 所有依赖可用
    Loading --> Active: 激活完成
    Loading --> Unloading: Failure 或 Dependency Target 改变
    Active --> Unloading: Retire 或 Provider Withdrawal
    Unloading --> Inactive: 累积 Cleanup 完成
    Inactive --> Loading: 新 Target 有效
~~~

## 4. 一个具体的 LLM Agent 示例

考虑一个 Multi Tenant Agent Host:

- `openaiModel` 和 `localModel` 在不同 Isolated Context 中提供 `model`.
- `vectorMemory` 提供 `memory`.
- `webTools` 提供 Tool Collection.
- `policyGuard` 拦截 Tool Access.
- `researchAgent` 依赖 `model`, `memory` 和 `tools`.
- `evaluationWorker` 依赖 `researchAgent` 和 `telemetry`.

Cordis 可以支持:

- 替换 `vectorMemory`, 而不重启 Model Client 或无关 Agent.
- 在同一逻辑 Service Name 下, 让一个租户使用 `localModel`, 另一个租户使用 `openaiModel`.
- 禁用 `webTools`, 并只停用依赖它的 Agent.
- 更新 Tool Access Policy Metadata, 但不改变 Tool Dependency 是否存在.
- 热重载 `researchAgent`, 同时保留未受影响的基础设施.
- 如果新版 Agent 启动失败, 回滚已经安装的 Route, Event Subscription 和 Worker.

整体结构如下:

~~~text
desired configuration
        |
        v
Cordis loader and reconciliation
        |
        v
plugin fibers and dependency bindings
        |
        +---- tracked effects and disposers
        +---- provided and required services
        +---- isolation and interception
        +---- reload, unload, and failure recovery
~~~

Cordis 是 Lifecycle Control Plane. 它不会替代 Model Gateway, Tool Protocol, Memory Database, Message Bus 或 Security Sandbox.

## 5. 对 LLM Agent 行业的潜在影响

### 5.1 从静态 Agent 走向动态 Agent Runtime

多数 Agent Framework 擅长定义 Graph 或 Workflow, 然后执行它. Cordis 指向另一类系统: Agent Host 在继续服务的同时改变内部 Component Graph.

这类能力适合 Long Running Assistant, Desktop Agent, IDE Agent 和 Multi Tenant Automation Platform. 在这些场景中, 每次组件变化都重启整个 Host 会造成明显中断.

### 5.2 更安全的 Self Modification

论文把 Self Evolving Agent Harness 作为一个动机. Model 生成的修改可以被安装为新 Component, 接受观察, 并在激活失败时移除.

实际收益是限制 Lifecycle Damage, 而不是证明生成行为正确. 生成组件仍然必须:

- 声明每一个依赖.
- 让所有受管理 Effect 经过 Context.
- 提供正确且 Idempotent 的 Cleanup.
- 避免不可逆外部行为, 或提供 Compensation.
- 如果代码不可信, 在真正的 Sandbox 后运行.

因此, Cordis 可以成为 Self Modification Pipeline 的一层:

~~~text
generate -> validate -> sandbox -> activate -> observe -> retain or unload
~~~

它主要增强 Activate 和 Unload 阶段.

### 5.3 更细粒度的故障隔离

如果一个插件在激活时失败, Cordis 可以撤销该插件已经完成的 Effect, 并记录局部 Failure, 而不必停止 Sibling Component. 对 Agent 平台来说, 这可以缩小损坏的 Tool Adapter, Evaluator 或 Provider Integration 的影响范围.

这是 Lifecycle Isolation, 不是 Memory Isolation 或 Security Isolation. 如果没有更强的 Runtime Boundary, 插件仍然可能让进程崩溃, 过度消耗资源, 修改 Global State, 或绕过 Context API.

### 5.4 降低 Stateful Upgrade 的中断

Agent Infrastructure 经常保存昂贵的进程内状态. 例如 Connection Pool, Cache, Loaded Index, Model Session, Browser Session 和 Task Queue. Selective Reconciliation 可以保留未受影响的状态, 只替换相关 Component 和它们的 Dependent.

这可能改善开发迭代速度和运行可用性. 但是论文没有提供 Reload Latency, Memory Overhead 或 Throughput Benchmark.

### 5.5 为 Agent 生态提供统一组件契约

如果所有 Component 对以下内容使用共同契约, 生态会更容易组合:

- Dependency.
- Service Publication.
- Resource Ownership.
- Cleanup.
- Configuration Change.
- Failure State.

Cordis 提供了这种契约. 如果 Tool 和 Agent Framework 作者采用类似模型, 可以减少 Framework Specific Glue. 但真正的可移植性仍然依赖共同 Service Interface, Versioning 和 Language Binding.

### 5.6 支持 Evaluation 和 Rollout 基础设施

因为 Plugin Identity 和 Lifecycle 都是显式的, Agent 平台可以围绕它们构建:

- Planner 或 Model Router 的 Canary Replacement.
- Tenant Scoped Service Selection.
- Health Check 失败后的自动 Rollback.
- 以 Plugin Fiber 为单位的 Tracing.
- Activation, Drain 和 Cleanup Latency 监控.
- Tool Access 或 Model Access 的 Policy Interception.

其中一些是该架构自然支持的扩展方向, 并不是论文已经完整实现和验证的功能.

## 6. Cordis 不解决什么

### 6.1 不可逆的外部操作

发送 Email, 发布 Message, 执行 Trade 或向客户扣款都属于 Local Runtime 之外的 Emission. Disposer 无法让已经发生的事实消失. 这些操作需要 Delayed Commit, Idempotency Key, Transactional Outbox 或领域专用 Compensation.

### 6.2 错误的 Cleanup

Cordis 可以按照正确顺序调用 Disposer, 但不能证明 Disposer 完整或语义正确. Plugin Test 应覆盖重复 Unload, 部分初始化失败和 Provider Replacement.

### 6.3 Global 和 Ambient State

直接修改 Global Object, Hidden Singleton, Unmanaged Thread 和 Third Party Library 都可能逃离 Ownership Tracking. 强保证只适用于由 Cordis Context 中介的 Effect.

### 6.4 Security Isolation

Dependency Access Check 和 Interception 类似 Capability Control, 但同一进程内的 JavaScript 不是 Sandbox. 不可信或模型生成的代码需要 Worker, Process, Virtual Machine 或 Container Boundary, 并通过 Narrow Bridge 访问服务.

### 6.5 循环依赖

Dependency Cycle 会让所有参与方一直等待. Cordis 假设 Provider Dependency Graph 无环. 团队可能需要提取 Shared Core Service 或引入 Broker 来消除 Cycle.

### 6.6 尚未证明的生产成本

Koishi 生态证明底层组合模型有大规模实际应用, 并拥有超过 4000 个 Community Plugin. 但论文说明 Koishi 使用 Cordis v3, 而论文描述的是 v4. 论文没有提供 Throughput, Memory, Recovery Time 或 Developer Productivity 的受控比较和量化结果.

## 7. 工程采用指南

### 适合的场景

以下情况适合考虑 Cordis:

- 一个进程承载许多独立管理的 Extension.
- 插件拥有复杂资源和 Dependency.
- Live Configuration 和 Hot Reload 很重要.
- 需要保留未受影响的进程内状态.
- 故障应该保持局部.
- 团队可以强制使用 Context 管理资源所有权.

例子包括 Agent Development Server, IDE Agent Host, Multi Tenant Bot Platform, 或支持动态安装 Adapter 的 Tool Gateway.

### 不适合的场景

以下情况中 Cordis 价值较低:

- 应用很小, 几秒内可以安全重启.
- Component 已经运行在独立进程或容器中.
- 大部分行为都是不可逆的外部 Emission.
- Plugin Author 无法遵守 Resource Ownership 规则.
- Dependency 高度循环, 或只能通过 Global State 发现.

### 最低生产检查清单

1. 把每个 Listener, Timer, Route, Worker, Provider 和 Child Plugin 包装为 Tracked Effect.
2. 让 Cleanup 支持 Idempotency, 并能处理 Partial Initialization.
3. 声明每个 Service Dependency, 避免 Ambient Lookup.
4. 测试异步激活期间发生 Provider Replacement 的情况.
5. 区分可逆 Local State 和不可逆 External Action.
6. 对 Emission 使用 Durable Workflow 或 Transaction Pattern.
7. 把不可信和 Model Generated Code 放入真正的 Sandbox.
8. 激活前检测 Dependency Cycle.
9. 监控 Activation Time, Unload Time, Dependent Drain Time 和 Disposer Failure.
10. 按预期 Plugin 数量进行 Reconciliation 和 Hot Reload Load Test.

## 8. 对论文的工程评价

论文最强的贡献是提供一个连贯的工程模型, 并用形式化推理解释为什么它可以成立. 对开发者来说, 形式部分支持以下预期:

- 如果插件之间的 Effect 相互独立, 一个插件完成的 Effect 可以被删除, 而不删除无关 Effect.
- Consumer 在 Provider 之后激活, 并在 Provider 消失前卸载.
- 一次 Plugin Activation 使用一套稳定的 Dependency Resolution.
- 在明确假设成立时, 不同 Reconfiguration Schedule 会到达等价 Final State.

实践中, 这些是 Design Constraint, 不是自动获得的保证. 它们依赖正确 Cleanup, Independent Effect, 有限且无环的 Dependency, 以及对 Context 的严格使用. Failure 和不可逆 Emission 不在最强 Convergence Result 的覆盖范围内.

作为 Cordis 的架构说明, 论文具有说服力. 但它还不是替换成熟 Agent Runtime 的完整商业和性能依据. 下一步最有价值的证据包括:

- 和 Restart Based 及 Callback Based Plugin System 的 Benchmark.
- Failure Injection Test.
- Property Based Lifecycle Test.
- Cordis v4 的生产评估.
- 包含大量 Tool 和 Provider 的 Agent Workload Measurement.
- Model Generated Plugin 的 Security Analysis.

## 最终总结

Cordis 把 Plugin 从 "由 Host 调用的一段代码" 变成受管理的运行时对象. 这个对象拥有显式 Dependency, Owned Effect, Recorded Cleanup 和 Controlled Lifecycle.

对普通软件来说, 它支持进程内的 Targeted Unload, Dependency Aware Restart 和更安全的 Hot Reload. 对 LLM Agent 平台来说, 它为动态替换 Tool, Provider, Policy, Memory System 和 Generated Component 提供基础, 而不必重启整个 Agent Host.

它对行业最可能的影响不是新的 Prompting Method 或 Agent Reasoning Algorithm. 它是一层 Infrastructure, 用更严格的方法运行可变且长时间存活的 Agent System. Cordis 可以减少 Stale Registration, Resource Leak, Invalid Provider Reference 和 System Wide Restart. 它本身不能验证 Agent Behavior, 撤销 External Action, 或安全容纳 Hostile Code.

最合理的采用方式是把它放在更完整的 Agent Platform 中:

~~~text
security sandbox + durable workflow + Cordis lifecycle + observability
~~~

在这个边界内, Cordis 为需要在运行时持续演化的 Plugin Based Agent Runtime 提供了有吸引力的模型.
