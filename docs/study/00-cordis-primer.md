# 00 · Cordis 插件框架速通

DeepSeek Harness 的代码大量使用 [Cordis](../../vendor/README.md)——一个 TypeScript 插件框架。如果你没接触过 Cordis，直接读后续章节会卡在概念层面。这一篇用最小篇幅覆盖你会在 dsh 源码中反复遇到的 Cordis 模式。

**阅读时间**：~15 分钟。建议在第 01 篇之前读完。

## 0.1 Cordis 是什么

Cordis 是一个**插件组合框架**。它的核心思想是：

- **一切皆插件**：模型适配器、工具注册表、会话日志、agent 循环本身——每个部分都是一个插件，都可以从配置中替换。
- **通过 Context 共享服务**：插件在共享的上下文（`ctx`）上注册和发现服务，而不是通过直接导入。
- **注册是可逆的 Effect**：提示词段落、工具 schema、适配器、监听器都通过 `ctx.effect()` 或 `ctx.on()` 安装，卸载时按序回滚。

**与 Effect 库的区别**：OpenCode 用 Effect 做函数式副作用管理；dsh 用 Cordis 做插件组合。两者解决的问题不同——Cordis 不包裹 async 函数，不提供 `yield*` 语法，而是提供插件生命周期管理和服务发现。

## 0.2 Service 与 Context —— 你会看到最多的模式

```ts
// 一个插件声明自己提供什么服务
export class MyService extends Service {
  constructor(ctx: Context, config: Config) {
    super(ctx, config)
    // 通过 ctx 注册自己的能力
  }
}

// 使用时通过 ctx.<key> 获取
const tools = ctx.tools        // 工具注册表
const llm = ctx.llm            // LLM 适配器注册表
const sessions = ctx.sessions  // 会话存储
```

**要点**：

- `Context` 是一个**服务仓库**（proxy 对象）。插件通过 `ctx.<key>` 访问服务，如 `ctx.tools`、`ctx.llm`。
- `Service` 是插件的基类。它的构造函数接收 `ctx` 和 `config`，在构造时注册自己的能力。
- 服务之间**通过 key 发现，不通过 import 导入**。这实现了"可替换"——只要 key 一样，换一个实现就行。

**你在 dsh 里会看到**：几乎每个 `packages/` 下的包都定义一个 Service，注册到 `ctx` 上。

## 0.3 `inject` —— 声明服务依赖

```ts
export class ToolRuntime extends Service {
  static inject = ['tools', 'llm'] as const  // 声明依赖

  constructor(ctx: Context) {
    super(ctx)
    // 此时 ctx.tools 和 ctx.llm 已经存在
    // 因为 Cordis 等待依赖就绪后才挂载这个插件
  }
}
```

**要点**：

- `static inject = [...]` 声明一个插件需要哪些服务才能启动。
- Cordis 按依赖关系决定加载顺序——不需要手动编排启动序列。
- 如果声明的依赖不存在，插件会等待；如果永远不出现，插件就不挂载。

**翻译表**：

| 你看到 | 心里翻译成 |
|---|---|
| `static inject = ['tools', 'llm']` | "这个插件需要 tools 和 llm 服务才能启动" |
| `ctx.tools.register(...)` | "往工具注册表里注册一个工具" |
| `ctx.llm.stream(request)` | "调用 LLM 流式 API" |
| `new SomePlugin(ctx)` | "实例化并挂载一个插件" |

## 0.4 事件（Events）—— 插件间通信

Cordis 提供四种事件派发模式，每种有不同的语义：

| 模式 | 方法 | 是否 await | 派发顺序 | 有返回值？ |
|---|---|---|---|---|
| `emit` | `ctx.emit()` | 否 | 按注册顺序 | 否 |
| `waterfall` | `ctx.waterfall()` | 否 | 按注册顺序 | 是 |
| `parallel` | `ctx.parallel()` | 是 | 并发 | 否 |
| `serial` | `ctx.serial()` | 是 | 按注册顺序，可 bail | 是 |

**事件通过 declaration merging 声明**：

```ts
// 在某个 Service 的类型文件里声明
declare module '@deepseek-ai/cordis' {
  interface Events {
    'tools/execute'(payload: ToolExecutionInput): ToolExecutionResult
  }
}

// 然后在任何插件里监听
ctx.on('tools/execute', (payload) => {
  // 拦截、包装、观测工具执行
})
```

**要点**：

