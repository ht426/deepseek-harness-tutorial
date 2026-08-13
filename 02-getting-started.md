# 第 2 章：环境与运行

本章把 `dsh` 真正跑起来，并解释「一个 `dsh` 进程到底是什么」。目标是让你能启动 Web UI / headless 模式、看懂 `--dump-config` 打印出的插件树，并理解 profile 目录里装着什么。

## 2.1 前置要求

- Node.js：支持 `22.19+` 与 `24+`（CI 覆盖 22.19、24、26）。
- pnpm：仓库在 `package.json` 里固定 `pnpm@11.7.0`；若 `pnpm --version` 不走 Corepack，先执行 `corepack enable`。
- Git 2.26+。
- 可选：一把 DeepSeek API key，用于 Web / headless / ACP 演示与真实 API 的 e2e 测试。

## 2.2 两种运行方式

### 通过 npm 直接跑（最快）

```sh
npx @deepseek-ai/dsh web
```

这会启动 Web UI，默认地址 `http://127.0.0.1:3080`。

### 从源码跑

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

`pnpm dsh <args...>` 是仓库里的 TypeScript 入口（走 tsx 的 ESM hook），并把所有参数原样转发给被启动的 profile。

首次克隆后，`pnpm install` 会配置 worktree 本地的 Lefthook 钩子和 `dsh-translation-pairing` Git merge 驱动。跑一次 `pnpm run typecheck` 验证环境就绪。

## 2.3 `dsh` 命令与 entry mode

`dsh` 命令是 profile 的产品启动器：按序叠加插件 bundle 补丁层，再叠上用户自己的覆盖层。命令语法在 [`apps/cli/src/args.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/src/args.ts)，[`src/bin.ts`](https://github.com/deepseek-ai/deepseek-harness/blob/master/apps/cli/src/bin.ts) 只加载被选中的 runner。

| 命令 | 用途 |
|---|---|
| `dsh --profile <name>` | 启动 `$DSH_HOME/profiles/<name>` 下的命名 profile |
| `dsh --profile headless "job"` | 跑一个全新的持久会话，打印最终答案后退出 |
| `dsh web` | `--profile web` 的别名 |
| `dsh plugin --profile <name> <pnpm args>` | 管理 profile 的插件（把参数转发给 profile 目录里的 pnpm） |

当前目录就是默认的 workspace root。`web` 与 `headless` profile 首次使用时从随产品模板自动初始化；其它 profile 必须通过 `dsh plugin` 创建。

**参数归属**：启动器只解析自己的 flag，把后面的 token 全部交给启动起来的 profile。所以启动器 flag 在前，第一个不认识的 token 开始就是 app 的参数：

```sh
dsh --profile web --port 8080       # --port 属于 web app
dsh --profile headless "run the tests"
dsh --profile web --help            # web app 的 flag，不是启动器的
dsh --help                          # 启动器自己的 help
```

## 2.4 Profile 目录里有什么

一个 profile 目录装着：

- `package.json`：树外插件依赖 + profile 清单 `dsh.profile`（含有序的 `bundles` 列表）。
- `cordis.patch.yml`：用户自己的补丁层。

插件树在**空根**上组合，顺序固定：

1. `dsh.profile.bundles` 里按顺序的每个 bundle 的补丁；
2. profile 的 `cordis.patch.yml`；
3. home 级的 `$DSH_HOME/cordis.patch.yml`；
4. `--patch` 覆盖层。

`dsh.profile.bundles` 里列出的 bundle 先从 dsh 安装解析（`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`、`@deepseek-ai/dsh-headless`），再从 profile 自己的 `node_modules` 解析（pnpm 在那里装树外插件）。

用 `--dump-default-config` 和 `--dump-config` 在不真正启动的情况下检查组合出的树：

```sh
dsh --profile web --dump-config
```

## 2.5 凭据

真实 DeepSeek 适配器和需要 key 的演示从环境变量或仓库根被 gitignore 的 `.env` 读凭据：

```sh
DEEPSEEK_API_KEY=sk-...
DEEPSEEK_BASE_URL=https://... # 可选，默认公共 API
```

不要提交真实凭据。没有 `DEEPSEEK_API_KEY` 时，真实 API 的 e2e 套件会自行跳过。

## 2.6 跑起来看看

```sh
# 单次 headless：开一个持久会话、执行任务、打印最终答案、退出（需要 key）
pnpm dsh --profile headless "summarize this workspace"

# 自引用 cordis 演示：agent 检查并修改自己的活插件运行时（需要 key；默认 web，可选 acp）
pnpm run demo:cordis

# ACP 自动化服务器：在 JSON-RPC stdio 上暴露全新 agent 会话（需要 key）
pnpm run demo:acp
```

## 2.7 目录结构速览

从源码运行前，先建立对仓库布局的第一印象（完整说明见根 [`AGENTS.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/AGENTS.md)）：

- `packages/<group>/<pkg>/`：所有 `@deepseek-ai/dsh-*` 工作区，按组归类（`core`、`shell`、`fs`、`session` …）。
- `apps/cli`：`dsh` 命令本身；`apps/web`：Web GUI 的 Vite 入口（不是独立应用——由 `dsh web` 注入 `window.__DSH_BOOT__`）。
- `examples/`：可运行的 `cordis.yml` 叶子，叠加在 `packages/examples` 的 bundle 上，是学习 profile 组合的最佳起点。
- `docs/`：架构、术语表、子系统参考、cookbook。
- `vendor/`：vendored 的 Cordis 源码。

TypeScript 工程用 Host 与 Client 两个隔离的聚合程序（`tsconfig.host.json` / `tsconfig.client.json`），因为两侧对 Cordis `Context` 的 declaration merging 会撞名。普通包只注册进一个聚合；`api/remotes` 是唯一拆成 Host + Client 两份 tsconfig 的包。

## 下一步

现在你有了一个能跑的 `dsh`。在继续深入各模块前，先补上它的地基——[第 3 章：Cordis 框架基础](03-cordis.md)。理解「插件 / 服务 / 事件 / effect」四个词之后，后面的每一章都会变简单。
