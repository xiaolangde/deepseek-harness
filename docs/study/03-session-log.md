← 上一篇：[02 · 一次 Turn 的旅程](./02-turn-journey.md)

# 03 · 会话日志与事件溯源

上一篇把一次 turn 的完整旅程串了起来。这一篇放大旅程中最重要的数据结构——**会话日志**。

dsh 的会话日志是一个 append-only 的事件流。它不是"聊天记录"，而是"agent 交互历史的唯一真相源"。LLM 消息历史从日志**派生**，不单独存储。

## 3.1 为什么用事件溯源

传统做法是"存消息列表"——每条消息一行，改了就 update。dsh 不这样做，原因：

1. **不可变性**：事件日志只追加，不改不删。任何时刻的状态都是一个投影，可以重新算。
2. **可重放**：给一个事件流，能完整重建模型所见的历史——包括工具调用、中途注入的上下文、token usage。
3. **崩溃恢复**：重载日志，发现未关闭的 turn，用合成事件关闭它。不截断、不迁移。
4. **多投影**：同一个日志可以派生出不同视图——模型历史、UI 转录、遥测统计、标题生成。

**代价**：事件日志比"消息列表"更啰嗦。一次 turn 可能产生几十个事件。但 dsh 认为这个代价值得——因为"模型看到了什么"是一个必须精确重建的事实。

## 3.2 SessionEvent —— 事件类型

`SessionEventMap` 是 merge-extensible 的事件词汇表。插件可以通过 declaration merging 添加新事件类型，不需要修改 `dsh-session` 包。

核心事件变体：

| 事件 | 载荷 | 用途 |
|---|---|---|
| `turn/start` | `{turn}` | 开启一个 turn |
| `turn/end` | `{turn, reason: TurnEndReason}` | 关闭一个 turn |
| `step/start` | `{turn, step}` | 开启一个 step |
| `step/end` | `{turn, step}` | 关闭一个 step |
| `user/message` | `UserMessage` | 用户消息（直接 prompt、inject 上下文、目标续行） |
| `assistant/chunk` | `{turn, step, chunk: StreamChunk}` | 原始流块（token 级保真） |
| `assistant/message` | `{turn, step, message, usage?}` | 组装后的完整助手消息 |
| `tool/call` | `{turn, step, callId, name, arguments}` | 工具调用（先记录） |
| `tool/result` | `{turn, step, message, error?, meta?}` | 工具结果（后记录） |
| `todo/write` | `{todos: TodoItem[]}` | todo 列表快照（last-write-wins） |
| `request/header` | `{header: EpochHeader, reason}` | 请求配置快照（日志只） |
| `request/context` | `RequestContext` | 路由容量元数据（仅变更时记录） |
| `session/end-seed` | `{}` | 种子边界（resume/fork/replay 的分界） |

每个事件携带：
- **`seq`**：单调递增序号（`= log.length`），连续。
- **`time`**：时间戳。
- **`type`**：判别标签。
- **`data`**：类型化载荷。

**Surface 事件**（`user/message`、`assistant/message`、`tool/result`）还携带：
- **`surfaceOp`**：`'append'` 或 `{op:'replace', start, end}`——表示是追加还是替换之前的某些事件。
- **`sourceEventSeqs`**：引用的早期事件 seq。

## 3.3 Surface 模型 —— 为什么有替换

有些操作不是"追加新消息"，而是"修正之前的消息"。例如：

- **压缩**（compaction）：把前 20 条消息压缩成一条摘要 → `assistant/message` 的 `surfaceOp` 是 `{op:'replace', start: 5, end: 25}`。
- **工具结果覆盖**：某些工具结果可能被后续操作修正。

**Surface 是只读活投影**：`SessionSurface` 把事件流投影成一组节点（`nodes`），每个节点知道自己是 append 还是 replace。UI 用 surface 渲染聊天界面；模型历史用 `deriveMessages()` 投影。

