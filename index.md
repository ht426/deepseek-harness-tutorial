# DeepSeek Harness 教程

本目录是一份从源码出发、按模块组织的 **DeepSeek Harness（`dsh`）教程**。它带你从「`dsh` 是什么」一路走到「如何在 `dsh` 上写出自己的插件」，每一章都对应仓库里的一组真实包（`packages/<group>/<pkg>/`），并标注关键源码与类型定义的位置，方便你随时回到源码核对。

> 本教程用中文编写；代码标识符、类型名、事件名、`ctx` 键等保持英文原文，首次出现时给出中文说明。

## 第一次接触？先读这篇

如果你对「agent / 智能体 / 大模型」这些词还不熟，**先读[零基础导读](00-basics.md)**——它用大白话讲清「agent 是什么、大脑和手怎么配合、dsh 为什么一切皆插件」，几分钟读完，后面 14 章就都能跟上了。

## 这份教程适合谁

- 想读懂 `dsh` 源码、理清其模块边界的开发者。
- 想给 `dsh` 加工具、加能力、加模型适配器、加 Web UI 节点的插件作者。
- 想用 `dsh` 组装自己的 agent 应用（CLI / Web / ACP / JSON-RPC）的集成者。

## 前置知识

- 会写现代 JavaScript / TypeScript（本教程不重讲语法；需要时可参考官方 [Cordis 教程的 TypeScript 笔记](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md#typescript-notes)）。
- 装了 Node.js（要求 `^22.19 || >=24`）与 pnpm。
- 对「大语言模型调用 + 工具调用」的基本流程有概念即可（没有也不怕，[零基础导读](00-basics.md)会补齐）。

## 如何使用

按顺序读即可——每一章都建立在前一章的概念之上。第 3 章是 Cordis 框架速览，是理解后续所有章节的地基；如果你已经读过官方 [Cordis 教程](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md)，可以快速跳过并只保留「服务 / 事件 / 接缝」三个术语。

各章在结尾给出「下一步」指针和对应的源码入口，方便按需深入。

## 章节地图

| 章节 | 内容 | 对应的仓库模块 |
|---|---|---|
| [零基础导读](00-basics.md) | 用大白话讲清 agent、大脑/手、插件、接缝——第一次接触必读 | — |
| [第 1 章：总体架构](01-overview.md) | 一切皆插件、profile/bundle、能力接缝、轮次流转、扩展点总览 | `vendor/`、`packages/bundle/`、`packages/boot/` |
| [第 2 章：环境与运行](02-getting-started.md) | 安装、从源码跑、profile、`--dump-config`、目录结构 | 仓库根、`apps/cli` |
| [第 3 章：Cordis 框架基础](03-cordis.md) | 插件、Context、服务、事件、effect/on、配置与组合 | `vendor/cordis` |
| [第 4 章：核心与 Agent 循环](04-core-loop.md) | session / system-prompt / tools / agent / agent-loop，turn/step 流程 | `packages/core/` |
| [第 5 章：LLM 能力](05-llm.md) | Message/StreamChunk、LlmAdapter 接缝、DeepSeek 提供方、token-meter | `packages/llm/` |
| [第 6 章：执行类能力](06-execution-capabilities.md) | shell、subprocess、terminal、code-runtime、sandbox、e2b | `packages/shell/` 等 |
| [第 7 章：工具与数据类能力](07-tool-capabilities.md) | fs、lsp、skill、web、mcp、attachment、spill | `packages/fs/` 等 |
| [第 8 章：编排](08-orchestration.md) | subagent、workflow、todo、plan、goal、schedule、jobs、compaction、guard | `packages/subagent/` 等 |
| [第 9 章：上下文与人类协作](09-interaction.md) | context、commands、approval、permission-presets、ask-user、feedback | `packages/context/`、`packages/interaction/` |
| [第 10 章：持久化与数据平面](10-data-plane.md) | session 持久化、投影、查询、storage、workspace、settings、credentials | `packages/session/` 等 |
| [第 11 章：启动、组合与预设](11-composition.md) | app-boot、bundle、preset、extensions（自修改） | `packages/boot/`、`packages/bundle/` 等 |
| [第 12 章：接入与对外接口](12-integration.md) | api/typert 网关、sdk、acp、hooks、Web GUI（host/client） | `packages/api/`、`packages/sdk/` 等 |
| [第 13 章：如何扩展 dsh](13-extending.md) | 加工具、加提供方、加包、加 LLM 适配器、加 Chat 节点 | `docs/cookbook/` |
| [第 14 章：开发、测试与文档](14-development.md) | 开发工作流、测试策略、文档规范、常用命令 | `scripts/`、`docs/` |
| [附录：术语速查](appendix-glossary.md) | turn/step、seam、scope、goal 等规范术语 | `docs/glossary.md` |

## 官方文档索引

本教程是「教学顺序」的路线图；官方文档是「查找」用的权威参考。需要精确类型、事件签名或配置字段时，优先查：

> 本教程完全自包含，不依赖本地源码仓库：文中所有 `packages/...`、`docs/...`、`apps/...` 等源码/文档链接，都指向 GitHub 上的 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 仓库（`master` 分支）。

- 架构总览：[docs/architecture.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/architecture.md)
- 术语表：[docs/glossary.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/glossary.md)
- 能力接缝与 `ctx` 服务图：[docs/capability-seams.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/capability-seams.md)
- 轮次/步骤时序：[docs/agent-lifecycle.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/agent-lifecycle.md)
- 工具执行流水线：[docs/tool-execution-pipeline.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/tool-execution-pipeline.md)
- Cordis 入门：[docs/cordis-primer.md](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md)、[docs/cordis-tutorial/](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md)
- 各子系统参考：[docs/subsystems/](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/README.md)
- 各包 README：[packages/](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/README.md)

> 注意：`dsh` 目前处于**开发者预览**阶段，接口会以破坏兼容的方式快速演进。本教程以仓库当前源码为准。
