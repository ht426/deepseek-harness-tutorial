# 第 12 章：接入与对外接口

这一章讲 `dsh` 怎么接进别的系统：Typert/API 的远程调用网关、进程外 SDK、ACP 自动化协议、Claude Code/Codex hook 桥，以及 Web GUI 的 host/client 两半。

## 12.1 Typert 与 API 网关

**Typert**（[`packages/typert/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/typert/README.md)）把「源码分析、运行时存储、Loader 发现」三件事分离：

| 包 | 角色 | Cordis key |
|---|---|---|
| `dsh-typert-registry` | 存运行时包反射与 schema | `ctx.typert` |
| `dsh-typert-loader` | 发现 Loader entry 并注册生成的 host 工件 | 消费 `ctx.loader` 与 `ctx.typert` |
| `dsh-typert-generator` | 从源类型生成运行时工件 | 构建期库 |

**API 远程栈**（[`packages/api/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/api/README.md)）：

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-api-remotes` | Host Agent/Session 查找策略与 Client Remote 贡献组装 | 配置 `ctx.typert`、消费 `ctx.remote` |
| `dsh-api-gateway` | Host Typert 派发器与 Client Remote 端点 | `ctx.typertGateway` / `ctx.remote` |

业务服务在 Host 上用 `@Remote` / `@RemoteScope` 声明可调用方法；Host 构建生成 Host-for-Client 类型与运行时贡献，Client 的 `api-remotes` 把这些贡献加载到 `ctx.remote` 与作用域 `agentCtx.remote` 命名空间。运行时依赖方向是 `remotes → gateway → connection → webserver`。细节见 [API Gateway](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/api-gateway.md)。

## 12.2 SDK（进程外驱动）

[`packages/sdk/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sdk/README.md) 的协议栈让你从**另一个进程**驱动 Harness 运行时。调用方提供运行时可执行文件及其 `cordis.yml`；该组不创建/配置/构建/启动开发者项目。

| 包 | 角色 |
|---|---|
| `dsh-sdk-protocol` | 定义 SDK 运行时 wire 协议 |
| `dsh-sdk-client` | 经 TypeScript 客户端 API 驱动 Harness 运行时 |
| `dsh-sdk-server` | 在 stdio JSON-RPC 上服务进程外 SDK 客户端 |

## 12.3 ACP（仅自动化的 Agent Client Protocol）

[`packages/acp/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/acp/README.md) 经 Agent Client Protocol 把 harness agent 暴露给程序化客户端。它是互操作传输，不是呈现或人机交互层；匹配的进程外 subagent *客户端* 在 [`subagent-acp`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subagent/subagent-acp/README.md)（因为它实现的是 subagent provider 接口）。

ACP 服务器在 JSON-RPC stdio 上暴露全新 agent 会话；`pnpm run demo:acp` 演示它。

## 12.4 hooks（Claude Code / Codex 桥）

[`packages/hooks/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/hooks/README.md) 让用户用 Claude Code / Codex 的方式在生命周期点扩展 agent——把桥插件指向已有的 `hooks.json`（或设置），让外部 shell hook 忠实运行。

| 包 | 角色 |
|---|---|
| `dsh-hook-protocol` | 共享 shell-hook 协议库 |
| `dsh-hooks-claude-code` | Claude Code hook 桥 |
| `dsh-hooks-codex` | Codex hook 桥 |

规范扩展面本身是 harness 的类型化拦截点（`agent/pre-step`、`tools/pre-execute` 等）；「原生 hook」就是这些扩展点上的普通 Cordis 插件。这些包是**桥**——把外部 shell-hook 协议翻译到同一扩展面上。

## 12.5 Web GUI：host 半与 client 半

Web GUI 分成两半。**Host 半**（[`packages/host/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/host/README.md)）是所有 client 形态共享的 API 网关与承载它的普通 HTTP 服务器：

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-apiproxy` | 共享 host API 网关与 wire 契约 | `ctx.apiProxy` |
| `dsh-webserver` | HTTP 路由载体 | `ctx.webServer` |
| `dsh-frontend-static` | webserver 回退座上的 SPA dist 服务器 | 消费 `ctx.webServer` |
| `dsh-directory-picker`（+ `-native`/`-browse`/`-auto`） | workspace 目录选择接缝 | `ctx.directoryPicker` |
| `dsh-plugin-inventory` | 当前 Loader entry 的只读投影 | Remote `pluginInventory/list` |

**Client 半**（[`packages/client/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/README.md)）是浏览器侧：shell 启动、浏览器-host 通信、共享 UI 服务、功能插件。骨架包是 `web`（从 client entry graph 启动浏览器 shell）、`modules`（加载浏览器侧 client 模块）、`web-react`（把 shell 运行时接到 React 渲染）、`connection`（浏览器-host RPC 通信与事件投递）、`runtime`（会话/workspace/UI 组合的共享 client 服务）、`hmr`（开发期刷新 client 插件）。其余 `ui-*` 包是功能插件，经 **slot 系统**（`ctx.slots.register({ name, children?, store?, inject? }, Component)`）组合。

组合出的应用是 `apps/cli` 启动 [`dsh-base` bundle](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/bundle/base/cordis.patch.yml) 去服务 `apps/web`。`apps/web` 的 Vite 入口构建浏览器 shell，**不是独立应用**——只有 `dsh web` 注入 `window.__DSH_BOOT__` 它才能启动。细节见[client-modules 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/client-modules.md)。

## 下一步

到这里，你已经走完了 `dsh` 的全貌。最后一章[第 13 章：如何扩展 dsh](13-extending.md)把这些知识变成可执行的步骤——加工具、加提供方、加包、加 LLM 适配器、加 Chat 节点。