```
事件流 (append-only):
  seq 1: user/message (append)
  seq 2: assistant/chunk (append)
  seq 3: assistant/chunk (append)
  seq 4: assistant/message (append, sourceEventSeqs: [2,3])
  seq 5: tool/call (append)
  seq 6: tool/result (append)
  seq 7: assistant/message (append)
  ...
  seq 20: assistant/message (replace, start:4, end:7)  ← 压缩

Surface 投影:
  node 1: user/message (seq 1)
  node 2: assistant/message (seq 20, replaces seq 4-7)
  node 3: user/message (seq 8)
  ...
```

## 3.4 deriveMessages() —— 从日志派生模型历史

`Session.deriveMessages()` 把事件流投影为模型所见的 `Message[]`。这是 agent 循环每次 step 重新加载历史的方式。

**投影规则**：

| 事件 | 投影为 |
|---|---|
| `user/message` (`source: 'user'`) | user-role 消息 |
| `user/message` (`source != 'user'`) | user-role 消息（注入上下文，原样保留） |
| `assistant/message` | assistant-role 消息 |
| `tool/result` | 带 `tool-result` 块的 user-role 消息 |
| `assistant/chunk` | **不投影**（原始流块，只用于重放保真） |
| `turn/*`、`step/*` | **不投影**（结构性事件） |

**关键点**：

- **`assistant/chunk` 不进派生历史**。它是 token 级保真记录，用于 UI 重放和调试。派生历史用 `assistant/message`。
- **空内容的 `assistant/message` 不进派生历史**。但事件保留——为了保存 usage 和 provider/model 信息。
- **`tool/result` 变成 user-role 消息**。这是因为大多数 LLM API 把工具结果放在 user role 里。
- **投影是缓存的**。每个 surface 节点首次见到时投影一次，之后缓存。surface 重写则重建缓存。

## 3.5 Session 存储 API

`ctx.sessions`（`SessionStore`）是会话存储的服务接口：

```ts
interface SessionStore {
  create(options): Session              // 创建新会话
  get(id): Session | undefined          // 获取会话
  list(): Session[]                     // 列出所有会话
  fork(source, boundary?, childId?): Session  // 分叉
  prepare(id): SessionPreparation       // 预留（恢复用）
  flush(): Promise<void>                // 持久化检查点
}
```

**`fork`** 是一个重要操作：从源会话的某个边界创建子会话。子会话继承父会话到边界点的历史，之后独立发展。用于"从某个点重新开始"或"尝试不同方向"。

**`flush`** 是持久化检查点：通知持久化插件"现在是一个安全的持久化点"。`session/flush` 是 awaited parallel 事件——持久化插件监听它，在回调中把内存中的事件写入磁盘。

## 3.6 session/event —— 事件流广播

`session/event` 是一个 emit 事件——每次 `Session.append()` 后 fire-and-forget 广播。

```
Session.append(event)
  │
  ├── 同步：追加到内存日志 + 通知监听器
  │     └── session/event (emit, fire-and-forget)
  │           └── UI/SDK 实时更新
  │
  └── 异步：持久化插件复制到磁盘
        └── session/flush (awaited parallel, 检查点)
```

**关键设计**：

- **`append()` 是同步的**——事件立刻进入内存日志，不阻塞在 I/O 上。
- **`session/event` 是 fire-and-forget**——通知监听器但不等待。UI/SDK 通过它实时更新。
- **持久化是异步的**——持久化插件把事件复制进 per-session 控制器，批量写入磁盘。不阻塞生产者。

## 3.7 turn/step 的不变量

turn 和 step 有严格的不变量，由运行时断言守护：

- **turn 编号和 step 编号单调递增**，从 1 开始。
- **step 属于 turn**——step 编号在 turn 内重置。
- **turn 必须有 `turn/end`**——如果崩溃导致 turn 未关闭，恢复时用合成 `turn/end {reason:{kind:'interrupted'}}` 关闭。
- **`interrupted` 是唯一非循环发出的 `TurnEndReason`**——它只在崩溃恢复时出现。

