# 第 13 章：如何扩展 dsh

这一章把前面各章的知识变成可执行步骤。完整的分步指南在 [`docs/cookbook/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/extension-cookbook.md)；这里给出「该选哪个扩展点、最小代码长什么样、下一步读哪个 cookbook」。

## 13.1 先选对扩展点

第 1 章的「新行为该放哪」表是起点。核心原则：**新行为挂到文档化的扩展点上，改动 agent 循环本身才需要更新架构图**。常用映射（详见[扩展 cookbook 的特性→机制表](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/extension-cookbook.md#the-feature--mechanism-map)）：

- 加模型提供方 → `LlmAdapter` 子类 + `registerAdapter`。
- 加模型能力 → `ctx.tools.register()`；schema 自动进入 prompt 组装。
- 拦截工具/请求/turn → `tools/pre-execute`、`agent/pre-step`、`agent/request`、`agent/turn-stopping`。
- 加人类命令 → `ctx.commands`。
- 加后台工作 → `ctx.jobs` + `job_*` 工具。
- 加模型可见上下文 → `agent.inject()`。
- 加 UI/编辑器集成 → 驱动 `ctx.agents`、从 `session/event` 渲染。

## 13.2 加一个工具

工具注册在 `ctx.tools` 上。`defineTool` 是类型化 helper（raw JSON-Schema `ToolDefinition` 也能直接 `register`，MCP 工具就是这么进来的）。最小形态：

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // 模型看到什么
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // 缺省即 optional
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args 从 schema 类型化：{ path: string; limit?: number }
      // exec 携带不可变身份 + token；signal 是可操作字段
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

`execute()` 契约的关键规则（详见[工具编写参考](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-tool.md)）：

- **args 已为你校验**：`defineTool` 在 `execute` 前按 `ParameterSchemaSpec` 校验模型生成的 `arguments`。DSL 表达不了的约束（非空、正数、跨字段）仍需你手工检查。
- **执行身份受保护**：注册表把 `arguments` 物化为分离的 lossless JSON、在策略开始前冻结、分配不透明 `exec.token`；`callId`/`name`/`arguments`/`agent`/`token`/`signal` 在派发中不可变。
- **声明并返回一个规范 JSON 值**：`output.schema` 用 `ValueSchemaSpec`；`execute` 只返回推断值，注册表快照、校验、冻结后交给 `output.render`。
- **抛错或返回无效值 = `isError`**：基础设施故障抛错；非零退出码这类「正常领域结果」放进规范值。
- **遵守 `exec.signal`**：触发时取消在途工作。
- **异步通知用 `exec.agent`**：`agent.inject({ content, source: { kind: 'plugin', plugin: '<name>' } })` 追加持久上下文供下一个模型请求看到——它不是唤醒。

**长时工作**：用 producer 配置门控 `run_in_background`，经 `ctx.jobs.start({ kind, label, owner: exec.agent, run })` 注册；成功返回 `{ kind: 'background', jobId }` 这类规范 handle。

**Code Mode 免费拿到你的工具**：每个可见工具都变成 `await tools.<name>(args)`，无需额外集成。

**UI 卡片**：`output.render` 是模型面内容；UI 卡片由 `presentCall`/`presentResult` 决定（`generic`/`terminal`/`diff`/`search`/`web` 意图）。它们是 `args` 的纯函数，必须能在回放时重跑。

## 13.3 加一个 hook / 策略插件

hook 插件是拦截点上的普通 Cordis 插件。权限门例子——从 `tools/pre-execute` 门返回类型化决定：

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision, ToolExecution } from '@deepseek-ai/dsh-tools'

declare function isAllowed(exec: ToolExecution): Promise<boolean>

export const name = 'permission-gate'

export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (!(await isAllowed(exec))) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()
  })
}
```

扩展点选择：`tools/pre-execute` 做可扩展 allow/deny/ask；`ctx.tools.guard()` 做后面监听器无法撤销的单调终态 deny；`tools/execute` 包裹实际派发生命周期（超时/重试/指标）；`tools/post-execute` 做显式结果改写；`tools/result` 观察不可变终态。

## 13.4 加一个 UI 插件或 Chat 节点

UI 插件从 `session/event` 流渲染（assistant token 流是 `assistant/chunk`，加上 turn/step 边界与工具活动），经 `agent.followup()` / `agent.steer()` 把输入送回。浏览器插件给内置 Web Client 贡献业务行时，注册一个 `ConversationNodeDefinition` + keyed Chat 渲染器（见 [Conversation Node 指南](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-conversation-node.md)）。

## 13.5 加一个协议驱动

协议驱动把 wire peer 适配到 `ctx.agents`。stdio 驱动拥有 stdout、经工厂创建/resume agent、把协议请求映射到 `followup()`/`cancel()`。拆 agent 用 `AgentHandle.dispose()`（停 + 等退出，到达 quiescence）。[`packages/acp/acp`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/acp/acp/README.md) 是「仅自动化」的完整范例。

## 13.6 加一个 LLM 适配器

子类化 `LlmAdapter`、实现 `stream()`、`ctx.llm.registerAdapter(providers, adapter)` 注册 provider 路由。`dsh-llm-deepseek`、`dsh-llm-pi-ai` 是随附范例。完整步骤见 [LLM 适配器指南](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-an-llm-adapter.md)。

## 13.7 加一个 provider（能力接缝）

给既有接缝加 provider，就是继承 Service Definition 的抽象服务、实现其方法、注册到对应的 `ctx.<key>`。以 shell 为例：继承 `ShellExecutor`（`run`/`start`/`sandboxMode`）并注册到 `ctx.shell`——`bash-local`、`bash-sandbox`、`pwsh-local` 都是这么做。要新建一个**完整**接缝，则需同时设计三角色：Service Definition + 至少一个 Provider + 至少一个 Consumer。

## 13.8 加一个新包

新包遵循统一的命名与布局规则，详见[加包 checklist](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-package.md)。要点：包名 `@deepseek-ai/dsh-<name>`；`src/types.ts` 只放类型；测试在包级 `tests/`；每包拥有 `./invariant`（注册 manifest 名，检查事件/数据关系或给 `No runtime invariant:` 理由）；README 记录模型/token/KV-cache 效果；注册进恰好一个 tsconfig 聚合。

## 下一步

写完代码只是开始——`dsh` 有严格的测试与文档纪律。最后一章[第 14 章：开发、测试与文档](14-development.md)讲清工作流、测试策略与文档规范，以及该跑哪些检查。
