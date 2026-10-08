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
| `system/message` | `{turn, step, message: SystemMessage}` | 渲染后的系统提示（surface 节点 0） |
| `developer/message` | `{turn, step, message, headerSeq?}` | 增量工具声明变更（additions / removals） |
| `user/message` | `UserMessage` | 用户消息（直接 prompt、inject 上下文、目标续行） |
| `assistant/message` | `{turn, step, message, stream, usage?, interrupted?}` | 组装后的完整助手消息，内嵌产生它的精确计时流记录 |
| `assistant/attempt` | `{turn, step, stream}` | 未提交任何模型可见消息的尝试（失败/重试/取消/流错误），log-only |
| `tool/call` | `{turn, step, callId, name, arguments}` | 工具调用（先记录） |
| `tool/result` | `{turn, step, message, error?, meta?}` | 工具结果（后记录） |
| `todo/write` | `{todos}` | todo 列表快照（由 `dsh-tool-todo` 通过 declaration merging 声明） |
| `request/header` | `{header: EpochHeader, reason, startsSeries?}` | 请求配置快照（日志只） |
| `request/context` | `RequestContext` | 路由容量元数据（仅变更时记录） |
| `session/end-seed` | `{inherited?}` | 种子边界（resume/fork/replay 的分界） |

每个事件携带：
- **`seq`**：单调递增序号（`= log.length`），连续。
- **`time`**：时间戳。
- **`type`**：判别标签。
- **`data`**：类型化载荷。
- **`ignorable?`**：缺省表示"读到不认识的类型必须拒绝重建"；只有纯信息性记录才标 `true`。

**Surface 事件**（`system/message`、`developer/message`、`user/message`、`assistant/message`、`tool/result` 五种产生模型消息的事件）还携带：
- **`surfaceOp`**：`'append'` 或 `{op:'replace', startSeq, endSeq}`——表示是追加还是替换之前的某些事件。
- **`sourceEventSeqs`**：引用的早期事件 seq。

非 surface 事件（边界标记、attempt、错误）永不携带 surface 元数据——编译器在 `Session.append()` 调用点强制这一点。

## 3.3 Surface 模型 —— 为什么有替换

有些操作不是"追加新消息"，而是"修正之前的消息"。例如：

- **压缩**（compaction）：把前 20 条消息压缩成一条摘要 → 摘要消息的 `surfaceOp` 是 `{op:'replace', startSeq: 5, endSeq: 25}`。
- **系统提示归一化**：能力不足的路由把非空提示文本合并到首个 system 节点，后续非空 system 节点用日志化的空替换清空。

**Surface 是只读活投影**：`SessionSurface` 把事件流投影成一组有序节点（`nodes`），每个节点知道自己是 append 还是 replace。UI 用 surface 渲染聊天界面；模型历史用 `deriveMessages()` 投影。

```
事件流 (append-only):
  seq 1: system/message (append)          ← surface 节点 0
  seq 2: user/message (append)
  seq 3: assistant/message (append)
  seq 4: tool/call (append)
  seq 5: tool/result (append)
  seq 6: assistant/message (append)
  ...
  seq 20: assistant/message (replace, startSeq:3, endSeq:6)  ← 压缩

Surface 投影:
  node 0: system/message (seq 1)
  node 1: user/message (seq 2)
  node 2: assistant/message (seq 20, replaces seq 3-6)
  node 3: user/message (seq 7)
  ...
```

## 3.4 deriveMessages() —— 从日志派生模型历史

`Session.deriveMessages()` 把 surface 节点投影为模型所见的 `Message[]`。这是 agent 循环每次 step 重新加载历史的方式。

**投影规则**：

| 事件 | 投影为 |
|---|---|
| `system/message` | system-role 消息（空内容投影为无消息） |
| `developer/message` | developer-role 消息（增量工具声明） |
| `user/message` | user-role 消息（人类 prompt 与注入上下文靠 `source` 区分） |
| `assistant/message` | assistant-role 消息 |
| `tool/result` | tool-role 消息（`ToolResultMessage`，带 `toolCallId`） |
| `assistant/attempt` | **不投影**（未提交的尝试，只用于保真与诊断） |
| `turn/*`、`step/*`、`request/*` | **不投影**（结构性事件） |

**关键点**：

