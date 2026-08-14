# 02 · 一次 Turn 的旅程

这一篇是**整个材料的骨架**。它把一条用户输入从进入 agent，到 LLM 回答、调用工具、再回到用户屏幕的完整时序串起来。后续每一篇都是这条时序上某一段的放大。

读完这一篇，你应该能回答：

1. 用户发消息之后，dsh 干了什么？
2. 一次 turn 在什么时候才算"被模型看见"？
3. 工具调用为什么会"在 LLM 回答之后、下一次 LLM 调用之前"发生？
4. 为什么中途插入的"steer"消息能影响 LLM，但不会打断当前 step？

## 2.1 核心概念：turn 与 step

dsh 的 agent 循环有两个层次：

- **turn（轮次）**：一次完整的模型交互循环——从用户输入到模型不再需要工具。一个 turn 包含零或多个 step。
- **step（步骤）**：一次模型请求 + 它调用的工具执行。每个 step 只调用一次 LLM。

```
turn 开始
  │
  ├── step 1: 请求 LLM → LLM 回答 + 工具调用 → 执行工具 → 工具结果
  │     ↓ LLM 还需要继续
  ├── step 2: 重新加载历史 → 请求 LLM → LLM 回答 + 工具调用 → 执行工具
  │     ↓ LLM 不再需要工具
  └── turn 结束
```

**关键不变量**（来自 [architecture.md](../architecture.md#turn-flow)）：

- **每个 step 只有一次 `llm.stream(request)`**。工具循环通过"重新加载历史 + 跑下一个 step"实现，不是在 LLM 调用内部循环。
- **turn 和 step 都是 durable 事件**——它们写入会话日志，崩溃后可恢复。
- **输入通过一个 inbox 到达驱动器**。有些消息立即唤醒；注入的上下文等在 inbox 里直到另一条消息唤醒。

## 2.2 时序总览

```
┌─────────┐                  ┌───────────┐                ┌──────────────┐
│ Client  │ ──followup()──▶  │  Agent    │ ──append()──▶  │  Session     │
│ (Web/   │                  │  Handle   │                │  (事件日志)   │
│  CLI)   │                  └───────────┘                └──────────────┘
└────┬────┘                       │                              │
     │ agent/* events              │ wake driver                  │ session/event
     │ (live, 非持久)               ▼                              │ (durable, 持久)
     │                   ┌──────────────────────┐                  │
     │                   │   Agent Loop (driver)│                  │
     │                   │                      │                  │
     │                   │  1. claim inbox      │                  │
     │                   │     → turn/start     │──────────────────┘
     │                   │  2. agent/pre-step   │
     │                   │     (waterfall)      │
     │                   │  3. step/start       │──────────────────┐
     │                   │     + user/message   │                  │
     │                   │  4. 组装 prompt      │                  │
     │                   │     + tool schemas   │                  │
     │                   │  5. agent/request    │                  │
     │                   │     (waterfall)      │                  │
     │                   │  6. llm/stream       │                  │
     │                   │     (waterfall)      │                  │
     │                   │  7. 流式持久化        │                  │
     │                   │     assistant/chunk* │──────────────────┤
     │                   │  8. assistant/message│                  │
     │                   │  9. 工具调用          │                  │
     │                   │     tool/call*       │──────────────────┤
     │                   │     tools/pre-execute│                  │
     │                   │     tools/execute    │                  │
     │                   │     tools/post-execute│                 │
     │                   │     tool/result*     │──────────────────┤
     │                   │ 10. step/end         │                  │
     │                   │ 11. agent/turn-stopping│                │
     │                   │ 12. 若需继续 → goto 1 │                 │
     │                   │ 13. turn/end         │──────────────────┘
     │                   └──────────────────────┘
     │
     ◀── agent/status (idle/running) + session/event (durable) ──┘
```

**几个关键设计选择**，会在后续章节展开：

- **durable 事件 vs live 事件**：`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 是 durable 事件（写入会话日志）；`agent/*`、`tools/*`、`llm/stream` 是 live 事件（只用于实时控制/拦截，不持久化）。
- **inbox 是 durable 投影**：用户输入先进入 agent 的 inbox（两个有序列表：`next-turn` 和 `next-step`），驱动器在 step boundary claim 它们。
- **pre-step 是 waterfall**：它决定模型看到什么——可以 reject（不开 step）、可以改写消息、可以原样通过。
- **工具先记录、后执行**：`tool/call` 在执行前就写入日志，这样即使执行中崩溃，恢复时知道有过这次调用。

## 2.3 阶段一：用户输入到 inbox

**入口**：用户（或 SDK、或 UI）调用 `agent.followup(content)` 或 `agent.steer(content)` 或 `agent.inject(content)`。

这三个方法的区别是 dsh 输入路由的核心：

| 方法 | target | 是否唤醒 | 语义 |
|---|---|---|---|
| `followup(msg)` | `next-turn` | 是 | 排一个普通轮次，成为自己 turn 的唯一普通消息 |
| `steer(msg)` | `next-step` | 是 | 为最近的 step 提供引导；运行中的驱动器在下一个 step boundary 消费；空闲时启动 turn |
| `inject(msg)` | `next-step` | 否 | 为下一个 pre-step 排模型可见上下文，不唤醒；运行中的驱动器在最近的 step boundary 消费 |

```ts
// packages/core/agent/src/types.ts （概念精简）
interface Agent {
  /** 路由输入到 inbox boundary 并可选唤醒驱动器 */
  send(message: UserMessage, target: InboxTarget, wakeup: boolean): void
  /** 排一个普通轮次并唤醒 */
  followup(message: UserMessage): void
  /** 为最近的 step 提交引导 */
  steer(message: UserMessage): void
  /** 为下一个 pre-step 排上下文，不唤醒 */
  inject(message: UserMessage): void
}
```

**inbox 的两个列表**：

```
Inbox
├── next-turn:  [ msg_A ]    ← followup 的消息排这里，一个 turn 只提一条
└── next-step:  [ msg_B, msg_C ]  ← steer/inject 的消息排这里，一个 step 全提
```

**claim 规则**（在 step boundary）：
- **next-step 全提**：所有 `next-step` 消息作为当前 step 的额外输入。
- **next-turn 一次一条**：在 turn boundary（turn 开始时）提一条 `next-turn` 消息。
- **rejected 的 claim 不回退**：如果 `agent/pre-step` reject 了，claimed 消息既不 discard 也不重新发为 `user/message`，turn 无 step 关闭。

## 2.4 阶段二：驱动器唤醒与 turn 开始

用户调用 `followup()` 后，驱动器被唤醒：

1. **`agent/status` → `running`**：同步状态切换，表示驱动器开始工作。
2. **`turn/start`**：在会话日志上开启一个新 turn。
3. **claim inbox**：声明待处理的 `next-step` 输入 + 一条 `next-turn` 消息。发 `agent/inbox/claimed`。
4. **`agent/pre-step` waterfall**：把 claimed 消息交给 pre-step 监听器。

```ts
// pre-step 的决策类型
type PreStepDecision =
  | { kind: 'reject' }                      // 拒绝：不开 step
  | { kind: 'enter'; messages: UserMessage[] }  // 进入：用这些消息（可改写）
