# 第 11 章：启动、组合与预设

这一章回答「一个 `dsh` 进程是如何拼出来的」：boot 胶水、bundle 补丁层、每会话的 agent preset，以及让 agent 检查并挂载自己插件的自修改工具集。

## 11.1 boot（启动胶水）

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-app-boot` | 共享启动胶水：`.env` 加载、fail-loud Loader 守卫、快照感知的配置解析、settle-the-tree 启动序列 | （bin 的库） |
| `dsh-cmdline` | 启动器到 app 的命令行交接与 app 自有的启动参数解析 | `cmdlineArgs`、`appExit` |

`apps/cli` 与 `examples/` 演示 bin 共享这套与通道无关的启动库。启动序列与 personal-config 契约见 [`app-boot/README.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/boot/app-boot/README.md)。

## 11.2 bundle（profile 插件捆绑包）

bundle 是 npm 包，其 manifest 声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`，使它们成为 `dsh --profile` 组合可安装的补丁层。bundle 的实质是它的补丁列表；有些还附带补丁所挂载的运行时胶水插件。

| 包 | 角色 |
|---|---|
| `dsh-base` | 每个 profile 最先应用的共享 dsh 核心（仅补丁） |
| `dsh-web-app` | 浏览器面：web 补丁层 + 运行时胶水插件 |
| `dsh-headless` | base 之上的直接单次任务模式，无 Host 或 Web 层（挂载 `headless-runner`） |

内置 bundle 从 dsh 安装解析；树外 bundle 经 `dsh plugin --profile <name> add <package>` 装进 profile。补丁按第 2 章的分层顺序应用到空 entry 列表。

## 11.3 preset（每会话 agent 组合）

一个 **agent preset** 是持有一份 `agent.cordis.yml` 的目录。把它挂到某个 agent 的作用域上下文下，就给了该会话自己的工具与 prompt 段，而其它活跃会话各持各的——于是一个进程能同时跑几个不同组合的 agent。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-agent-presets` | preset 词汇、可信/用户作者根上的文件系统发现、受守卫的每 agent 挂载 | `ctx.agentPresets` |
| `dsh-persona` | 作为可组合行的 agent persona（preset 可改身份，不只是工具） | — |

组合分工：注册表与跨会话设施是进程单例，留在 host 组合里；preset 携带「一个 agent 对它们贡献了什么」。一个发布进程全局服务的 preset 行会在挂载时被**拒绝**，而不是与下一个会话撞名。

`ctx.agentPresets` 的关键方法（详见[生成参考](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/core.md#ctxagentpresets--agentpresets)）：`list()`/`resolve()` 每次调用重读根目录（作者创作即时可见）；`mount(agentCtx, id)` 在 `setup` 里调用，失败回滚 agent 创建；`composeFrom(agentCtx, parentCtx)` 让子 agent **绑定**父的同一份 standing 组合（同 plugin 对象、同工具注册、同 prompt 段）——这是两个进程内 subagent driver 在同步 `setup` 里组合子 agent 的机制；`recompose()` 重链接到另一 preset（仅在该 agent 尚未产出任何内容时有效）。

部署随附的 preset 在 [`apps/cli/config/agent-presets/`](https://github.com/deepseek-ai/deepseek-harness/tree/master/apps/cli/config/agent-presets)（每个目录一份，目录列表即花名册）。

## 11.4 extensions（agent 修改自己的运行时）

模型面工具直接作用在 agent 自己所在的活 Cordis 运行时上：检查已加载插件与服务 API、定义并运行模型编写的动态包、再撤回它们。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-tool-cordis` | 模型面运行时检查与动态包工具 | （注册 `ctx.tools`） |
| `dsh-cordis-host-runner` | 定义注册表、host 半的 `node:vm` 沙箱、request-run 往返 | `ctx.dynamicCordisRunner` |
| `dsh-cordis-client-runner` | 双半包（dual-half）的浏览器半：把定义求值成活浏览器插件、回答 run 请求 | （client face） |
| `dsh-ui-cordis` | 浏览器面：操作每个定义的 frame 面板、只读 define 卡片 | （注册 slots） |

动态包事件（以源码为准）：`cordis/dynamic-package`、`cordis/dynamic-retract`、`cordis/request-run`、`cordis/request-run-resolved`。`ctx.cordisInspect` 注册 host 检查 provider、镜像 client provider 清单、经动态 Cordis 传输路由 client 查询。设计见[自引用 Cordis 工具集 Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-07-08-self-referential-cordis-toolset.md)。

## 下一步

组合讲完，最后一层是「对外接口」：怎么通过 API 网关、SDK、ACP、hooks 把 `dsh` 接进别的系统，以及 Web GUI 的 host/client 两半如何划分。进入[第 12 章：接入与对外接口](12-integration.md)。
