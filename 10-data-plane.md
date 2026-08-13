# 第 10 章：持久化与数据平面

`core/session` 的活跃内存服务之外，是一整层持久数据平面：会话如何落盘、如何查询、如何投影出整日志派生的值，以及设置、凭据、非会话存储如何分层解析。全部是 product 包。

## 10.1 会话持久化

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-session-persistence` | 定义持久化服务与共享写协调 | `ctx.sessionPersistence` |
| `dsh-session-checkpoint-policy` | 应用语义持久化检查点 | （包裹 `ctx.llm` 与 `ctx.tools`） |
| `dsh-session-persistence-jsonl` | JSONL 文件持久化 | （注册 `ctx.sessionPersistence`） |
| `dsh-session-persistence-sqlite` | SQLite 持久化 | （注册 `ctx.sessionPersistence`） |

两个后端共享同一个 `PersistenceBackend` 契约与 `PersistenceCoordinator`，持久化的是**同一个 `SessionEvent` 词汇**；应用在组合时选后端。SQLite 用单调 `SCHEMA_VERSION`，`dsh-session` 的 `SESSION_FORMAT_VERSION` 保持 `0`、无兼容承诺（开发者预览立场）。细节见[持久化子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/persistence.md)（`session/flush`、崩溃恢复、`SessionHeader`）。

## 10.2 投影（projection）

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-session-projection` | 定义并驱动会话投影单元 | `ctx.sessionProjections` |
| `dsh-session-projection-cache` | 持久化并恢复投影检查点 | `ctx.sessionProjectionCache` |
| `dsh-session-stats` | 整日志对话计数与 wall time（`sessionStats` 单元） | （注册 `ctx.sessionProjections`） |

投影把「整日志派生的当前值」供给客户端载体，而不必每次重放整个日志：`session-projection-cache` 每会话持久化检查点，提供「缓存行 + 持久化尾回放」的冷读阶梯。领域注册状态驱动的 fold 单元；`tool-todo`、`session-title`、`host-apiproxy` 都消费它。细节见[投影子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session-projection.md)。

## 10.3 会话标题

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-session-title` | 拥有标题状态、回退行为、provider 注册与刷新 | `ctx.sessionTitle` |
| `dsh-session-title-llm` | 共享的模型支撑标题生成 | — |
| `dsh-session-title-first-prompt-llm` | 从第一条符合条件的人类消息标题化 | （注册 `ctx.sessionTitle`） |
| `dsh-session-title-all-prompts-llm` | 从所有符合条件的人类消息标题化 | （注册 `ctx.sessionTitle`） |

部署可注册**一个**模型支撑 provider；没有时服务保留确定性回退。细节见[标题子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session-title.md)。

## 10.4 会话遥测

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-session-telemetry` | 定义捕获、脱敏、投影、live/按需后端投递 | `ctx.sessionTelemetry` |
| `dsh-session-telemetry-otel` | 经 OpenTelemetry logs 投递（`FULL`/`FEEDBACK_ONLY`/`DISABLED`） | （注册 `ctx.sessionTelemetry`） |

接缝捕获、脱敏并把会话记录交给一个后端；没有别的东西消费该服务——其输出离开进程。细节见[遥测子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session-telemetry.md)。

## 10.5 会话查询与导出

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-session-query` | 定义可信读、关系查询与搜索操作 | `ctx.sessionQuery` |
| `dsh-session-query-sqlite` | SQLite 全文搜索实现 | （注册 `ctx.sessionQuery`） |
| `dsh-session-log-export` | Web `/export` 命令、浏览器下载状态、结果 modal | `ctx.sessionLogDownload` |
| `dsh-tool-session-query` | 向模型暴露 workspace 授权的会话查询 | （注册 `ctx.tools`） |

接口供给精确读、过滤器与 trace；具体后端加全文搜索、相关性排序、片段与游标。细节见[查询子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session-query.md)。

## 10.6 非会话存储

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-storage` | 连接已注册后端与类型化数据 form | `ctx.storage` |
| `dsh-storage-json` | JSON 文件存储 | （注册后端 `json`） |
| `dsh-storage-sqlite` | SQLite 存储 | （注册后端 `sqlite`） |
| `dsh-storage-domain` | 提供校验过的领域记录存储 | `ctx.storageDomain` |

消费者用**数据 form** 而非直接访问后端。`storage-domain` 等所有后端就绪后，把领域 form 发布为一个生命周期绑定的服务（供 `workspace`、`message-feedback` 消费）。细节见[存储子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/storage.md)。

## 10.7 workspace

`dsh-workspace`（`ctx.workspaceRegistry`）拥有持久 workspace：带标题的用户目录与有序会话成员关系。`WorkspaceId` 是 branded；会话 `cwd` 关系由它解析。细节见[workspace 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/workspace.md)。

## 10.8 设置（settings）

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-settings` | 定义命名空间注册、分层解析与提交 | `ctx.settings` |
| `dsh-settings-file` | 存本地文件并观察外部编辑 | （注册 `ctx.settings`） |

插件注册命名空间 schema、解析分层值；provider 存原始文档。解析顺序：默认 → 组合 `base` → 用户文档。细节见[设置子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/settings.md)（`SettingsNamespace`、owner scope、hot commits）。

## 10.9 凭据（credentials）

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-credentials` | 凭据引用接缝 | `ctx.credentials` |
| `dsh-credentials-local` | 环境 + 本地文件 provider | （注册 `ctx.credentials`） |

**配置携带引用，不携带秘密值**。消费者在各自的操作边界解析引用；一次轮换的凭据在下一次请求即生效。UI 拿到 value-free 视图与只写存储。细节见[凭据子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/credentials.md)（`CredentialRef`、`CredentialInfo`、provider source layers）。

## 10.10 身份（identity）

`dsh-anonymous-user-id`（[`packages/identity/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/identity/README.md)）提供共享匿名身份。

## 下一步

数据平面讲完，回到「组合」：这一切如何在启动时拼成一棵树。下一章[第 11 章：启动、组合与预设](11-composition.md)讲 app-boot、bundle、preset 与 agent 自修改。
