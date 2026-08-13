# 第 3 章：Cordis 框架基础

`dsh` 的每个零件都是 Cordis 插件，所以这一章是整个教程的地基。这里给出**概念 + 最小代码**；如果你想跟着动手敲一遍，走官方 [Cordis 教程](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-tutorial/index.md)（7 个可运行章节）。本章的目标：让你读懂任何 `dsh` 插件源码里的 `ctx`、`inject`、`ctx.effect`、`ctx.on`、`ctx.waterfall`。

## 3.1 五个核心概念

Cordis 可以用五个概念概括（详见 [primer](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cordis-primer.md)）：

1. **插件是实现了 Service 的对象**：它可以是一个带可选 `inject` 与 `apply(ctx)` 字段的函数，也可以是一个 `Service` 子类（其生命周期由 Cordis 挂进当前 context）。
2. **Context 是服务的仓库**：一个服务从 context 认领一个稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`）；其它插件通过 key 找服务，而不是 import 具体实现。
3. **用 `inject` 声明服务依赖**：命名了必需服务的插件会等到这些服务存在才启动——加载顺序由服务依赖表达，而不是手工编排启动顺序。
4. **类型化事件用于通信**：服务通过 TypeScript declaration merging 声明事件名，然后按 `emit` / `waterfall` / `parallel` / `serial` 分发，分别对应「观察 / 包裹 / 扇出 / 顺序执行」。
5. **注册是可逆 effect**：prompt 段、工具 schema、适配器、provider、监听器都通过 `ctx.effect()` 或 `ctx.on()` 安装，因此重载与拆卸时能被可预测地撤销。

## 3.2 提供一个服务

一个 `Service` 子类本身就是插件（类形式）。运行期与编译期两件事配合：

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    greeter: GreeterService
  }
}

export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')
  }

  greet(who: string) {
    return `Hello, ${who}!`
  }
}

export const name = 'greeter'

export function apply(ctx: Context) {
  ctx.plugin(GreeterService)
}
```

- **运行期**：`super(ctx, 'greeter')` 把实例以名字 `greeter` 注册；此后任何插件都能用 `ctx.greeter` 访问。注册是 effect——卸载 provider 会移除服务。
- **编译期**：`declare module '@deepseek-ai/cordis'` 是 TypeScript declaration merging，给 `Context` 接口加 `greeter`，让 `ctx.greeter` 到处通过类型检查。它不产生代码。

## 3.3 用 `inject` 消费服务

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'consumer'
export const inject = ['greeter']

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))
}
```

`inject` 列出该插件必需的服务。Cordis 让插件停在 PENDING，直到所有列出的服务都存在；因此在 `apply` 内部，`ctx.greeter` 保证已就绪。`cordis.yml` 里的行序无关紧要——依赖决定启动时机，不是文件顺序。

`inject` 不是一次性的启动检查：如果必需服务在运行中消失（provider 被卸载或热替换），依赖它的插件也会被卸载，服务回来时再加载。配合 effect，运行中的 consumer 不会握着一个已不可用的服务引用——依赖消失时它自己的注册也会被撤销。这正是「服务可替换」的原理：卸载 `dsh-bash-local`、挂载另一个 `shell` provider，所有注入 `'shell'` 的插件干净地重启到新实现。

**可选依赖**：能没有也能活的场景，跳过 `inject`，在使用点探测：

```ts ignore-check
export function apply(ctx: Context) {
  const greeter = ctx.get('greeter') // 没有 provider 时为 undefined；插件照常运行
  console.log(greeter?.greet('maybe') ?? 'no greeter available')
}
```

> 注意区分：可选服务用 `ctx.get(name)`，而声明注入用 `ctx.<name>`——属性代理对拓扑敏感，`ctx.get` 读的是全局服务存储。

## 3.4 事件：声明、发出、监听

服务支持直接调用；事件让插件在不知道谁在听的情况下宣布一件事。`dsh` 用事件处理工具结果、模型请求、审批决定等交互。

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    stats: StatsService
  }
  interface Events {
    'stats/report'(name: string, count: number): void
  }
}

export class StatsService extends Service {
  private counts = new Map<string, number>()

  constructor(ctx: Context) {
    super(ctx, 'stats')
  }

  bump(name: string) {
    const next = (this.counts.get(name) ?? 0) + 1
    this.counts.set(name, next)
    this.ctx.emit('stats/report', name, next)
  }
}
```

