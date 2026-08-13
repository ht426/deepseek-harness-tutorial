# 第 5 章：LLM 能力

这一章讲模型适配层（[`packages/llm/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/llm/README.md)）：会话与流式词汇、`LlmAdapter` 接缝、DeepSeek 与 pi-ai 提供方适配器、重试策略与 token 计量。

## 5.1 五个包

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-llm` | LLM 服务与共享流式词汇（同时承担 Service Definition 与 Consumer 角色） | `ctx.llm` |
| `dsh-token-meter` | 可回放的 token 计量 | `ctx.tokenMeter` |
| `dsh-llm-retry` | provider 作用域的重试策略 | （监听 `agent/request-error`） |
| `dsh-llm-deepseek` | 直连 DeepSeek 适配器 | （注册在 `ctx.llm`） |
| `dsh-llm-pi-ai` | 多 provider 的 pi-ai 适配器 | （注册在 `ctx.llm`） |

适配器在接缝上注册 provider 路由；重试与 token 计量是独立的消费者。细节见[LLM 流式子模块](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/llm-streaming.md)。

## 5.2 会话词汇：`Message` 与 `ContentBlock`

一段对话是 `Message` 的序列；一条消息是类型化 **content block** 的数组。block union 从 `ContentBlockMap` 派生：

```ts type-equiv
interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}
```

五种块：`TextBlock`（可见文本）、`ReasoningBlock`（思考，区别于可见文本）、`ImageBlock`（持久图片附件）、`ToolCallBlock`（`id: CallId`、`name`、raw-JSON `arguments`）、`ToolResultBlock`（`toolCallId`、嵌套 `content`、`isError?`）。

```ts type-equiv
interface Message {
  readonly id: MessageId
  readonly role: 'system' | 'user' | 'assistant'
  readonly content: ContentBlock[]
  readonly source: MessageSource
}
```

消息来源是 merge-extensible 求和类型（`MessageSourceMap`：`user` / `plugin` / `model` / `tool`）。生产者的**身份**（`kind`：谁产生的）与**呈现形式**（`form`：什么信息）是两条独立轴；`form` 的值是语义化的——`instructions` / `catalog` / `snapshot` / `notice` / `relay` / `recall`，颜色、图标、排序等视觉决定归消费者，不进这个 union。

## 5.3 原始流协议：`StreamChunk`

流式响应交错着几种类型化块（文本、思考、多个工具调用）。`index` 把每个 delta 绑到它的块；`block-end` 携带**已组装完成**的 `ContentBlock`，消费者不必自己重装 delta。它是一个**封闭**判别联合——`switch` 以 `assertNever` 收尾，新增变体会在必须处理它的每个消费者处编译失败。

```ts type-equiv
type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: CallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | { type: 'finish'; reason: FinishReason; replayState?: unknown }
```

其中 `FinishReason` 从 merge-extensible 的 `FinishReasonMap` 派生（`stop`/`tool-calls`/`max-tokens`/`aborted`/`error`，后两者携带 `failure: LlmFailure`）。约定：适配器在终态 `finish` **之前**发 `usage`，其后不再发任何东西；工具参数保持 raw JSON 字符串。适配器实现可以抛错，但 `LlmRuntime.stream()` 把失败规范化为携带 `kind:'error'|'aborted'` 的 `finish` 再暴露给消费者。

## 5.4 适配器契约：`LlmAdapter`

`LlmAdapter` 是 provider 契约：子类化它、实现 `stream()`，用 `ctx.llm.registerAdapter(providers, adapter)` 注册一个适配器实例。`GenerateOptions.provider` 选择已注册的适配器；`GenerateOptions.model` 传给该适配器，无需在生命周期启动时注册。重复的 provider 路由原子失败。

```ts type-equiv
declare abstract class LlmAdapter {
  abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>
}
```

两条受认可的失败路径，一个 `LlmFailure` 类型：

- 从 `stream()` **抛**（传输/协议错误）；
- 或以 `finish {kind:'error'|'aborted', failure}` 结束流（provider 带内错误）。

**一次适配器调用是一次 provider 尝试**：适配器禁用库级重试；agent 级恢复另开一个持久编号的 turn；直接 `ctx.llm.stream()` 调用者保持单次尝试。

**replay state 是适配器自有的**：成功的 `finish` 可携带重建原生 provider 响应所需的 lossless-JSON 状态；循环把它和组装好的 assistant 消息一起存储。之后请求时，仅当历史 provider 与目标 provider 当前都注册到**同一个**适配器实例，`LlmRuntime` 才把该状态传给该适配器；其它适配器收到 provider 中性内容 + provider/model 字段，而没有私有状态。

## 5.5 路由与重试

Provider 配置在路由注册前解析成不可变判别联合：`normal` 模式带有限 `maxRetries`、`retryableCodes` 与必需的退避字段；`always` 模式带同样的退避字段但没有有限上限。`llm-retry` 监听 `agent/request-error`，追加 `llm/retry` / `llm/retry-started` 会话事件。

`agent/request` 与 `llm/stream` 是两个 waterfall：前者替换冻结的调用配置（`LlmCallConfig`），后者包裹每一次流式模型调用（重试、回放、路由）——调用 `next()` 到达已解析适配器的 stream，或产出自己的 chunk 来短路。

## 5.6 token 计量

`dsh-token-meter`（`ctx.tokenMeter`）拥有按会话隔离的**回放折叠**：不可变的标量测量与位置化回放测量，配以已消费日志修订号。压缩消费者（`compaction-basic`）共享这些不可变修订测量，判定上下文压力。

## 5.7 适配器如何注册路由

以 DeepSeek 为例，适配器注册 `deepseek-official` 路由到 `ctx.llm`。换模型提供方 = 写一个 `LlmAdapter` 子类、实现 `stream()`、`registerAdapter(providers, adapter)`——其余产品（agent 循环、UI、工具）都通过 `ctx.llm` 的 provider 中性接口工作，不感知具体适配器。

## 下一步

模型适配器负责「生成」，而「执行」由能力接缝负责。进入[第 6 章：执行类能力](06-execution-capabilities.md)——shell、subprocess、terminal、code-runtime、sandbox 如何共享一个执行世界。