- 事件名是字符串 key（如 `'tools/execute'`、`'agent/pre-step'`）。
- declaration merging 让插件**无需修改声明包**就能添加新事件类型。
- `waterfall` 是 around-middleware：监听器收到 `(...args, next)`，调用 `next()` 委托给下一个监听器，不调用就短路。

**你在 dsh 里会看到**：

```ts
// waterfall 模式：拦截 + 委托
ctx.on('agent/pre-step', (payload, next) => {
  // 可以修改 payload
  const decision = await next()  // 委托给下一个监听器
  // 可以修改 decision
  return decision
})

// emit 模式：纯通知
ctx.emit('agent/status', { agent, status: 'running' })
```

## 0.5 `ctx.effect()` —— 可逆的注册

```ts
// 注册一个工具，返回 disposer
const dispose = ctx.effect(() => {
  ctx.tools.register(myTool)
  return () => {
    // 卸载时清理
    ctx.tools.unregister(myTool)
  }
})
```

**要点**：

- `ctx.effect(callback)` 注册一个有副作用的操作，返回一个 disposer。
- 插件卸载时，所有 effect 按注册的**逆序**回滚。
- 这保证了"热重载"和"卸载"不会留下残留。

**实际用法**：大多数时候你不会手动调 `ctx.effect()`，而是用 Cordis 提供的辅助函数（如 `ctx.on()` 自动返回 disposer），或者在 Service 构造函数里用 `ctx.effect()` 包裹注册逻辑。

## 0.6 Scope —— 作用域隔离

```ts
// 创建一个子上下文，隔离某个服务
const childCtx = ctx.isolate('tools')

// 或者扩展上下文，加额外元数据
const childCtx = ctx.extend({ agentId: 'my-agent' })
```

**要点**：

- `ctx.isolate(name)` 创建一个子上下文，其中 `name` 服务有独立的 scope——可以挂载不同的实现而不影响父上下文。
- `ctx.extend(meta)` 创建一个继承父上下文所有属性、但带有额外元数据的子上下文。
- 这是 dsh 实现"每个 agent 有自己的工具集、提示词、能力"的基础。

**你在 dsh 里会看到**：`agent-loop` 为每个 agent 创建一个 scoped context（`agent.ctx`），所有 agent 级别的注册（工具、提示词段、监听器）都挂在这个 scoped context 上，agent 销毁时自动回滚。

## 0.7 配置（Config）

插件可以通过 Cordis 配置系统接收参数：

```ts
// cordis.yml 里的配置行
- name: dsh-llm-deepseek
  config:
    baseURL: https://api.deepseek.com
    timeout: 30000

# 可以用 !!js 表达式做条件配置
- name: dsh-fs-sandbox
  disabled: !!js process.env.SANDBOX !== '1'
  config:
    mode: write-only
```

**要点**：

- 每个插件有一个 `config` 字段，在 `cordis.yml` 里声明。
- `!!js` 标记允许在 `config` 和 `disabled` 字段中使用 JavaScript 表达式（在插件激活时求值）。
- 其他元数据（如 `name`、`inject`）保持字面值。
- 配置覆盖通过 patch 层实现——上层 patch 可以替换下层整个 config。

## 0.8 一个完整的插件示例

```ts
import { Service, Context } from '@deepseek-ai/cordis'

interface Config {
  maxLength: number
}

export class MyPlugin extends Service {
  static inject = ['tools'] as const

  constructor(ctx: Context, config: Config) {
    super(ctx)

    // 注册一个工具（可逆 effect）
    ctx.effect(() => {
      ctx.tools.register({
        name: 'my_tool',
        description: 'Does something useful',
        // ...
      })
      return () => { /* unregister */ }
    })

    // 监听事件（自动可逆）
    ctx.on('tools/pre-execute', (input, next) => {
      console.log('about to execute', input.name)
      return next()
    })
  }
}
```

**总结**：一个 Cordis 插件 = 声明依赖（`inject`） + 构造时注册能力（`ctx.effect()` / `ctx.on()`） + 通过 `ctx` 发现其他服务。

## 深入追问

- `waterfall` 和 `serial` 都按注册顺序执行，区别在哪？（提示：`next()` 委托 vs bail 短路）
- 为什么 dsh 不用 Effect 库而用 Cordis？（提示：插件组合 vs 函数式副作用）
- `ctx.isolate()` 如何实现"同一个进程里两个 agent 用不同的文件系统后端"？

下一篇 → [01 · 整体定位与分层](./01-overview.md)
