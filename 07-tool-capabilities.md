# 第 7 章：工具与数据类能力

这一章覆盖「模型能直接调用、用来读世界、写世界、查资料」的能力族：文件系统、LSP、技能、Web、MCP、附件、溢出存储。它们大多是标准的能力接缝——Service Definition 拥有 `ctx.<key>`，Provider 注册实现，Consumer（通常是 `tool-*` 包）面向模型暴露工具名。

## 7.1 文件系统（fs）

文件系统栈（[`packages/fs/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/README.md)）由四类零件组成：provider 契约、本地实现、策略门插件、面向模型的文件工具 + 执行器，以及 ripgrep 驱动的发现工具。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-fs` | Service Definition：规范进程路径 / 文件 URI / 包含关系、文本 IO、原子变更原语；拥有 `fs/*` 策略事件 | `ctx.fs` |
| `dsh-fs-local` | 本地文件系统 `FileSystem` 实现 | （注册 `ctx.fs`） |
| `dsh-fs-e2b` | E2B 支撑的实现，共享 `ctx.e2b` 拥有的远程运行时 | （注册 `ctx.fs`） |
| `dsh-fs-sandbox` | 沙箱强制的实现：继承 `fs-local`，按每次调用的 mode + workspace root 策略围住 write/edit（read-only 拒绝、workspace-write 限制在会话 workspace + temp 根内），read 透传 | （注册 `ctx.fs`） |
| `dsh-fs-observation-policy` | 策略门插件：observed-state + read-before-edit + 带版本守卫的 write/edit，通过 `fs/*` 事件门参与 | （无服务——`fs/*` 监听器） |
| `dsh-tool-fs` | 模型面 `read`/`write`/`edit` 工具 **和** 执行器（读走 `ctx.fs`，拥有读窗口化，派发 `fs/*`） | （注册在 `ctx.tools`） |
| `dsh-tool-fs-search` | 模型面 `glob`/`grep` 发现工具，走打包的 `@vscode/ripgrep` 二进制（通过 `ctx.subprocess` spawn），**不是** `ctx.fs` provider 方法 | （注册在 `ctx.tools`） |

Service Definition 在 [`packages/fs/fs/src/index.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/fs/fs/src/index.ts)。沙箱化、远程或项目作用域的文件系统后端可以替换 `fs-local` 而不碰 Service Definition、策略门或模型面 schema：`fs-sandbox` 提供共享沙箱模式上的进程内路径围栏，`fs-e2b` 把文件状态放进与 E2B 子进程 provider 共享的远程执行世界。

策略（`fs-observation-policy`）是只通过 `fs/*` 事件门参与的插件，不是工具注入的服务——所以丢掉它只是优雅地失去策略、留下未约束的裸 provider，而不会弄坏工具。

**文件 IO 不带超时**：`read`/`write`/`edit` 没有 `timeoutMs`，provider 契约也不布防 deadline——因为 deadline 会杀掉操作系统仍会完成的工作。取消仍通过工具执行 signal 传播，在系统调用边界做尽力而为的中止。细节见[文件系统子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/filesystem.md)。

## 7.2 LSP（lsp）

语言服务器能力接缝：Service Definition、通用 stdio provider、面向模型的 `lsp` 工具。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-lsp` | Service Definition（按 branded id 的 provider 注册表 + 扩展映射、每次查询选择、词汇、`LspError`） | `ctx.lsp` |
| `dsh-lsp-stdio` | 基于 `ctx.fs` 与 `ctx.subprocess` 的通用多服务器 stdio 后端（JSON-RPC、瞬态打开查询） | （注册 provider 到 `ctx.lsp`） |
| `dsh-tool-lsp` | 模型面 `lsp` 工具（四种操作、从 1 开始的 UTF-16 游标坐标） | （注册在 `ctx.tools`） |

接缝暴露**恰好四种**语义操作——`goToDefinition`、`findReferences`、`goToImplementation`、`hover`——且**没有**通用 JSON-RPC 逃生门。因此换 provider 不改变模型请求导航的方式，也不会有协议载荷或未经审查的变更进入模型契约。Provider 注册的是**能力**，不是工具；`tool-lsp` 是模型面名字、schema、prompt 指引与呈现的唯一拥有者。

## 7.3 技能（skill）

技能族发现可复用的 agent 指令，并通过 provider 无关的目录与加载器暴露给模型。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-skill` | 定义 skill provider 注册与查找 | `ctx.skills` |
| `dsh-skill-badge` | 提供可选的打包 dsh badge 技能 | （注册在 `ctx.skills`） |
| `dsh-skill-filesystem` | 从本地文件系统发现技能 | （注册在 `ctx.skills`） |
| `dsh-tool-skill` | 发布技能目录与模型面加载器 | （注册在 `ctx.tools`） |

技能能力在核心控制主轴之外，换本地 / 内嵌 / 远程 provider 都不改变模型面契约。细节见[技能子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/skills.md)（发现优先级、目录快照、`skill` 加载器）。

## 7.4 Web（web）

Web 能力族提供 provider 无关的搜索与抓取，加上消费它们的模型面工具。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-web` | 定义 web provider 注册、选择与共享错误 | `ctx.web` |
| `dsh-web-search-exa` | 经 Exa 提供搜索 | （注册在 `ctx.web`） |
| `dsh-web-search-perplexity` | 经 Perplexity 提供搜索 | （注册在 `ctx.web`） |
| `dsh-web-search-deepseek` | 原生 DeepSeek 搜索 | （注册在 `ctx.web`） |
| `dsh-web-fetch-http` | 抓取公共 HTTP/HTTPS 资源 | （注册在 `ctx.web`） |
| `dsh-tool-web` | 向模型暴露搜索与抓取 | （注册在 `ctx.tools`） |

搜索与抓取共享同一个 provider 选择服务。细节见[web 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/web.md)（`WebSearchRequest`/`Result`、`WebFetchRequest`/`Result`、`WebError`）。

## 7.5 MCP（mcp）

| 包 | 角色 |
|---|---|
| `dsh-mcp-client` | MCP 客户端桥：把外部服务器工具注册到 `ctx.tools` |

它把 harness 桥接到 MCP 生态——外部 MCP 服务器的工具，以模型面工具的形式出现在 `ctx.tools` 上。

## 7.6 附件（attachment）

持久二进制附件接缝与本地文件系统实现。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-attachment` | 不可变附件引用、图片限制、存储服务 | `ctx.attachments` |
| `dsh-attachment-local` | `DSH_HOME` 下内容寻址的私有存储 | （注册在 `ctx.attachments`） |

未发送的浏览器草稿有意处在该能力之外：字节只在用户 prompt 提交、或 provider 适配器提交结构化模型输出时才进入持久存储。细节见[附件子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/attachment.md)。

## 7.7 溢出存储（spill）

溢出存储族把超大工具输出持久化，并用有界预览 + 检索定位器替换内联结果。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-spill` | 定义溢出存储 | `ctx.spillStore` |
| `dsh-spill-local` | 把溢出文本存进会话作用域的本地文件 | （注册在 `ctx.spillStore`） |
| `dsh-spill-policy` | 应用执行后溢出策略 | （监听在 `ctx.tools`） |

细节见[溢出子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/spill.md)（`SaveTextSpill`、owner/source、branded `SpillLocator`）。

## 7.8 这些接缝的共同点

回顾一下，你会发现它们共享同一条规律（正是第 1 章「能力接缝」的定义）：

1. Service Definition 声明接口与 `ctx.<key>`（抽象类或注册表）。
2. Provider 实现接口、注册到该 key；可以有多个 provider 并存。
3. Consumer 注入该 key 并使用；模型面工具是 Consumer 的典型形态。
4. 换 provider 不碰 Service Definition 与 Consumer——这就是「换一个 provider，整个产品跟着变」在数据/工具层的体现。

## 下一步

这些是「单次调用」的能力。下一章[第 8 章：编排](08-orchestration.md)看更高一层的组织能力：subagent 委派、工作流扇出、todo/plan/goal 的持久协作状态、jobs 后台任务与 compaction 压缩。