- **空内容的 `assistant/message` 不进派生历史**（如只承载 usage 的 max-tokens step）。但事件保留——为了保存 usage 和 provider/model 信息。
- **投影是增量的**。`deriveMessages()` 只处理 surface 新增节点，已投影前缀按 surface 的 content generation 缓存；surface 重写（replace）才重建。
- **插件可以注册 pure message projections**（见 [subsystems/session.md](../subsystems/session.md#plugin-owned-message-projections)）在派生时改写消息内容——这是"模型可见 ⟺ 已记录"的扩展通道。

除 `deriveMessages()` 外，**durable 折叠状态**走另一个 seam：`dsh-session-projection` 的 `ctx.sessionProjections`。注册的投影单元增量折叠已提交事件，宿主消费者用 `stateOf()` 读一份类型化状态（如 turn 边界、token 计量、todo 列表）。

## 3.5 Session 存储 API

`ctx.sessions`（`SessionStore`）是会话存储的服务接口：

```ts ignore-check
interface SessionStore {
  create(id?, options?): Session          // 创建新会话
  get(id): Session | undefined            // 获取会话
  list(): Session[]                       // 列出所有会话
  fork(source, boundary?, childId?): Session  // 分叉
  prepare(id?, options?): Session         // 预留（恢复用）
  flush(session): Promise<boolean>        // 持久化检查点（awaited session/flush）
}
```

**`fork`** 是一个重要操作：从源会话的某个边界创建子会话。子会话继承父会话到边界点的历史，之后独立发展。用于"从某个点重新开始"或"尝试不同方向"。fork 血缘记录在 `SessionHeader.isSeeded` 上，精确的继承前缀长度随存储元数据走，不进可重放的事件日志。

**`flush`** 是持久化检查点：`session/flush` 是 awaited 事件——持久化插件监听它，把缓冲中的事件写入磁盘。turn 边界不等待 flush；语义化的 durability 检查点（模型请求、顶层工具派发、step 完成）由 `dsh-session-checkpoint-policy` 决定。

## 3.6 session/event —— 事件流广播

`session/event` 是一个 emit 事件——每次 `Session.append()` 后 fire-and-forget 广播。

```
Session.append(event)
  │
  ├── 同步：追加到内存日志 + 通知监听器
  │     └── session/event (emit, fire-and-forget)
  │           └── UI/SDK 实时更新
  │
  └── 异步：持久化插件缓冲事件，批量写入磁盘
        └── session/flush (awaited, 检查点)
```

**关键设计**：

- **`append()` 是同步的**——事件立刻进入内存日志，不阻塞在 I/O 上。
- **`session/event` 是 fire-and-forget**——通知监听器但不等待。UI/SDK 通过它实时更新。
- **持久化是异步的**——持久化插件把事件深拷贝进 per-session 写缓冲，在有界批处理窗口后写入磁盘。不阻塞生产者。

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
seq  type               data                                     surface
───────────────────────────────────────────────────────────────────────
1    turn/start         {turn: 1}                                —
2    step/start         {turn:1, step:1}                         —
3    system/message     {turn:1, step:1, message}                append (节点 0)
4    user/message       {id, role:'user', content:[text]}        append
5    request/header     {header: EpochHeader, reason}            —
6    assistant/message  {turn:1, step:1, message, stream, usage} append
7    tool/call          {turn:1, step:1, callId, name:'read'}    —
8    tool/result        {turn:1, step:1, message(role:'tool')}   append
9    step/end           {turn:1, step:1}                         —
10   step/start         {turn:1, step:2}                         —
11   assistant/message  {turn:1, step:2, message, stream, usage} append
12   step/end           {turn:1, step:2}                         —
13   turn/end           {turn:1, reason:{kind:'completed'}}      —
```

**派生历史**（`deriveMessages()` 的输出）：

```
[
  { role: 'system',    content: [text:"..."] },                    // seq 3
  { role: 'user',      content: [text:"读 README.md"] },           // seq 4
  { role: 'assistant', content: [text:"让我读一下", tool-call:...] }, // seq 6
  { role: 'tool',      toolCallId, content: [...] },               // seq 8
  { role: 'assistant', content: [text:"README.md 的内容是..."] },   // seq 11
]
```

注意：`turn/*`、`step/*`、`request/header` 不进派生历史；如果有失败或被重试的尝试，它们以 `assistant/attempt` 留在日志里，同样不进派生历史。

## 深入追问

- 为什么流记录内嵌在 `assistant/message` 里，而不是每个 chunk 一个事件？未提交的尝试去哪了？（提示：durable 结算 vs live 增量；`assistant/attempt` + `agent/assistant-stream`）
- Surface 的 `replace` 操作会不会破坏 append-only 不变量？（提示：replace 是投影层概念，日志本身不改）
- 如果一个 turn 很大（几百个 step），`deriveMessages()` 会不会很慢？（提示：增量投影 + surface content generation 缓存）
- 压缩（compaction）如何与 surface replace 配合？（提示：压缩产生 replace 操作，把旧消息替换成摘要）
- `ctx.sessionProjections` 和 `deriveMessages()` 各负责什么？（提示：durable 折叠状态 vs 模型消息历史）

下一篇 → [04 · LLM 流式与 Provider 适配](./04-llm-streaming.md)