`interface Events` 合并是事件系统的 `interface Context` 合并的孪生：声明事件名与监听器签名，让 `ctx.emit` 和 `ctx.on` 完全类型化。`namespace/action` 命名约定让扁平事件命名空间可读。

监听方：

```ts ignore-check
import type { Context } from '@deepseek-ai/cordis'
import type {} from './stats.ts'

export const name = 'reporter'
export const inject = ['stats']

export function apply(ctx: Context) {
  ctx.on('stats/report', (name, count) => {
    console.log(`[stats] ${name} -> ${count}`)
  })
}
```

`import type {} from './stats.ts'` 运行期不导入任何东西，只为让 TypeScript 看到 declaration merging。因为 `ctx.on()` 是 effect，监听器随插件一起消失——永远不用手动 `removeListener`。

## 3.5 分发模式

一个事件用哪种分发模式，是其公开契约的一部分：它决定监听器能否返回值、能否并发、能否互相短路。

| 模式 | 调用 | 语义 |
|---|---|---|
| emit | `ctx.emit(name, ...args)` | 同步广播；返回的 promise 与值不被 await 或收集 |
| parallel | `await ctx.parallel(name, ...args)` | 所有监听器并发运行，一起 await |
| serial | `await ctx.serial(name, ...args)` | 监听器按序 await；第一个非 `null`/`false`/`undefined` 的返回值胜出并停下其余 |
| bail | `ctx.bail(name, ...args)` | serial 的同步版本 |
| waterfall | `ctx.waterfall(name, ...args, next)` | 环绕式中间件，见下 |

每个 `dsh` 事件都在其所属[子系统页](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/README.md)的生成参考里注明 `@mode`。

## 3.6 Waterfall：改写或短路

Waterfall 是驱动拦截的模式。每个监听器收到参数外加一个 `next()` 续体；它可以改写 `next()` 的返回值，或不调用 `next()` 直接短路——即「否决」。

```ts ignore-check
declare module '@deepseek-ai/cordis' {
  interface Events {
    'demo/transform'(input: string, next: () => Promise<string>): Promise<string>
  }
}

export const name = 'waterfall-demo'

export function apply(ctx: Context) {
  ctx.on('demo/transform', async (input, next) => {
    const downstream = await next()
    return downstream.toUpperCase() // 包装下游结果
  })
  ctx.on('demo/transform', async (input, next) => {
    if (input.includes('blocked')) return '** blocked **' // 拥有决定权时短路
    return next()
  })
}
```

由此而来的纪律（本仓库的长期规则）：**一个只观察或注解的 waterfall 监听器必须调用 `next()`**；不调用就返回是刻意的短路。一个日志监听器忘写 `next()`，会悄悄吞掉所有人的默认行为。

`dsh` 用 waterfall 承载「协作插件可以包裹或回答」的决定：`agent/request` 让插件替换模型调用配置，`approval/request` 让策略代替用户作答。

## 3.7 配置与组合

`@deepseek-ai/cordis-plugin-include` 把 `!!js` 解析成表达式节点。Loader 在声明注入激活后、针对该插件 context 插值一个 entry 的 `config`，并在每次 mount 决定时（针对 loader context）插值 `disabled` 字段；其它元数据保持字面值。环境要选择插件时，用 overlay。

一个 `cordis.yml` 就是一棵插件树（entry 列表）。组合机制在第 11 章讲 bundle/profile 时展开。

## 3.8 实用规则

- 把行为封装进插件：工具流水线事件归 `ctx.tools`，模型流式归 `ctx.llm`，活跃 agent 协调归 `ctx.agents`。
- 拦截与策略优先用事件；直接能力调用用服务方法。
- 每个注册都要有 disposer：从 `ctx.effect()` 返回一个，或用自动处理它的 Cordis helper。若拆卸顺序重要，把相关工作放进同一个 effect，使销毁按预期顺序回退。

## 下一步

地基打好了。进入[第 4 章：核心与 Agent 循环](04-core-loop.md)，看这些概念如何在 `packages/core/` 里变成 `ctx.sessions`、`ctx.tools`、`ctx.agents` 与 turn/step 流转。
