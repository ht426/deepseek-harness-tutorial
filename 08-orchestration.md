# 第 8 章：编排

这一章讲「一个 agent 如何组织更大规模的工作」：把活委派给子 agent、用脚本扇出、用 todo/plan/goal 维护持久协作状态、用 jobs 管理后台任务、用 compaction 压缩长上下文、用 guard 维护循环卫生。

## 8.1 subagent（子 agent 委派）

subagent 能力族（[`packages/subagent/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/README.md)）让一个 agent 把工作交给子 agent。多个命名 provider 可在同一 ctx 共存，共用一套服务 API。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-subagent` | Service Definition：provider 注册、委托、结果、可续子 agent 编排 | `ctx.subagents` |
| `dsh-subagent-in-process-driver` | spawn/fork 共享的运行驱动 `startInProcessRun()` | （无 ctx key） |
| `dsh-subagent-spawn-in-process` | 进程内空对话新子 agent（provider `spawn`，四项能力全支持） | （注册 provider） |
| `dsh-subagent-fork-in-process` | 以父已完成回合前缀为种子（provider `fork`；种子仅传对话历史，不传权限） | （注册 provider） |
| `dsh-subagent-acp` | 子进程按 ACP 驱动（provider `acp`；无 start-time 能力、`inheritsParentContext: false`） | （注册 provider） |
| `dsh-subagent-claude-code` | 官方 `@anthropic-ai/claude-agent-sdk` 驱动真实 Claude Code（provider `claude-code`） | （注册 provider） |
| `dsh-subagent-codex` | `codex app-server --stdio` 临时线程（provider `codex`） | （注册 provider） |
| `dsh-subagent-dsh-sdk` | 完整 Harness 运行时子进程，stdio JSON-RPC（provider `dsh-sdk`） | （注册 provider） |
| `dsh-tool-subagent` | 模型面委托工具（`subagent`；one-shot 或 continuable） | （注册 `ctx.tools`） |
| `dsh-tool-subagent-control` | `send_message`/`interrupt_agent`/`list_agents` | （注册 `ctx.tools`） |
| `dsh-tool-subagent-report` | 子作用域 `report` 工具（child→parent），经 `registerContinuableSetup` 注入 | （注入） |

关键概念：

- **命名 provider 注册表**：`registerProvider()` 命名注册（重复名 loud fail），transport 可换（spawn / fork / acp / codex / …），契约不变。
- **start-time 能力 vs 运行时能力分离**：`SubagentCapabilities`（`outputSchema`/`depthLimit`/`toolFilter`/`persona`）让服务在创建前拒绝不支持请求。
- **one-shot vs continuable**：one-shot（`start → result → dispose` 所有权转移，返回 `JobId`）对比 continuable（持久 Session + 进程本地 Activation，返回 `subagentId`，可 `Agent.followup()` 续写）。
- **委托深度**：`SessionHeader.delegationDepth` 单调权威。

类型与事件见[subagent 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/subagent.md)（`SubagentStartRequest`/`Result`/`Run`）。

## 8.2 workflow（工作流）与 Ralph

workflow 能力族（[`packages/workflow/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/workflow/README.md)）运行模型编写的编排脚本、扇出 subagent。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-workflow` | Service Definition：执行与生命周期事件 | `ctx.workflowEngine` |
| `dsh-workflow-worker-thread` | 每 run 一个 worker thread 的执行引擎（脚本 hooks `agent`/`parallel`/`pipeline`/`phase`/`log`） | （注册 `ctx.workflowEngine`） |
| `dsh-tool-workflow` | 模型面 `workflow` 工具 | （注册 `ctx.tools`） |
| `dsh-tool-ralph` | 固定 Ralph 工作流工具（`objective` + `maxRounds`；每轮全新子 agent） | （注册 `ctx.tools`） |

关键概念：

- **`meta` 是数据不是脚本**：`WorkflowMeta`（`name`/`description`/`whenToUse?`/`phases?`）先校验再跑。
- **错误语义**：`parallel()`/`pipeline()` 抛 fatal（`WorkflowError`）；普通子 agent 失败返回 `null`。
- **holder-owned run**：`result` 永不 reject，必须 `dispose()`。
- **worker 隔离事件循环，非安全边界**。
- **Ralph** 是「面向不可变目标的前台全新 agent 工作流」：每轮全新子会话，报告 `continue | complete | blocked`，共享工作区 + 有界交接承载跨轮状态（见[术语表 Ralph 条目](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/glossary.md#ralph-loop)）。

## 8.3 todo（任务清单）

| 包 | 职责 |
|---|---|
| `dsh-tool-todo` | 注册 `todo_write(todos:[{content,status}])` |

要点：**整表替换**、无局部更新；`status: pending|in_progress|completed`；每次调用追加 `todo/write` 会话事件（last-write-wins）；单一拥有者（一个 agent session）。`todo/write` 事件驱动 UI 与回放，不是第二模型消息。

## 8.4 plan（plan mode）

`dsh-plan-mode`（`ctx.planMode`）是每 agent 的**已记录协作状态**，不是模式注册表。

要点：

- 事件 `plan/mode`（`{ active }`，log-only、整值替换，`foldPlanMode` 从日志恢复）。
- 命令 `/plan [message]`、`/plan off`；工具 `exit_plan_mode`（需用户批准，`plan-review` 意图）。
- **soft guidance，不强制**：沙箱/审批独立执行。
- 状态变更走 step-boundary flush（running 时 pending 到下一个 pre-step）。

细节见[plan 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/plan.md)。

## 8.5 goal（同会话目标）

goal 能力族（[`packages/goal/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/goal/README.md)）维护同会话持久目标，与模型工具、续跑策略解耦；状态属会话日志。

| 包 | 职责 | ctx key |
|---|---|---|
| `dsh-goal` | Service Definition（`ctx.goals`）：`GoalId`/`GoalRef{id,revision}`（CAS 栅栏）、`GoalPhase`（`active|paused|blocked|complete`）、`GoalActivation`（`armed|disarmed`，进程本地）；事件 `goal/change`（durable，每次携带完整快照）+ scoped `goal/changed` | `ctx.goals` |
| `dsh-goal-round-driver` | 空闲时保留 `<goal_round>` 提示、按 `GoalMessageSource` 递增 `roundsStarted` | — |
| `dsh-tool-goal` | 模型面 `get_goal`/`create_goal`/`update_goal`（`edit|pause|resume|complete|blocked`） | （注册 `ctx.tools`） |
| `dsh-command-goal` | 人类 `/goal` 命令 | （注册 `ctx.commands`） |

关键概念：

- **事件源状态**：每次 `goal/change` 写完整快照，revision CAS 拒 stale。
- **activation 从不持久化**：resume / fork 后必须显式 resume 重新 armed。
- **回合驱动**：`roundsStarted` 只在 admitted goal-sourced `user/message` 上递增，人类消息不计。
- 一次一个 current goal；blocked 用 policy code + 说明，而非多个生命周期态。

## 8.6 schedule（会话提醒）

`dsh-schedule` 提供 `schedule_create`/`schedule_list`/`schedule_delete`（`after_seconds`/`at`/`every_seconds`）。要点：会话日志是唯一持久源，timer/工具值是可丢弃投影（fork 不继承父提醒）；到期经 Agent 普通 `followup()` 队列投递，不打断当前回合；操作先 `ctx.sessions.flush()`，持久化不确定返回 `persistence_uncertain`。

## 8.7 jobs（后台任务）

jobs 能力族（[`packages/jobs/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/jobs/README.md)）为长时工具提供统一的 owner 隔离后台任务协议。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-jobs` | 注册表契约（`JobRegistry`、`JobId`、`JobKindMap`、`JobStart`、`JobHooks`、`JobStatus`） | `ctx.jobs` |
| `dsh-jobs-local` | 进程内 `LocalJobRegistry`（id `<kind>-N`，`maxConcurrentJobsPerOwner` 默认 10） | （注册 `ctx.jobs`） |
| `dsh-tool-jobs` | 模型面 `job_output`/`job_list`/`job_kill` + 完成通知 | （注册 `ctx.tools`） |

关键概念：生产者/消费者模型——producer 提供 kind + `run()`/hooks；owner 隔离以 `SessionId` 为栅栏；流输出单游标；settlement first-wins，通知抑制（reported 位）防重复。细节见[jobs 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/jobs.md)。

## 8.8 compaction（压缩）

压缩能力缝（[`packages/compaction/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-compaction` | Service Definition：`CompactionEngine`（`compactIfNeeded`/`compactNow`/`compactRegion`） | `ctx.compaction` |
| `dsh-compaction-basic` | Provider：`ctx.tokenMeter` 压力 + `ctx.llm.stream()` 摘要 | （注册 `ctx.compaction`） |
| `dsh-compaction-tool-result-pruner` | 无模型裁剪（`ctx.toolResultPruner`），重写超预算 `tool/result` | `ctx.toolResultPruner` |
| `dsh-command-compact` | 人类 `/compact` 命令 | （注册 `ctx.commands`） |

关键概念：WHAT vs HOW 分离（Service Definition 只定语义）；surface 契约——仅 `user/message|assistant/message|tool/result` 可带 `surfaceOp`，压缩以 `surfaceOp:{op:'replace'}` 单点突变 + 锁为 durable bracket（`compaction/start`…`compaction/end`）；`compactionId` 关联 checkpoint 与事务。细节见[压缩子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/compaction.md)。

## 8.9 guard（循环卫生）

| 包 | 职责 |
|---|---|
| `dsh-repeat-tool-reminder` | 建议性提醒：按 `(tool, canonicalized args)` 统计连续重复，阈值 `[3,5,8]`；经 `tools/post-execute` 的 `additionalContexts` 注入，不 veto、不重写 |
| `dsh-tool-call-timeout-policy` | 零配置 `tools/execute` 包装器，按 `ToolDefinition.timeoutMs` 布防 deadline，到期返回结构化 `TOOL_TIMEOUT` |

要点：守卫经事件监听而非改 `agent-loop`；超时是协作式（signal 通知，工具须转发 signal）。

## 下一步

这一章讲的是「agent 自己怎么组织工作」。下一章[第 9 章](09-interaction.md)转向「agent 怎么和人、和上下文协作」——context、commands、approval、permission-presets、ask-user、feedback。
