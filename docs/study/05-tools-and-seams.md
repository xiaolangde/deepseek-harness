← 上一篇：[04 · LLM 流式与 Provider 适配](./04-llm-streaming.md)

# 05 · 工具系统与能力 Seam

这一篇放大 turn 旅程中的工具调用环节——从 `ToolDefinition` 到执行管线到能力 seam。

dsh 的工具系统不是一个简单的"函数注册表"，而是一个有权限检查、沙箱隔离、结果改写、UI 呈现的完整管线。而能力 seam 则是 dsh "换一个 provider 就改变整个产品"的基础。

## 5.1 ToolDefinition —— 一个工具是什么

每个注册到 `ctx.tools` 的工具是一个 `ToolDefinition`。它扩展了 `ToolSchema`（name / description / parameters）并加上执行逻辑和输出声明。

```ts
interface ToolDefinition extends ToolSchema {
  // 强制规范输出声明
  output: ToolOutputDefinition

  // 执行函数：返回 lossless-JSON 值
  execute(args: unknown, exec: ToolRunContext): Promise<unknown>

  // 最后一英里内容变换（对每个结果调用一次，必须 total）
  finalizeContent?(exec, result): ContentBlock[] | undefined

  // 协作超时（由 timeout-policy 执行，不发给模型）
  timeoutMs?: number

  // 纯同步分类：是否可以并发执行
  isConcurrencySafe?(args): boolean

  // UI 呈现（纯函数，无副作用）
  presentCall?(args): ToolCallView | undefined
  presentResult?(args, result): ToolResultView | undefined
}
```

**关键设计**：

- **`execute` 返回 `unknown`**——一个 lossless-JSON 值。工具自己负责验证参数（`args` 是冻结的 `unknown`，工具自校验）。
- **`output` 是强制的**——每个工具必须声明输出的 JSON Schema 和 render 函数。这保证了输出是类型安全的、可验证的。
- **`finalizeContent` 必须 total**（不抛异常）——它是对每个结果（含管线失败）调用的最后一道内容变换。
- **`presentCall` / `presentResult` 是纯函数**——UI 和日志重放共用同一份呈现逻辑，不会因为渲染而产生副作用。

## 5.2 defineTool DSL —— 类型安全的工具定义

dsh 提供了一个 `defineTool` DSL，让工具定义类型安全且简洁：

```ts
const myTool = defineTool({
  name: 'read_file',
  description: 'Read a file',
  parameters: {
    path: { type: 'string', required: true },
    encoding: { type: 'string' },  // 可选
  },
  output: {
    schema: { type: 'object', properties: { content: { type: 'string' } } },
    render(args, value) {
      return [{ type: 'text', text: value.content }]
    },
  },
  async execute(args, exec) {
    const content = await readFile(args.path)
    return { content }
  },
})
```

**DSL 的类型推导**：

- `InferArgs<P>` 把每属性的 `required` 转为必选/可选键。
- `InferValue<OutputSchema>` 把 output schema 推导为 TypeScript 类型。
- 参数不匹配抛 `ToolArgsError`（`INVALID_ARGS`）；非法输出抛 `ToolOutputError`（`INVALID_TOOL_OUTPUT`）。

**JSON Schema 子集**：dsh 使用一个受限制的 JSON Schema 子集（`JsonSchemaNode`），支持 `string` / `number` / `integer` / `boolean` / `null` / `array` / `object` / `json` / `oneOf`。`oneOf` 需要至少 2 个分支且恰好匹配一个。

## 5.3 工具执行管线

工具执行是一个多阶段管线，每个阶段都是 waterfall：

```
模型输出 tool-call 块
  │
  ▼
tool/call (持久化：先记录，后执行)
  │
  ▼ 1. tools/pre-execute (waterfall)
  │     返回 PreToolDecision:
  │     ├── allow → 继续
  │     ├── deny → 跳过执行
  │     └── ask → ctx.approval (一次性用户审批)
  │
  ▼ 2. 注册的单调守卫 (ToolGuard)
  │     只能减权限：deny 或 abstain
  │
  ▼ 3. tools/execute (waterfall: around-dispatch)
  │     ├── timeout-policy (超时截止)
  │     ├── retry / metrics
  │     └── tool.execute(args) → 结构化 result
  │
  ▼ 4. tools/post-execute (waterfall)
  │     返回 PostToolDecision:
  │     ├── accept → 通过
  │     ├── block → 替换为错误结果
  │     ├── replace → 替换为新结果
  │     └── add context → 注入额外上下文
  │
  ▼ 5. finalizeContent
  │     最后一英里内容变换
  │
  ▼ 6. tool/result (持久化：面向模型的结果)
  │     + tools/result (同步通知：冻结权威结果)
```