```

**pre-step 是唯一的串行 listener chain**——在请求派发之前运行。Hook 桥（如 Claude Code hooks）在这里做决策：拒绝危险操作、改写消息内容、注入额外上下文。

**如果 reject**：已 claim 的消息保持移除状态（不回 inbox），turn 无 step 关闭，日志记录这次尝试。这意味着"用户发了消息但被 hook 拒绝"是一个 durable 事实。

## 2.5 阶段三：step 执行 —— 从请求到回答

pre-step 返回 `enter` 后，驱动器进入一个 step：

1. **`step/start`**：在日志上开启新 step。
2. **`user/message`**：把 entered 的消息逐条写入日志（成为 model-visible history）。
3. **组装 prompt**：`ctx.systemPrompt.assemble()` waterfall 组装提示词段落 + 工具 schema。
4. **`agent/request` waterfall**：冻结调用配置（provider、model、maxTokens 等）。
5. **`llm/stream` waterfall**：调用 LLM 适配器，返回 `StreamChunk` 流。
6. **流式持久化**：每个 `StreamChunk` 写入 `assistant/chunk` 事件（token 级重放保真）。
7. **`assistant/message`**：流结束后，组装完整的 `AssistantMessage` 写入日志（派生历史用此事件）。

```
step/start
  │
  ├── user/message (每条 entered 消息)
  │
  ├── systemPrompt.assemble()
  │     ├── 提示词段落 (persona, harness:source, 工作区指令, ...)
  │     └── 工具 schema (所有注册工具的 JSON Schema)
  │
  ├── agent/request (waterfall)
  │     └── 冻结 LlmCallConfig { provider, model, maxTokens, ... }
  │
  ├── llm/stream (waterfall)
  │     └── StreamChunk*
  │           ├── block-start / text-delta / reasoning-delta
  │           ├── tool-call-delta
  │           ├── block-end
  │           ├── usage
  │           └── finish { reason, replayState? }
  │
  ├── assistant/chunk* (持久化每个 chunk)
  │
  └── assistant/message (持久化完整消息 + usage)
