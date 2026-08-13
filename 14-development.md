# 第 14 章：开发、测试与文档

最后一章是「怎么在仓库里好好干活」：开发工作流、测试策略、文档纪律与常用命令。完整参考是[开发指南](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/development.md)与[测试策略](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/testing.md)。

## 14.1 日常命令

根 [`AGENTS.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/AGENTS.md#commands) 汇总常用命令；`package.json` 与 `scripts/run-gates.ts` 拥有当前脚本与门清单。选能覆盖改动面的最小检查，而不是无条件跑全套：

```sh
pnpm install            # pnpm workspaces, node ^22.19 || >=24
pnpm run clean          # 移除构建产物与已删包的安全残留
pnpm run test           # vitest 单元测试
pnpm run test:coverage  # CI 覆盖率门：packages/*/*/src 每文件 100%
pnpm run test:e2e       # 真实 API 测试；无 DEEPSEEK_API_KEY 时自跳
pnpm run test:snapshot  # 无 key 的 ACP/headless 回放 vs 期望输出；过滤: -t <name>
pnpm run typecheck
pnpm run lint
pnpm run build          # tsc 产出 lib/types，tsdown 打包运行时
pnpm run hygiene        # knip + publint + workspace 约束 + NodeNext 消费者检查
pnpm run doc-sync       # 全部文档门
pnpm run website:build  # VitePress 构建（兼作死链检查）
```

注意 `test:coverage`（不是 `test`）才是 CI 覆盖率门。GUI 改动跑 `pnpm run test:gui`（秒级内环）；会改变组装浏览器或可见输出的改动再跑 `DSH_SNAPSHOT=replay pnpm run test:web`。

## 14.2 测试策略

测试策略在 [`docs/testing.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/testing.md)，要点：

- **匹配证据到面**：行为用聚焦测试，模型或用户输出用快照，文档用 `doc-sync`，发布路径用 build/hygiene 与构建后 smoke，provider 行为用真实 API e2e。
- **每个非平凡的模型或产品可见行为变更**，在同一 PR 里通过一个真实可运行示例新增或更新 keyless 快照——包测试、仅 e2e 断言、纯 mock 夹具不能替代组装应用的 transcript。
- 夹具必须在 macOS/Linux 上可回放；修夹具，不修 normalizer。
- 产品可见插件需要非单元的真实组合测试（通过 Loader 与 app/process 启动测试专用 `cordis.yml`）。

## 14.3 文档纪律

`docs/` 受文档标准约束（[`docs/AGENTS.md`](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/AGENTS.md)）：每份人类文档归为教程或参考；每份文档自有主题、直系子文档只概述并链接；一个事实一个家。写作规则的关键几条：

- **写当前状态，不写变更史**：避免「以前/现在/不再」、PR、commit；把变更故事放进 commit/PR/Agent Note/postmortem。
- **每个非平凡变更在同一 PR 里至少带一条 Agent Note**（[范围](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/README.md#when-to-write-one)）。
- **段落每段一个物理行**；`ts` 围栏必须能编译（`doc-typecheck`）。
- **双语成对**：`docs/**` 下的文档成对维护（英文 `foo.md` + 中文 `foo.zh.md` + `foo.i18n.yaml`），改动任一侧须在同一 PR 更新对应侧并 `--write` 重新记录（[契约](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/i18n/README.md)）。

本教程是仓库之外的独立目录（`deepseek-harness-tutorial/`），不在 `docs/**` 配对范围内，因此是纯中文、不参与双语门。

## 14.4 仓库布局与 TypeScript 约定

- Host 与 Client 两个隔离聚合程序（`tsconfig.host.json` / `tsconfig.client.json`），普通包注册进恰好一个聚合；`api/remotes` 是唯一拆成 Host+Client 的包。
- ESM 到处（`"type": "module"`）；跨包用包名，包内相对导入用 `.ts`。
- **注册是 effect**：每个贡献经 `ctx.effect()`/`ctx.on()`；注册表的 `register()` 返回 disposer。
- **运行时不变式断言归属关系**：检查权威事件流或可变数据，不是服务/方法存在、插件元数据、effect 或固定纯示例。
- **类型化事件用 declaration merging**；事件 JSDoc 需 `@mode` 与 payload `@param`。
- **switch 判别 tag**；封闭 union 以 `assertNever` 收尾。
- **瀑布监听器必须 `next()`**。
- **模型可见 ⟺ 已入日志**。
- opaque 跨边界 id 用 branded，不用裸 `string`。
- **显式 > 隐式**：默认化是拥有实现里的显式 `resolve(request): Spec`，不是藏在 `run()` 里的 `?? default`。
- **无误吞**：空 `catch` 要说明吞掉什么、为何没有别的路径能到达。

## 14.5 环境与凭据

真实 DeepSeek 适配器读 `DEEPSEEK_API_KEY`（可选 `DEEPSEEK_BASE_URL`）或根 `.env`。不要提交凭据。没有 key 时 e2e 自跳。

## 14.6 提交与门

推前检查用 [dsh-pre-push-checks](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/skills/dsh-pre-push-checks/SKILL.md) 选最窄覆盖；报告只报告跑过的命令。CI 拥有穷尽覆盖与平台矩阵，不要为每次提交重跑全套。

## 结语

到这里，你已经走完 `dsh` 从概念到扩展的完整路径：

1. 它**一切皆插件**，运行中的 `dsh` 是一棵由 bundle/profile 组合出的插件树（第 1、11 章）；
2. 插件、服务、事件、effect 是它的地基（第 3 章）；
3. 会话日志是唯一真源，模型所见必入日志（第 4、10 章）；
4. 能力是三角色的接缝，换 provider 换掉整块能力（第 5–8 章）；
5. 扩展挂到文档化扩展点，改动循环才更新架构图（第 13 章）。

下一步，建议挑一个方向动手：给 `dsh` 加一个 `defineTool` 工具（第 13.2 节），或按[第一个 Harness 插件](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/index.md)从 `cordis.yml` 挂一个自己的插件。遇到精确类型与事件签名，回到[附录：术语速查](appendix-glossary.md)与官方[子系统参考](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/README.md)。