**`TurnEndReason` 变体**：

```
completed    —— 模型正常完成
aborted      —— 被取消
blocked      —— 被阻塞
error        —— 结构化 LlmFailure
max-tokens   —— 任何 step 触顶
interrupted  —— 崩溃恢复合成
```

## 3.8 Model-visible ⟺ Logged

dsh 有一条铁律：

> **模型可见的任何东西都必须能从会话日志重建。**

这意味着：

- 如果你想给模型注入新的上下文，不能直接改 LLM 请求——必须通过 `agent.inject()`，它会产生 `user/message` 事件（`source` 非 `'user'`），成为日志的一部分。
- 如果你替换了模型的请求配置，必须通过 `agent/request` waterfall，它会记录 `request/header` 事件。
- 运行时断言会检查：如果 LLM 请求中的任何内容无法从日志重建，断言失败。

**为什么**：可重放性。如果模型行为异常，你需要能从日志精确重建"模型当时看到了什么"。如果有些东西没记录，重建就不完整，调试就无从下手。

## 3.9 一个完整的事件流示例

用户发 `"读 README.md"` → 模型调用 `read` 工具 → 模型总结：

```
seq  type               data                                    surface
───────────────────────────────────────────────────────────────────────
1    session/end-seed   {}                                      —
2    turn/start         {turn: 1}                               —
3    user/message       {id, role:'user', content:[text]}       append
4    step/start         {turn:1, step:1}                        —
5    request/header     {header: EpochHeader, reason}           —
6    assistant/chunk    {turn:1, step:1, chunk: block-start}   —
7    assistant/chunk    {turn:1, step:1, chunk: text-delta}    —
8    assistant/chunk    {turn:1, step:1, chunk: text-delta}    —
9    assistant/chunk    {turn:1, step:1, chunk: tool-call-delta}—
10   assistant/chunk    {turn:1, step:1, chunk: block-end}     —
11   assistant/chunk    {turn:1, step:1, chunk: usage}         —
12   assistant/chunk    {turn:1, step:1, chunk: finish}        —
13   assistant/message  {turn:1, step:1, message, usage}       append, src:[6-12]
14   tool/call          {turn:1, step:1, callId, name:'read'}  —
15   step/end           {turn:1, step:1}                        —
16   step/start         {turn:1, step:2}                        —
17   tool/result        {turn:1, step:2, message}              append
18   assistant/chunk    {turn:1, step:2, ...}                  —
19   assistant/chunk    {turn:1, step:2, ...}                  —
20   assistant/message  {turn:1, step:2, message, usage}       append
21   step/end           {turn:1, step:2}                        —
22   turn/end           {turn:1, reason:{kind:'completed'}}    —
```

**派生历史**（`deriveMessages()` 的输出）：

```
[
  { role: 'user',      content: [text:"读 README.md"] },          // seq 3
  { role: 'assistant', content: [text:"让我读一下", tool-call:...] }, // seq 13
  { role: 'user',      content: [tool-result:{...}] },            // seq 17
  { role: 'assistant', content: [text:"README.md 的内容是..."] },  // seq 20
]
```

注意：`assistant/chunk`（seq 6-12, 18-19）不进派生历史；`turn/*`、`step/*`、`request/header` 也不进。

## 深入追问

- 为什么 `assistant/chunk` 和 `assistant/message` 都要存？（提示：重放保真 vs 派生效率）
- Surface 的 `replace` 操作会不会破坏 append-only 不变量？（提示：replace 是投影层概念，日志本身不改）
- 如果一个 turn 很大（几百个 step），`deriveMessages()` 会不会很慢？（提示：缓存 + surface 节点级投影）
- 压缩（compaction）如何与 surface replace 配合？（提示：压缩产生 replace 操作，把旧消息替换成摘要）

下一篇 → [04 · LLM 流式与 Provider 适配](./04-llm-streaming.md)