```

**关键点**：

- **请求配置从日志 header 重建**，不是每次调用重新构建。`request/header` 事件记录了 call config + 适配器默认 + 渲染提示 + 工具 schema，使请求可从会话日志完整重建。
- **`assistant/chunk` 保原始流块**，`assistant/message` 保组装后的消息。派生历史（`deriveMessages()`）用 `assistant/message`；UI 重放用 `assistant/chunk`。
- **空内容也记录 `assistant/message`**——为了保存 usage 和 provider/model 信息，但不进派生历史。

## 2.6 阶段四：工具调用的 settlement

如果 LLM 回答中包含 `tool-call` 块，工具执行流程启动：

1. **`tool/call` 持久化**：每个工具调用在执行前就写入日志（**先记录、后执行**）。
2. **`tools/pre-execute` waterfall**：检查权限。返回 `allow` / `deny` / `ask`。`ask` 经 `ctx.approval` 一次性提示用户。
3. **单调守卫**：注册的 `ToolGuard` 只能减权限（deny 或 abstain），不能加权限。
4. **`tools/execute` waterfall**：超时、重试、metrics 包装 → 调用工具的 `execute()` 体。
5. **`tools/post-execute` waterfall**：accept / block / replace / add context。
6. **`finalizeContent`**：最后一英里内容变换，对每个规范化结果调用一次。
7. **`tool/result` 持久化**：单一面向模型的结果写入日志。

```
LLM 输出 tool-call 块
  │
  ├── tool/call (持久化：先记录)
  │
  ├── tools/pre-execute (waterfall: allow/deny/ask)
  │     └── ask → ctx.approval (一次性用户审批)
  │
  ├── ToolGuard (单调守卫：只能减权限)
  │
  ├── tools/execute (waterfall: around-dispatch)
  │     ├── timeout-policy (超时截止)
  │     └── tool.execute(args) → 结构化 result
  │
  ├── tools/post-execute (waterfall: accept/block/replace)
  │
  ├── finalizeContent (内容不变量)
  │
  └── tool/result (持久化：面向模型的结果)
```

**工具执行模式**：

- `parallel`：可与兄弟工具重叠执行（如多个 `read`）。
- `exclusive`：独占屏障，必须等前一个完成（如 `write`）。

循环根据 `isConcurrencySafe(args)` 分类工具，形成屏障与滚动池。

**工具结果如何回到模型**：`tool/result` 事件写入日志后，下一个 step 的 `deriveMessages()` 会把它投影为带 `tool-result` 块的 user-role 消息。模型在下一个 step 看到工具结果，决定是否继续调用工具。

## 2.7 阶段五：turn 结束

step 结束后，驱动器检查是否需要继续：

1. **`step/end`**：在日志上关闭当前 step。
2. **检查续行条件**：
   - 工具是否还需要另一个请求？（tools owe another request）
   - 是否有新的 `next-step` 输入到达？（racing steering）
3. **如果需要继续**：claim 新输入 → pre-step → 下一个 step。
4. **如果不需要继续**：`agent/turn-stopping` serial 事件。
   - 监听器可以 object（调 `agent.steer()` 注入新引导）→ 机器重读 inbox → 有新引导就跑另一个 step。
   - 没有新引导 → 关闭 turn。
5. **`turn/end`**：在日志上关闭 turn，记录 `TurnEndReason`。
6. **`agent/status` → `idle`**：驱动器回到空闲。

`TurnEndReason` 是 merge-extensible 的判别联合：

```
completed   —— 模型正常完成，不再需要工具
aborted     —— 被取消（user/parent/hook/disposed）
blocked     —— 被阻塞（如审批拒绝）
error       —— 结构化 LlmFailure
max-tokens  —— 任何 step 触顶
interrupted —— 崩溃恢复时合成的关闭（唯一非循环发出的原因）
```

## 2.8 把旅程画成一张图

```
agent.followup(msg)
   │
   ▼
Inbox.append(next-turn, msg) ──► agent/inbox/inserted
   │
   │ wake driver
   ▼
agent/status: running
   │
   ▼
turn/start ──► Session 日志
   │
   ▼
claim inbox (next-step + 1× next-turn)
   │
   ▼
agent/pre-step (waterfall)
   │
   ├── reject → turn/end (无 step) → agent/status: idle
   │
   └── enter(messages)
        │
        ▼