**三个 waterfall 可以变换一次调用**：

1. **`tools/pre-execute`**：决定是否执行（权限/审批）。
2. **`tools/execute`**：包装执行过程（超时/重试/metrics）。
3. **`tools/post-execute`**：变换结果（accept/block/replace/add context）。

**单调守卫**：`ToolGuard` 只能减权限（deny 或 abstain），不能加权限。这保证了"如果一个守卫 deny 了，后面的监听器不能 un-deny"。

## 5.4 工具执行模式

工具按 `isConcurrencySafe(args)` 分为两种执行模式：

| 模式 | 行为 | 示例 |
|---|---|---|
| `parallel` | 可与兄弟工具重叠执行 | `read`、`glob`、`grep` |
| `exclusive` | 独占屏障，必须等前一个完成 | `write`、`edit`、`bash` |

循环根据执行模式形成**屏障与滚动池**：

```
模型一次输出 4 个工具调用:
  read(a)  [parallel]
  read(b)  [parallel]
  write(c) [exclusive]
  read(d)  [parallel]

执行顺序:
  ├── 并发: read(a) + read(b)
  ├── 屏障: 等待 read(a) + read(b) 完成
  ├── 独占: write(c)
  ├── 屏障: 等待 write(c) 完成
  └── 并发: read(d)
```

## 5.5 ToolExecutionResult —— 执行结果

```ts
type ToolExecutionResult =
  | ToolExecutionSuccess
  | ToolExecutionFailure

interface ToolExecutionSuccess {
  value: unknown             // 规范 lossless-JSON 值（循环不持久化）
  content: ContentBlock[]    // 面向模型的内容
  meta?: JsonValue           // 额外元数据
  additionalContexts?: UserMessage[]  // 注入额外上下文
  concludesTurn?: boolean    // 标记成功结果终止当前 turn
}

interface ToolExecutionFailure {
  error: ToolFailure          // 结构化错误
}
```

**关键设计**：

- **`value` 是执行本地的**——循环只持久化 `content` / `error` / `meta`。`value` 不进日志。
- **`concludesTurn`** 是一种数据驱动的 turn 终止——工具结果可以标记"这个 turn 该结束了"，不需要等 `agent/turn-stopping`。
- **`additionalContexts`** 在记录的工具结果后作为 `user/message` 注入——同 step 的额外上下文 FIFO 排序。

## 5.6 能力 Seam —— 三角色模式

dsh 的每个可交换能力都是一个 **seam**，包含三个角色：

```
┌─────────────────────┐
│  Service Definition  │  声明接口、类型、事件词汇
│  (dsh-fs, dsh-shell) │  注册到 ctx.fs / ctx.shell
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │             │
    ▼             ▼
┌─────────┐  ┌──────────┐
│ Provider │  │ Consumer  │
│ (实现)    │  │ (使用者)  │
│ fs-local │  │ tool-fs   │
│ fs-sandbox│ │ (read/    │
│          │  │  write/   │
│          │  │  edit)    │
└─────────┘  └──────────┘
```

**为什么三个角色**：

- **Service Definition** 是类型契约——定义"能做什么"，不关心怎么做。它是一个 Cordis 插件，注册 `ctx.<key>`。
- **Provider** 是具体实现——可以有多个，通过配置选择。换一个 provider 就改变执行环境。
- **Consumer** 是使用者——通常是模型工具，通过 `ctx.<key>` 调用能力。它不知道背后是哪个 provider。

**核心效果**：文件系统（`ctx.fs`）和子进程（`ctx.subprocess`）provider 共享一个执行世界。所以把它们指向远程沙箱（如 E2B）时，Bash、终端、LSP 都跟着走——不需要为每个工具写 provider 分支。

## 5.7 现有能力 Seam 一览

