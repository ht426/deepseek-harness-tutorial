# 第 6 章：执行类能力

这一章讲「让代码真正跑起来」的能力族：shell、subprocess、terminal、code-runtime、sandbox，以及 POC 状态的 e2b。它们的核心设计是一条：**共享同一个执行世界**——换一个 provider，Bash、PTY 终端、LSP 一起跟着走。

## 6.1 共享执行世界

看这张依赖关系图（摘自[能力接缝图](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md)）：

- `ctx.subprocess` 是公共的 spawn 原语：bash 执行器（`bash-local`/`bash-sandbox`）、PTY 后端（`terminal-bash`）、LSP 宿主（`lsp-stdio`）、out-of-process 的 ACP/Codex/Claude Code subagent 后端都通过它 spawn。
- `ctx.sandbox` 是进程约束接缝：consumer 把「即将 spawn 的精确 argv」交给它，同一世界的后端按每次调用策略包一层并报告 enforcement。
- `ctx.sandboxPolicy` 是部署默认 mode + workspace root 的唯一归属；沙箱执行器与 provider 都读它，因此 bash 与 fs 不会约束到不同的根。

把 `ctx.fs` 和 `ctx.subprocess` 的 provider 指向远程沙箱（e2b），Bash、PTY、LSP 就一起移动，**无需为 provider 分叉出不同版本**。

## 6.2 shell（bash / pwsh）

