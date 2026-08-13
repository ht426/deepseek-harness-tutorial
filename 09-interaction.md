# 第 9 章：上下文与人类协作

这一章讲两个相邻面：**请求上下文**（给模型可见上下文但不定义工具）与**人类协作平面**（人类与运行中的 agent 协作——问题、审批、权限预设、命令、反馈）。

## 9.1 请求上下文（context）

[`packages/context/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/context/README.md) 的产品插件给模型可见请求上下文，但不定义工具。

| 包 | 职责 | ctx key |
|---|---|---|
| `dsh-session-reference` | 其它会话的有界快照 | `ctx.sessionReferenceResolver` |
| `dsh-time-context` | 当前时间与已用时间上下文 | — |
| `dsh-tmux-context` | tmux 位置上下文 | — |
| `dsh-agent-instructions` | workspace 指令上下文 | — |

`agent-instructions` 被默认 `dsh-agent-spine-demo` bundle 包含（可通过 bundle 配置禁用）；`time-context`、`tmux-context`、`session-reference` 是 opt-in。

共同机制：上下文插件**统一经 `agent/pre-step` 注入带 source 的 `user/message`**——它们是模型可见的，因此也遵循「模型可见 ⟺ 已入日志」不变式，必须能从会话日志重建。这正是第 1 章「加模型可见上下文 → 调用 `agent.inject()`」的落地。

## 9.2 人类协作平面（interaction）

[`packages/interaction/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/interaction/README.md) 的服务与插件让人与运行中的 agent 协作。它们通过既有 agent/session 契约集成，而不是改循环。

| 包 | 职责 | ctx key |
|---|---|---|
| `dsh-commands` | 为交互式适配器注册并派发人类命令 | `ctx.commands` |
| `dsh-user-approval` | 协调一次性审批决定 | `ctx.approval` |
| `dsh-permission-presets` | 呈现并持久化面向用户的权限预设 | `ctx.permissionPresets` |
| `dsh-user-questions` | provider 无关的人类问答接缝 | `ctx.userQuestions` |
| `dsh-tool-ask-user` | 向模型暴露人类问题 | （注册 `ctx.tools`） |

交互式应用提供具体的命令、审批、问答适配器；自动化走 [`acp/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/acp/README.md)。

## 9.3 审批（approval）

`ctx.approval` 是一次性权限决定：经 `approval/request` waterfall 派发。**回答者是监听器**（ACP 桥为自己 agent 作答）；缺席时 fail closed 到 `unavailable`——这正是「无人能回答 = 拒绝」的安全默认。

审批在工具执行流水线里的位置：`tools/pre-execute` 之后、单调守卫之前；`ctx.approval` 在守卫前解析 ask。细节见[审批子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/approval.md)与[工具执行流水线](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/tool-execution-pipeline.md)。

## 9.4 权限预设（permission-presets）

`ctx.permissionPresets` 是面向用户的预设表（`workspace-write` / `danger-full-access`），把沙箱模式与审批策略两个旋钮捆绑。一个开关写一条 `permission/preset` 事件，贯穿到两个旋钮事件。

> 源码核对：该事件是 `permission/preset`（部分 README 旧写为 `permissionPresets/preset`，以源码为准）。

## 9.5 人类命令（commands）

**人类命令**是以斜杠开头的指令，由面向人类的适配器通过 `ctx.commands` 解释执行，**不成为模型消息**。它既区别于模型面工具，也区别于经 `ctx.shell` 的 shell 命令执行。命令输出是 UI 状态，除非处理器另行变更持久领域。典型例子：`/goal`（`dsh-command-goal`）、`/compact`（`dsh-command-compact`）、`/feedback`（`dsh-command-feedback`）、`/plan`（`dsh-plan-mode`）。

## 9.6 问答（user-questions / ask-user）

`ctx.userQuestions` 是 UI 支撑的人类问答接缝（`AskUserQuestionRequest`、answer/options 词汇、provider API、错误分类）。`tool-ask-user` 在 provider 无关的 `ask()` promise 上**暂停一个工具调用**——模型用 `ask_user_question` 工具向人提问，答案回来后才继续。细节见[问答子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/user-questions.md)。

## 9.7 反馈（feedback）

[`packages/feedback/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/feedback/README.md) 暴露两个刻意分离的契约，**都不进入模型对话**：

| 包 | 职责 | ctx key |
|---|---|---|
| `dsh-command-feedback` | 触发无关的 `feedback/record` 事件 + 人类面 `/feedback` 生产者 | — |
| `dsh-message-feedback` | 生命周期绑定的每条消息评分/备注 sidecar + Host `messageFeedback.list/put/delete` Remote 契约 | `ctx.messageFeedback` |

命令反馈是 log-only（永不进模型上下文或派生历史）。消息反馈不是 Session 事件或投影，留在 storage-domain sidecar 里，不触发遥测交接。

## 下一步

协作状态是「活的」，但 `dsh` 的价值在于它把一切都落在持久日志上。下一章[第 10 章：持久化与数据平面](10-data-plane.md)看会话如何落盘、查询、投影，以及设置、凭据如何分层解析。