step/start ──► Session 日志
   │
   ├── user/message* ──► Session 日志
   │
   ├── systemPrompt.assemble()
   ├── agent/request (waterfall)
   ├── llm/stream (waterfall)
   │      │
   │      └── StreamChunk* → assistant/chunk* ──► Session 日志
   │                                    → session/event → UI/SDK
   │
   ├── assistant/message ──► Session 日志
   │
   ├── tool/call* ──► Session 日志 (先记录)
   │
   ├── tools/pre-execute → tools/execute → tools/post-execute
   │      │
   │      └── tool/result* ──► Session 日志
   │
   ├── step/end ──► Session 日志
   │
   ├── 需要继续? ──是──► claim → pre-step → 下一个 step
   │
   └── 否
        │
        ▼
agent/turn-stopping (serial)
   │
   ├── 有新 steer? ──是──► 下一个 step
   │
   └── 否
        │
        ▼
turn/end ──► Session 日志
   │
   ▼
agent/status: idle
```

## 2.9 具体示例：一次真实 trace

用户在 web 界面发了 `"读一下 README.md 然后总结"` 之后，实际触发的事件序列（简化）：

```
时间线                    事件                                  说明
─────────────────────────────────────────────────────────────────────
T+0ms     agent.followup(msg)           用户发消息
T+1ms     agent/inbox/inserted          消息进入 next-turn 列表
T+2ms     agent/status: running         驱动器唤醒
T+3ms     turn/start                    日志记录 turn 开始
T+4ms     agent/inbox/claimed           claim 消息
T+5ms     agent/pre-step → enter        pre-step 通过
T+6ms     step/start                    日志记录 step 1 开始
T+7ms     user/message                  用户消息写入日志
T+10ms    systemPrompt.assemble         组装提示词 + 工具 schema
T+12ms    agent/request                 冻结调用配置
T+15ms    llm/stream                    ← 唯一一次 LLM 调用
T+100ms     → assistant/chunk (text)    流式输出文本
T+200ms     → assistant/chunk (text)    ...
T+300ms     → assistant/chunk (tool-call)  模型决定读文件
T+301ms    assistant/message            完整消息写入日志
T+302ms    tool/call (read, ...)        先记录工具调用
T+310ms    tools/pre-execute → allow    权限检查通过
T+320ms    tools/execute                执行 read
T+330ms    tool/result                  工具结果写入日志
T+331ms    step/end                     step 1 结束
T+332ms    需要继续? → 是               工具调用了，模型需要看结果
T+333ms    claim → pre-step → enter     下一个 step
T+334ms    step/start                   step 2 开始
T+340ms    deriveMessages()             从日志重新派生历史（含工具结果）
T+345ms    llm/stream                   ← 第二次 LLM 调用
T+500ms      → assistant/chunk (text)   模型基于文件内容总结
T+800ms      → finish                   模型回答完毕
T+801ms    assistant/message            完整消息写入日志
T+802ms    step/end                     step 2 结束
T+803ms    需要继续? → 否               不需要工具
T+804ms    agent/turn-stopping          检查是否有新 steer → 无
T+805ms    turn/end (completed)         turn 结束
T+806ms    agent/status: idle           回到空闲
```

**对照上面这个 trace**，再看 §2.2 的时序图和 §2.3-2.7 的阶段描述，每个步骤都能对上号。

## 2.10 一句话总结

> **dsh 把"用户输入"和"模型执行"通过 inbox 解耦**。inbox 是 durable 投影——输入一旦进入就不会丢。驱动器是临时编排——它崩溃了不会"恢复驱动器"，但 inbox 和日志里的数据都在，下次唤醒重新跑。turn 和 step 是 durable 事件——它们记录在日志里，崩溃后用合成 `interrupted` 关闭孤儿 turn。

## 深入追问

- 如果 `agent.followup()` 之后进程立刻崩溃，会发生什么？（提示：消息在 inbox 里，但 inbox 是 durable 投影……）
- 为什么 `agent/pre-step` reject 后不把消息放回 inbox？（提示：durable 事实 + 避免无限循环）
- `steer` 和 `inject` 都是 `next-step` target，区别只在是否唤醒——为什么这个区别重要？（提示：inject 是"顺便带上"，steer 是"现在就处理"）
- 工具的"先记录、后执行"如何帮助崩溃恢复？（提示：恢复时看到 tool/call 但没有 tool/result，知道执行被中断）

下一篇 → [03 · 会话日志与事件溯源](./03-session-log.md)