bash 能力族（[`packages/shell/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/shell/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-shell` | Service Definition：抽象 `ShellExecutor`（`run`/`start`/`sandboxMode`） | `ctx.shell` |
| `dsh-bash-local` / `dsh-pwsh-local` | 本地实现 | （注册 `ctx.shell`） |
| `dsh-bash-sandbox` / `dsh-pwsh-sandbox` | 经 `ctx.sandbox` 约束的实现 | （注册 `ctx.shell`） |
| `dsh-shell-env` | 托管 bash 环境注册表（`DSH_*` 事实） | `ctx.shellEnv` |
| `dsh-tool-bash` / `dsh-tool-pwsh` | 模型面 shell 工具 | （注册 `ctx.tools`） |
| `dsh-tool-bash-persistent` | 持久 bash 会话工具 | （注册 `ctx.tools`） |

关键概念：

- **两段式请求**：`ShellExecRequest`（模型面请求）→ `ShellExecSpec`（解析后的 spec）。默认化是拥有实现里的显式 `resolve(request): Spec` 步骤，绝不藏在 `run()` 里的 `?? default`。
- **前台 `run` 只对基础设施故障 reject**：命令跑完但退出码非零是「正常结果」，不是异常。
- **`sandboxMode` 是能力事实**：consumer 据此判断该执行器是否受限，并据此渲染/提示。
- 插件通过 `ctx.shellEnv` 声明 effect 作用域的 `DSH_*` 事实；每个 shell 工具每次执行收集一份可信快照，执行器据此重建命名空间。

> 源码核对：`dsh-tool-bash` / `dsh-tool-pwsh` 的 inject 是 `['tools','shell','systemPrompt','shellEnv']`（部分 README 旧写为 `bash`/`bashEnv`，以源码为准）。

细节见[shell 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/shell.md)（`ShellExecRequest`/`Spec`、`ShellRunResult`、后台 `ShellProcess` handle）。

## 6.3 subprocess（子进程）

子进程能力族（[`packages/subprocess/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/subprocess/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-subprocess` | Service Definition：`SubprocessRuntime`（`spawn`/`spawnTerminal`/`resolveExecutable`） | `ctx.subprocess` |
| `dsh-subprocess-local` | 本地进程树提供方 | （注册 `ctx.subprocess`） |

关键概念：

- **零默认 spec**：`SubprocessSpawnSpec` 完全显式——不隐藏默认值。
- **offset 读取不消费**：输出读取器按 offset 读取，不消费底层流。
- **树级 `terminate()`**：杀掉整棵进程树，而非单个 pid。
- **`scrubbedParentEnv()`**：清理父环境。

细节见[子进程子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/subprocess.md)（`SubprocessSpawnSpec`、`SubprocessOutcome`、托管的 `DSH_*` 环境词汇）。

## 6.4 terminal（持久终端）

持久 PTY 能力族（[`packages/terminal/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/terminal/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-terminal` | Service Definition：`TerminalSessionService`（品牌化 `TerminalSessionId` + 精确 Agent owner 栅栏） | `ctx.terminals` |
| `dsh-terminal-bash` | bash 后端（type `'shell'`） | （注册 `ctx.terminals`） |
| `dsh-tool-terminal` | 六个 `terminal_*` 模型面工具 | （注册 `ctx.tools`） |

注册表拥有「精确到 Agent 的会话身份」与清理；后端拥有终端机制；`tool-terminal` 暴露 owner 作用域的模型工具。细节见[终端子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/terminal.md)。

## 6.5 code-runtime（代码执行）

代码执行能力族（[`packages/code-runtime/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/code-runtime/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-code-runtime` | Service Definition：`CodeRuntime` | `ctx.codeRuntime` |
| `dsh-code-runtime-worker-thread` | worker-thread 后端 | （注册 `ctx.codeRuntime`） |

关键概念：

- 跑一个模型编写的程序，对着宿主提供的异步绑定。
- **`CodeRunFailure` 正交 kind 分类**：错误按独立维度分类，而非单一错误码。
- **可移植绑定标识符契约**：`PORTABLE_RESERVED_WORDS` 等约束，让绑定名可移植。
- **worker-thread 后端是 containment（隔离事件循环），不是安全边界**——`node:vm` / worker 不构成安全隔离。

细节见[代码运行子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/code-runtime.md)（`CodeRunRequest`/`Result`、绑定命名空间、捕获日志）。

## 6.6 sandbox（沙箱）

进程约束接缝（[`packages/sandbox/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sandbox/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-sandbox` | Service Definition：`SandboxProvider.confine()` 返回 `ConfinedArgv` | `ctx.sandbox` |
| `dsh-sandbox-policy` | 部署默认 mode + workspace root 的唯一归属；拥有 `sandbox/mode` 事件写路径 | `ctx.sandboxPolicy` |
| `dsh-sandbox-local` | 本地后端 | （注册 `ctx.sandbox`） |
| `dsh-sandbox-windows-acl` | Windows ACL 受限令牌后端（partial） | （注册 `ctx.sandbox`） |

关键概念：

- `confine()` 返回 `ConfinedArgv`，含 `denialSignatures` / `runnerFailureRules`——enforcement 与 fail-closed 错误的依据。
- 每会话策略解析 + 文件效果模式（read-only / workspace-write / danger-full-access）、执行/provider 策略、`ConfinedArgv`、enforcement 与 fail-closed 错误，见[沙箱子系统](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/sandbox.md)。
- Linux 侧的 Landlock 原生 addon 源在 [`native/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/native/README.md)。

## 6.7 e2b（POC）

E2B 能力族是 POC（[`packages/e2b/`](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/e2b/README.md)）。

| 包 | 角色 | ctx key |
|---|---|---|
| `dsh-e2b` | 拥有一个共享 E2B SDK handle、远程工作目录与最终沙箱处置 | `ctx.e2b` |
| `dsh-fs-e2b` | E2B 支撑的 `ctx.fs` 实现 | （注册 `ctx.fs`） |
| `dsh-subprocess-e2b` | E2B 支撑的 `ctx.subprocess` 实现 | （注册 `ctx.subprocess`） |

两个适配器**替换** `ctx.fs` 与 `ctx.subprocess` 的 provider，于是既有的 bash / PTY / LSP consumer **零分叉**地把执行世界平移到远程 Linux 运行时。

## 下一步

执行类能力解决了「跑命令、跑代码」。下一章[第 7 章：工具与数据类能力](07-tool-capabilities.md)看文件、LSP、技能、Web 等数据/工具能力；随后是[第 8 章：编排](08-orchestration.md)。
