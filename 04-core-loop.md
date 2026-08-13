# 第 4 章：核心与 Agent 循环

这一章讲 `dsh` 的**产品 API 主轴**（[`packages/core/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/core/README.md)）：会话日志、系统提示组装、工具注册表、agent 词汇、默认模型选择，以及驱动它们的具体循环。这是每个组合都会启动的包，也是插件与消费者构建所依赖的稳定面。

## 4.1 六个包与一个循环

一个 turn 流经六个包，形成一个闭环：driver 认领排队中的 prompt → 在会话日志上打开 turn → 通过 system-prompt 组装请求前缀、从日志派生历史 → 通过 LLM 接缝流式出模型响应 → 通过工具注册表派发工具调用 → 把每个模型可见事实追加回日志，供下一步派生。

| 包 | 拥有什么 | ctx key |
|---|---|---|
| `dsh-scope` | 作用域上下文注册原语（依赖无关库） | 无 key |
| `dsh-session` | 追加式的 `SessionEvent` 日志与内存存储——唯一真源 | `ctx.sessions` |
| `dsh-system-prompt` | prompt 段与工具 schema 组装注册表 | `ctx.systemPrompt` |
| `dsh-tools` | 作用域工具注册表 + 受守卫的执行流水线 | `ctx.tools` |
| `dsh-agent` | `Agent` 接口、活跃注册表、initiator 作用域、`agent/*` 事件词汇 | `ctx.agents` |
| `dsh-agent-default-model` | Agent 入口点共享的默认模型选择 | `ctx.agentDefaultModel` |
| `dsh-agent-loop` | 默认的具体 agent driver | `ctx.agentLoop` |

关键分工：`agent` 拥有公开契约，`agent-loop` 是它的默认实现。**扩展插件依赖 `agent`（包括需要 initiator Agent 时），从不直接依赖 `agent-loop`**，这样循环保持可替换。`scope` 是唯一非服务包，位于 `session` 与 `system-prompt` 之下，恰好让它们消费它而不成环。把这个主轴接成可运行 agent 的默认组合是 [`agent-spine-demo`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/examples/agent-spine-demo/README.md)。

## 4.2 Agent 的创建与所有权

消费者通过 `ctx.agents` 创建 agent——`create()` 在调用者提供的 `SessionId` 下新建会话与 agent，`resume()` 先加载持久会话——或者通过循环的配置项声明式创建。编程式创建返回 owner 的 handle：

```ts type-equiv
interface AgentHandle {
  agent: Agent
  dispose(): Promise<void>
}
```

`dispose()` 停循环、等它退出、注销 agent、把它的会话从存储移除，最后回退它的作用域世界。`AgentFactory` 是注册表背后的创建接口：循环通过 `ctx.agents.setFactory()` 注册自己的工厂，消费者用 `ctx.agents` 而不依赖具体循环包。

`CreateAgentOptions` 携带共享身份和一个全新 agent 发布前所需的一切（会话元数据、可选的 fork 种子前缀、每 agent 选项、仅创建期的取消 signal、`setup`）。`setup` 回调在**两个 id 都还未发布时**组装 agent 的作用域世界；`setup` 拒绝、commit 抛错或 owner 销毁都会回滚事务、不发布任何 id。

## 4.3 Agent handle：`Agent` 接口

`Agent` 是每个插件（UI、hooks、编排器）编程所面对的面。它有一个 `send` 方法直接暴露目标与唤醒路由，`followup`、`steer`、`inject` 是固定预设的别名。

```ts type-equiv
interface Agent {
  readonly id: SessionId
  readonly options: AgentOptions
  readonly session: Session          // 该 agent 驱动的活跃会话；其日志是持久真源
  readonly inbox: Inbox              // 持久待办工作的 agent 自有投影
  readonly status: AgentStatus
  readonly ctx: Context              // agent 作用域上下文；贡献是 agent 本地的、销毁时回退
  cancel(cause: AgentCancelCause, options?: CancelOptions): void
  whenIdle(): Promise<void>
  runMaintenance<T>(task: (signal: AbortSignal) => Promise<T>): Promise<T>
  send(message: UserMessage, target: InboxTarget, wakeup: boolean): void
  followup(message: UserMessage): void   // 排队一个普通后续 turn 并唤醒 driver
  steer(message: UserMessage): void      // 提交最近一步的 steering
  inject(message: UserMessage): void     // 排队模型可见上下文，不唤醒 driver
}
```

三个别名语义对照：

- **`followup`**：排队一个普通后续 turn 并唤醒；该项成为它自己 turn 的唯一普通消息。
- **`steer`**：为最近一步提交 steering。空闲 driver 启动一个 turn；运行中的 driver 在下一步边界消费它。
- **`inject`**：为下一个 pre-step 排队模型可见上下文而不唤醒 driver。运行中的 driver 在最近的后一步边界认领；空闲 driver 保持 pending 直到 follow-up/steering 唤醒它。

`AgentStatus` 只有 `'idle' | 'running'`。`running` 描述 driver 级的排空间隔，可能横跨连续排队的 turn，并不证明某个 turn 仍开着。注销把 agent 移出注册表并发 `agent/disposed`，不是第三种状态。

## 4.4 Inbox：投递词汇

inbox 是 agent 拥有的两列有序待处理消息（持久投影）：`'next-turn'` 与 `'next-step'`。`claim(target)` 取出提议步骤的批次——所有 `next-step` 输入加上（在 turn 边界时）一条 `next-turn` 消息。`MessageId` 是唯一身份。

## 4.5 取消

```ts type-equiv
type AgentCancelCause =
  | { readonly kind: 'user' }
  | { readonly kind: 'parent' }
  | { readonly kind: 'hook'; readonly reason: string }
  | { readonly kind: 'disposed' }
```

原因是 TypeScript 强制的同进程输入；`keepInbox` 选项保留排队与 steering 项（活跃 turn 仍被中止）。持久 `turn/end` 只保留粗粒度的 `{ kind: 'aborted' }` 结果。

## 4.6 拦截决定

`agent/pre-step` 是请求派生前**唯一**的 serial 监听链。它返回 `PreStepDecision`：

```ts type-equiv
type PreStepDecision =
  | { kind: 'reject' }
  | { kind: 'enter'; messages: UserMessage[] }
```

Reject 不打开 step；Enter 提供在 `step/start` 后追加的完整消息批次。`agent/request-error` 在一个失败的模型 step 关闭后、其 turn 关闭前运行：处理监听器不调用 `next()` 而返回 `{ kind: 'retry' }`，默认 `undefined` 让失败成为终态。`agent/turn-stopping` 在一个 turn 没有工具或 steering 续体时、最后一次 steering 排空前运行。

`agent/session-start` 携带 `SessionStartSource`（`'startup' | 'resume' | 'clear' | 'compact'`）——为什么这个会话生命周期开始。

## 4.7 会话日志：`Session`

一个 `Session` 是类型化 `SessionEvent` 的**追加式日志**——唯一真源。LLM 消息历史从日志**派生**（`deriveMessages()`），不单独存储。每条 entry 携带单调 `seq`、`time`、`type` 判别的 `data` 载荷。

核心事件变体：`turn/start`、`turn/end`、`step/start`、`step/end`、`user/message`、`assistant/chunk`、`assistant/message`、`tool/call`、`tool/result`、`steering/message`、`todo/write`、`request/header`。`SessionEventMap` 是合并可扩展的——插件经 declaration merging 加自己的变体（如 `goal/change`、`compaction/start`、`llm/retry`）。完整投影规则与 `TurnTrigger`/`TurnEndReason` 见[会话子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)。

## 4.8 `ToolDefinition`：工具的唯一契约

每个注册工具是什么——一个模型面 `ToolSchema` 加一个 `execute` 函数，外加可选的 final-content 与 UI 回调。工具作者很少手写它（`defineTool` DSL 用类型化参数构建），但它是注册表持有、循环派发的契约。完整字段见[工具子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/tools.md)，执行流水线见[工具执行流水线](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/tool-execution-pipeline.md)。

## 4.9 两条全仓库类型模式

`dsh` 每个子系统都复现的两条模式，这里讲一次：

**`…Map → derived-union`**：几乎所有可扩展求和类型都由判别 tag 键控的接口派生 union，插件用 declaration merging 加变体，无需改拥有包：

```ts ignore-check
interface ThingMap {
  'a': { kind: 'a' }
  'b': { kind: 'b' }
}
type ThingKind = keyof ThingMap          // 'a' | 'b'
type Thing = ThingMap[keyof ThingMap]    // 判别联合

// 插件扩展它，不碰源包：
declare module '@deepseek-ai/dsh-llm' {
  interface ThingMap { 'c': { kind: 'c' } }
}
```

六张规范 map：`ContentBlockMap`、`MessageSourceMap`、`FinishReasonMap`（dsh-llm）、`TurnTriggerMap`、`TurnEndReasonMap`、`SessionEventMap`（dsh-session）。消费者最常 `switch` 的两个大判别联合是 `StreamChunk`（流协议）与 `SessionEvent`（日志条目）。

**Branded ID**：跨包传递的 id 是 branded——结构上是字符串，但类型上不可互换（`SessionId` 不能传进期待 `CallId` 的位置）。`Branded<B>` 原语在类型专用的 [`dsh-brand`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/util/brand/README.md)（无运行时代码、无 harness 包依赖）。

## 下一步

循环把会话词汇（`Message`、`ContentBlock`、`StreamChunk`、模型请求）搬来搬去——这些类型由 `packages/llm` 声明。进入[第 5 章：LLM 能力](05-llm.md)看模型适配器接缝。
