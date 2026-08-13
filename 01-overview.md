# 第 1 章：总体架构

> 第一次接触 agent / 大模型？先花几分钟读[零基础导读](00-basics.md)，再回来更顺。

本章回答「`dsh` 到底是什么、由哪些部分拼成」，建立后续所有章节共同依赖的词汇和心智模型。读完本章，你应该能画出 `dsh` 的高层结构图，并说出「为什么改一个 provider 就能换掉整个产品的某块能力」。

## 1.1 一句话定义

DeepSeek Harness（`dsh`）是一个开源的 **agent harness（智能体框架）**：它负责把「大语言模型调用、工具调用、会话记录、权限、沙箱、人机协作」这些零件组装成一个可运行的 agent，并让每一个零件都可以从配置里替换。

它的架构原则只有一条：**一切皆插件（everything is a plugin）**。

## 1.2 一切皆插件

`dsh` 构建在 [Cordis](https://github.com/cordiverse/cordis) 之上（vendored 源码在 [`vendor/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/vendor/README.md)）。Cordis 是一个小型运行时：插件向一个共享的 **Context（上下文）** 贡献 **服务（service）**、**类型化事件（typed event）** 和**可逆 effect（effect）**。

关键推论：在 `dsh` 里没有需要打补丁的「特权核心」。模型适配器、工具注册表、会话日志、甚至 agent 循环本身，**都是插件**，因此都可以通过配置替换。你要扩展 `dsh`，就是在别的插件旁边挂载一个自己的插件；注册本身是可逆的 effect——插件卸载时会被撤销。

第 3 章会展开 Cordis 的全部机制；这里先记住这个结论，它决定了 `dsh` 的一切设计。

## 1.3 Profile 与 Bundle：运行中的 `dsh` 是一棵插件树

一个运行中的 `dsh`，是在启动时由若干**有序分层**拼出来的一棵插件树。

- **Profile（配置档案）**：存放在 Harness home 里的一个命名组合。它列出自己叠加哪些 bundle、安装了哪些树外插件，并保存用户自己的 `cordis.patch.yml`。`web` 和 `headless` 是随产品提供的两个模板。
- **Bundle（捆绑包）**：Cordis 配置行及其所挂载代码的**分发格式**。无论 bundle 插入了什么，都能被它上层的 layer 继续打补丁。

每个 bundle/profile 都在自己的 `package.json` 里通过 `dsh` 字段声明自己：`dsh.profile` 列出一个 profile 的 bundle，`dsh.bundle` 指向一个 bundle 的补丁文件。

三个内置 bundle：

- [`dsh-base`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/README.md)：每个 profile 的第一层——模型适配器、工具、持久化、沙箱与审批策略、设置、凭据、遥测。
- [`dsh-web-app`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/web-app/README.md)：加上浏览器应用（Web GUI）。
- [`dsh-headless`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/headless/README.md)：加上一个完全不启动服务器的单次运行器。

### 分层的应用顺序

各 layer 作用在一个**空的 entry 列表**上，顺序固定：

1. profile 里按列出顺序的每个 bundle；
2. profile 自己的 `cordis.patch.yml`；
3. home 级的补丁；
4. `--patch` 覆盖层。

一个补丁按 row id 定位某一行并整体替换它的配置，或者插入新行。想看你机器实际启动出来的树：

```sh
dsh --profile web --dump-config
```

它打印出的任意一行，都可以被你自己的一份补丁替换掉。组合机制细节见 [`app-boot`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md#profiles)；配置字段在生成的[配置目录](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/config-catalog.md)。

## 1.4 能力接缝（capability seam）

`dsh` 最核心的抽象是 **seam（接缝）**：一种**可替换能力**，由三种角色组成：

- **Service Definition**：声明接口的 Cordis `Service`，拥有自己的 `ctx.<key>` 和词汇类型（抽象类或具体注册表，绝不是 TypeScript `interface`）。
- **Service Provider**：实现该接口的一方。
- **Consumer**：注入并使用该服务的一方（常见形态是面向模型的工具）。

三种角色构成**完整**的能力，缺一不可。`packages/shell` 是规范范例：

| 角色 | 包 |
|---|---|
| Service Definition | `dsh-shell` |
| Service Provider | `dsh-bash-local`、`dsh-bash-sandbox`、`dsh-pwsh-local` |
| Consumer | `dsh-tool-bash`、`dsh-tool-pwsh` |

接缝是「换一个 provider，整个产品都跟着变」的原因：文件系统（`ctx.fs`）和子进程（`ctx.subprocess`）provider 共享同一个执行世界，把它们指向远程沙箱，Bash、PTY 终端、LSP 会一起跟着走，而无需为 provider 分叉出不同版本。Subagent 的 provider 同样在同一接口背后差别巨大——从全新子 agent 到「在另一个产品里委派一个 turn」。

第 6、7、8 章会逐个讲每个接缝；完整清单见[能力接缝图](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md)。

## 1.5 事件：扩展点

事件是 `dsh` 的扩展点，选对事件域是大多数改动的第一步。三类事件：

- **Session 事件**：追加到日志、并在 `session/event` 上广播的持久事实。当某个事实必须在重载后仍存在时使用。
- **Agent 事件**（`agent/*`）：携带一个活跃的 `Agent`（inbox、step、status、request、validation、continuation）。用来观察或拦截在途工作。
- **Capability 事件**：把策略和适配器挂到一个接缝上（`fs/*`、`tools/*`、`telemetry/*`），而不引入 agent 循环。

[事件地图](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/event-producer-consumer.md)列出了每个事件的生产者与消费者。

## 1.6 轮次（turn）流转

一个 **step（步骤）** 是「一次模型请求 + 它调用的工具」。一个 **turn（轮次）** 是零个或多个 step：它在第一条输入被认领前打开，在「没有任何待办」时关闭。流程图如下（`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 是持久会话事件；其余是跨三个域的活跃扩展点）：

```text
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step                   reject | enter(messages)
     reject, or a first enter rewritten empty -> close the turn with no step
     step/start
     append entered messages as user/message
     derive model history from the log
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
     tools owe another request, or next-step input arrived -> claim -> next step
  -> agent/turn-stopping
turn/end
```

几个关键点：

- `agent/pre-step`、`agent/request`、`llm/stream` 和三个 `tools/*` 事件是 **waterfall**（其监听器必须调用 `next()` 才能继续委托）；`agent/turn-stopping` 是 **serial** 且没有 `next()`。
- 输入通过**一个 inbox** 到达 driver。有些消息会立即唤醒它；注入的上下文则等在 inbox 里，直到另一条消息到来。
- `agent/pre-step` 决定模型看到什么。监听器可以改写被认领的消息，或直接拒绝；被拒绝（或首次认领被改写为空）的首次认领仍会关闭一个**不花任何 step** 的持久 turn——日志记录了这次尝试。

完整时序见[轮次/步骤时序图](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/agent-lifecycle.md)；工具细节见[工具执行流水线](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/tool-execution-pipeline.md)。

## 1.7 会话日志：模型所见即日志

会话日志是模型所见上下文的来源。`deriveMessages()` 从日志投影出模型历史；原始 `assistant/chunk` 事件保留了回放和 UI 保真度。fork、resume、transcript、遥测、持久化全都从这条流派生。

**模型可见 ⟺ 已入日志**：任何进入模型请求的内容，都必须能从日志重建，且有一条运行时不变式（invariant）断言这一点。这就是「新的模型可见输入需要新的 session 事件」的原因——扩展 `SessionEventMap` 并从日志渲染。

## 1.8 新行为该放哪

新行为挂到一个文档化的扩展点上；改动 agent 循环本身才需要更新架构图。常用映射：

| 目标 | 机制 |
|---|---|
| 加模型提供方 | 在 `ctx.llm` 上注册其适配器 |
| 加面向模型的能力 | 在 `ctx.tools` 上注册；其 schema 进入 prompt 组装 |
| 给某会话不同的能力集 | 组装一个 agent preset；其中的服务行需要 `isolate` realm |
| 加 shell 执行 | 注册一个 `ctx.shell` 后端；本地实现通过 `ctx.subprocess` spawn |
| 加持久终端执行 | 注册 `ctx.terminals` 后端 + `dsh-tool-terminal` |
| 加人类命令 | 在 `ctx.commands` 上注册；它不经过模型 turn 直接派发 |
| 加后台工作 | 在 `ctx.jobs` 上注册；`job_*` 工具收集或停止它 |
| 加文件访问或策略 | 注册 `ctx.fs` provider 或监听 `fs/*` 事件 |
| 约束子进程 | 使用 `ctx.sandbox` 后端；consumer 在 spawn 前包一层 argv |
| 拦截请求/工具/turn | 用 `agent/*` 或 `tools/*` 事件；`agent/turn-stopping` 停止 turn |
| 加模型可见上下文 | 调用 `agent.inject()`；落入下一个被接纳的请求 |
| 加 UI 或编辑器集成 | 驱动 `ctx.agents` 并从 `session/event` 渲染 |
| 加 Web Chat 节点 | 注册 `ConversationNodeDefinition` + 带 key 的渲染器 |
| 加持久会话状态 | 扩展 `SessionEventMap`；从日志渲染与回放 |
| 生成会话标题 | 注册唯一的 `ctx.sessionTitle` provider |
| 管理同会话目标 | 用 `ctx.goals`；通过 `agent/*` 续行 |
| fork 一个活跃会话 | `ctx.sessions.fork(source, boundary?, childSessionId?)` |
| 把注册限定到某个 agent | 用该 agent 的 `agent.ctx` |

第 13 章会把这些映射变成可执行的步骤。

## 1.9 仓库地图

顶层目录（详见根 [`AGENTS.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/AGENTS.md)）：

```
vendor/       vendored 的 Cordis 源码（manifest + 同步流程见 vendor/README.md）
packages/     @deepseek-ai/dsh-<pkg> 工作区，位于 packages/<group>/<pkg>/
  core/         产品 API 主轴：session、system-prompt、tools、agent、agent-loop
  api/          远程 BFF 组装与 Typert RPC 网关
  typert/       类型图生成器、加载器与运行时注册表
  llm/          LLM 能力：Service Definition/Consumer + DeepSeek 提供方
  e2b/          E2B POC：沙箱 + FS/subprocess 适配器
  shell/        bash 能力：Service Definition + local/pwsh 提供方 + shell Consumer
  subprocess/   子进程能力 + 本地进程树提供方
  terminal/     持久终端会话
  fs/           文件系统能力 + 策略
  lsp/          语言服务器能力
  skill/        技能提供方注册表 + 本地实现 + 目录/加载工具
  web/          web 能力：Service Definition + search/fetch 提供方 + 工具 Consumer
  compaction/   压缩能力 + 基础提供方
  context/      请求上下文插件
  subagent/     subagent 能力：Service Definition + 提供方 + 委派 Consumer
  bundle/       可安装的 dsh --profile 补丁层
  workflow/     工作流能力 + worker-thread 提供方 + 工具 Consumer
  todo/         todo_write 工具
  plan/         以日志状态呈现的 plan mode
  preset/       从 preset cordis.yml 组装每个会话的 agent
  guard/        循环卫生 + 工具超时插件
  session/      持久会话数据：持久化、投影、标题、遥测
  identity/     匿名身份
  settings/     用户设置能力 + 文件提供方
  credentials/  凭据引用能力 + env/.env 提供方
  acp/          仅自动化的 Agent Client Protocol 服务器
  interaction/  审批/交互能力、权限、命令、ask-user
  boot/         共享的 app-bin 启动胶水
  sdk/          JSON-RPC 协议、服务器与 TypeScript 客户端
  examples/     演示 bundle（agent-spine + CLI/ACP/JSON-RPC bins）
  support/      开发/测试基础设施
  util/         零依赖工具
python/       Python SDK 与捆绑运行时（见 python/README.md）
native/       @deepseek-ai/node-addon-landlock-run 的源（见 native/README.md）
examples/     可运行的 cordis.yml 叶子，叠加在 packages/examples 的 bundle 上
docs/         架构、生成的目录、postmortem、cookbook
website/      选定双语 docs 源的 VitePress 投影
```

每个组的「包 / `ctx` key 映射」由各组 README 维护（[`packages/README.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/README.md)）。

## 下一步

继续读[第 2 章：环境与运行](02-getting-started.md)，把 `dsh` 真正跑起来，看看 `--dump-config` 打印出的插件树长什么样；然后进入[第 3 章](03-cordis.md)补上 Cordis 的机制细节。