| Seam | ctx key | Service Definition | Providers | Consumer |
|---|---|---|---|---|
| 文件系统 | `ctx.fs` | `dsh-fs` | `fs-local` / `fs-sandbox` / `fs-e2b` | `tool-fs` |
| 子进程 | `ctx.subprocess` | `dsh-subprocess` | `subprocess-local` / `subprocess-e2b` | `tool-bash` |
| Shell | `ctx.shell` | `dsh-shell` | `bash-local` / `bash-sandbox` / `pwsh-local` | `tool-bash` |
| 终端 | `ctx.terminals` | `dsh-terminal` | `terminal-bash` | `tool-terminal` |
| 沙箱 | `ctx.sandbox` | `dsh-sandbox` | `sandbox-local` | — |
| LSP | `ctx.lsp` | `dsh-lsp` | `lsp-stdio` | `tool-lsp` |
| Web | `ctx.web` | `dsh-web` | `web-search-exa` / `web-fetch-http` / ... | `tool-web` |
| 子代理 | `ctx.subagents` | `dsh-subagent` | spawn-in-process / fork / acp / codex / claude-code | `tool-subagent` |
| 代码运行 | `ctx.codeRuntime` | `dsh-code-runtime` | `code-runtime-worker-thread` | — |
| 压缩 | `ctx.compaction` | `dsh-compaction` | `compaction-basic` | — |
| 持久化 | `ctx.sessionPersistence` | `dsh-session-persistence` | `jsonl` / `sqlite` | — |
| 设置 | `ctx.settings` | `dsh-settings` | `settings-file` | — |
| 凭证 | `ctx.credentials` | `dsh-credentials` | `credentials-local` | — |

## 5.8 一个 Seam 示例：文件系统

以文件系统 seam 为例，看三角色如何协作：

```
1. Service Definition (dsh-fs):
   - 声明 FsTarget, FsReadResult, FsWriteResult 等类型
   - 声明 fs/* 事件词汇 (fs/observed, fs/write-intent, ...)
   - 注册 ctx.fs 服务接口

2. Provider (dsh-fs-local):
   - 实现 ctx.fs 的方法 (read, write, edit, ...)
   - 直接调用 Node.js fs API
   - 注册到 ctx.fs

3. Consumer (dsh-tool-fs):
   - 定义 read/write/edit 工具 (defineTool)
   - 注册到 ctx.tools
   - execute() 里调 ctx.fs.read(args.path)
   - 不知道背后是 fs-local 还是 fs-sandbox
```

**如果换 provider**：

```yaml
# cordis.yml
# 从本地文件系统切到沙箱：
- name: dsh-fs-local
  disabled: true          # 禁用本地
- name: dsh-fs-sandbox    # 启用沙箱
  config:
    mode: write-only
    workspaceRoot: /workspace
```

所有文件工具（`read`、`write`、`edit`）自动切到沙箱执行，不需要改任何工具代码。

## 5.9 工具呈现 —— UI 渲染意图

工具的 UI 呈现是**设计的一部分**，不是事后补丁。每个工具在定义时就声明它的呈现方式：

```ts
interface ToolCallView {
  kind: 'card'
  variant: 'generic' | 'terminal' | 'diff' | 'search' | 'read' | 'web'
  title?: string
  locations?: FileLocation[]
  diff?: FileDiff[]
  // ...
}
```

**呈现方法是纯函数**——`presentCall(args)` 和 `presentResult(args, result)` 都是无副作用的纯函数，输入是参数/结果，输出是视图描述。UI 和日志重放共用同一份呈现逻辑。

这意味着：

- 工具的 UI 卡片不需要额外的渲染插件——工具自带呈现。
- 日志重放时，UI 卡片能从参数和结果精确重建——不需要存储渲染状态。
- 不同的前端（Web、CLI）可以解释同一份视图描述，各自渲染。

## 深入追问

- 为什么 `ToolGuard` 只能减权限？（提示：安全单调性——一旦 deny 不能被 un-deny）
- `concludesTurn` 和 `agent/turn-stopping` 有什么关系？（提示：数据驱动 vs 事件驱动，前者优先）
- 如果一个工具的 `execute` 抛异常，管线如何处理？（提示：异常被捕获，归一为 `ToolExecutionFailure`）
- 为什么工具参数用受限 JSON Schema 子集而不是完整 JSON Schema？（提示：验证一致性 + 模型兼容性）

下一篇 → [06 · 持久化与启动组合](./06-persistence-and-boot.md)
