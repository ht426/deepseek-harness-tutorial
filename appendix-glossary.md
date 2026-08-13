# 附录：术语速查

本附录按主题给出 `dsh` 的规范术语（完整版见 [`docs/glossary.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/glossary.md)）。代码标识符保留英文，首次出现处给中文。

## 框架与组合

- **插件（plugin）**：实现 Service 的对象——带可选 `inject` 与 `apply(ctx)` 的函数，或 `Service` 子类。
- **Context（上下文）**：服务的仓库；服务认领 `ctx.<key>`，其它插件按 key 查找。
- **服务（service）**：命名能力；一个插件提供，其它插件经 `ctx` 消费。
- **`inject`**：声明硬依赖；插件停在 PENDING 直到依赖存在。
- **事件（event）**：类型化通信；按 `emit` / `parallel` / `serial` / `bail` / `waterfall` 分发。
- **effect**：可逆注册；`ctx.effect()` / `ctx.on()` 安装，卸载时回退。
- **waterfall**：环绕式中间件；监听器收 `(...args, next)`，调用 `next()` 委托，不调用即短路。
- **profile**：Harness home 里的命名组合；列出 bundle、树外插件与用户 `cordis.patch.yml`。
- **bundle**：Cordis 配置行及其代码的分发格式（`package.json` 的 `dsh.bundle` 指向补丁文件）。
- **preset**：每会话 agent 组合；一份 `agent.cordis.yml` 挂到 agent scope 下。

## 能力接缝（capability seam）

- **seam（接缝）**：一种可替换能力，含三种角色：
  - **Service Definition**：拥有 `ctx.<key>` 与词汇类型的 Cordis `Service`（抽象类或注册表，不是 `interface`）。
  - **Service Provider**：实现接口的一方。
  - **Consumer**：注入并使用的服务的一方（常见形态是模型面工具）。
- 三角色构成完整能力；换 provider 不碰 Service Definition 与 Consumer。

## 作用域（agent-scope）

- **scope**：按 agent 的注册单位；贡献要么全局、要么归属一个 scope key。
- **scope key**：scope 的不透明身份（活跃 agent 是它自己 scope 的 key）。
- **`agent.ctx`**：agent 的作用域上下文；经它注册既 scope 可见又 scope 生命周期。
- **shadowing**：最具体者胜出——作用域工具/section 在该 scope 内替换同名全局项。
- **restriction**：`tools.restrict` 为单个 scope 过滤全局工具集（取交集）。
- **setup window**：创建时隙——scope 与 agent 对象已存在、尚未发布、首个 prompt 未组装。
- **lineage**：父子关系事实（`parentSession`、`delegationDepth`、`subagentDepth`），不影响可见性。

## 循环层级

- **turn（轮次）**：会话中一次对已接纳输入的排空，模型与工具停止或终止策略介入后结束。
- **step（步骤）**：一次模型请求 + 它引发的工具执行；一个 turn 含零或多个 step。
- **round**：承载 turn 的外层策略迭代（如 Goal Round 或 Ralph Round）。

## 目标与 Ralph

- **goal（目标）**：附着会话的单个持久完成目标，阶段 `active`/`paused`/`blocked`/`complete`，带 revision 与回合上限。
- **Goal Round**：为当前目标接纳的一次续行周期，具体化为一个 goal-sourced turn。
- **goal activation**：进程本地权限（`armed`/`disarmed`），不持久化。
- **Ralph loop / Ralph Round / Ralph handoff**：面向不可变目标的前台全新 agent 工作流 / 其全新子会话 / 跨轮的有界结构化交接。

## 会话与数据

- **Session**：类型化 `SessionEvent` 的追加式日志——唯一真源。
- **`deriveMessages()`**：从日志投影出 LLM 消息历史。
- **模型可见 ⟺ 已入日志**：任何进入模型请求的内容必须能从日志重建（运行时不变式断言）。
- **projection（投影）**：整日志派生的当前值，供客户端载体消费。
- **persistence（持久化）**：`ctx.sessionPersistence` 接缝 + JSONL/SQLite 后端。
- **`SessionEventMap` / `…Map → derived-union`**：由判别 tag 键控的接口派生 union；插件 declaration merging 扩展。
- **Branded ID**：结构上是字符串、类型上不可互换的跨边界 id（`Branded<B>`）。

## 交互

- **human command（人类命令）**：斜杠前缀指令，经 `ctx.commands` 解释执行，不成为模型消息。
- **approval（审批）**：一次性权限决定，经 `approval/request` waterfall 派发；无人作答 fail closed 到 `unavailable`。
- **ask-user**：模型经 `tool-ask-user` 在 `ctx.userQuestions` 的 `ask()` promise 上暂停工具调用、向人提问。

## 常用 `ctx` 服务速查

| key | 拥有者 | 角色 |
|---|---|---|
| `ctx.sessions` | `dsh-session` | 会话日志 + 内存存储（core） |
| `ctx.systemPrompt` | `dsh-system-prompt` | prompt 段与工具 schema 组装（core） |
| `ctx.tools` | `dsh-tools` | 工具注册表 + 受守卫执行流水线（core） |
| `ctx.agents` | `dsh-agent` | 活跃 Agent 注册表 + initiator 作用域（core） |
| `ctx.agentLoop` | `dsh-agent-loop` | 具体循环 driver |
| `ctx.llm` | `dsh-llm` | LLM 适配器注册表（seam） |
| `ctx.shell` / `ctx.subprocess` / `ctx.terminals` | shell/subprocess/terminal | 执行接缝 |
| `ctx.sandbox` / `ctx.sandboxPolicy` | sandbox | 进程约束接缝 / 策略归属 |
| `ctx.fs` / `ctx.lsp` / `ctx.skills` / `ctx.web` | 各自组 | 数据/工具接缝 |
| `ctx.subagents` / `ctx.workflowEngine` / `ctx.jobs` / `ctx.goals` / `ctx.compaction` | 各自组 | 编排 |
| `ctx.sessionPersistence` / `ctx.sessionProjections` / `ctx.sessionQuery` / `ctx.sessionTitle` / `ctx.sessionTelemetry` | session 各组 | 数据平面 |
| `ctx.settings` / `ctx.credentials` / `ctx.storage` / `ctx.storageDomain` / `ctx.workspaceRegistry` | 各自组 | 配置/存储 |
| `ctx.commands` / `ctx.approval` / `ctx.permissionPresets` / `ctx.userQuestions` | interaction | 人类协作 |
| `ctx.apiProxy` / `ctx.webServer` / `ctx.clientModules` / `ctx.directoryPicker` | host | Web GUI host 半 |
| `ctx.typert` / `ctx.typertGateway` | typert/api | 类型图 / 远程调用 |

完整清单与实现/消费者关系见 [`docs/capability-seams.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md)。
