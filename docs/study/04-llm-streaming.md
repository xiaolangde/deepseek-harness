← 上一篇：[03 · 会话日志与事件溯源](./03-session-log.md)

# 04 · LLM 流式与 Provider 适配

这一篇放大 turn 旅程中的 LLM 调用环节——从 `llm/stream` waterfall 到 `StreamChunk` 流再到 provider 适配器。

dsh 的 LLM 子系统不绑定任何具体模型供应商。它定义了一套消息模型、流式词汇和适配器 seam，任何遵守契约的 provider 都可以注册。

## 4.1 ContentBlock —— 模型输出的原子单位

模型消息的内容由若干 `ContentBlock` 组成（[packages/llm/llm/src/message.ts](../../packages/llm/llm/src/message.ts) 与 [types.ts](../../packages/llm/llm/src/types.ts)）。`ContentBlockMap` 是 merge-extensible 的——插件可以添加新块类型，不改 `dsh-llm` 包。

当前核心块类型：

| `type` | 类型 | 内容 |
|---|---|---|
| `text` | `TextBlock` | 文本内容 |
| `reasoning` | `ReasoningBlock` | 推理过程（thinking） |
| `image` | `ImageBlock` | 图像（持久附件引用） |
| `file` | `FileBlock` | 文件附件引用 |
| `tool-call` | `ToolCallBlock` | 工具调用（`id: CallId`、`name`、`arguments: string`） |
| `tool-addition` | `ToolAdditionBlock` | 会话中新增的工具声明 |
| `tool-removal` | `ToolRemovalBlock` | 会话中移除的工具声明 |

**关键设计**：

- **`tool-call` 的 `arguments` 是原始 JSON 字符串**，不是解析后的对象。工具自己负责解析和验证。这避免了"适配器解析了一遍，工具又解析一遍"的重复，也保证了端到端的 JSON 原文保持不变。
- **`CallId` 是 branded 类型**——`Branded<'call'>`，不能和 `SessionId` 混淆。
- **工具结果不是内容块**——它是一等公民的 tool-role 消息（`ToolResultMessage`），见 §4.2。

## 4.2 Message —— 会话的基本单位

`MessageRoleMap` 按角色键控，是一个**闭合**映射——每个模型可见角色都必须有对应的 durable 会话事件和适配器投影：

```ts ignore-check
interface MessageBase {
  readonly id: MessageId             // 跨表示边界的稳定标识
  readonly content: readonly ContentBlock[]  // 精确的模型可见内容块
  readonly source: MessageSource     // 谁产生的这条消息
}

// role: 'system' | 'developer' | 'user' | 'assistant' | 'tool'
// system    → 渲染后的系统提示（source 固定为 system-prompt）
// developer → 增量工具声明变更
// user      → 人类 prompt 或注入上下文
// assistant → 模型产出（source: model，带 provider/model 元数据）
// tool      → ToolResultMessage（toolCallId + isError?）
```

`MessageSourceMap` 是 merge-extensible 的——每个生产者在自己的模块里声明自己的 `kind`，**没有共享的 catch-all `plugin` kind**：

| `kind` | 含义 |
|---|---|
| `user` | 人类用户直接输入 |
| `model` | 模型产出（携带 provider/model 元数据） |
| `tool` | 工具结果（携带 `callId`） |
| `system-prompt` | system-prompt 插件产出 |

**`ContextForm`** 声明内容语义形式，与生产者身份独立：

```
instructions  —— 指令
catalog       —— 目录
snapshot      —— 快照（带 sections）
notice        —— 通知（带 summary，上限 120 字符）
relay         —— 中继（另一个 agent 发来的消息）
recall        —— 回忆（取自另一个会话的日志）
```

这意味着同一条 `user` 角色消息可以是"人类 prompt"也可以是"注入的上下文"——`source` 区分它们。`deriveMessages()` 都投影为 user-role 消息，但 UI 可以根据 `source` 和 `form` 显示不同样式。

## 4.3 StreamChunk —— 流式协议

`StreamChunk` 是 LLM 流式输出的线路协议。它是一个按 `type` 判别的联合——`switch` 以 `assertNever` 结束。

| `type` | 载荷 | 说明 |
|---|---|---|
| `block-start` | `{index, blockType}` | 开始一个新块 |
| `text-delta` | `{index, text}` | 文本增量 |
| `reasoning-delta` | `{index, text}` | 推理增量 |
| `tool-call-delta` | `{index, id, name?, argumentsDelta}` | 工具调用参数增量（原始 JSON 串） |
| `block-end` | `{index, block}` | 结束一个块，带组装好的完整块 |
| `usage` | `{usage: TokenUsage}` | token 用量 |
| `finish` | `{reason: FinishReason, replayState?}` | 流结束 |

**`index`** 关联交错 delta——同一个 `index` 的 delta 属于同一个块。模型可以同时输出文本和工具调用的交错流。

**`FinishReason`**（merge-extensible）：

```
stop          —— 正常停止
tool-calls    —— 需要调用工具
max-tokens    —— 达到 max-tokens
aborted       —— 中止（携带结构化 LlmFailure）
error         —— 错误（携带结构化 LlmFailure）
```

## 4.4 BlockAssembler —— 从流到块

`BlockAssembler`（[packages/llm/llm/src/assembler.ts](../../packages/llm/llm/src/assembler.ts)）是把 `StreamChunk` 流折叠回完整结果的唯一共享实现。

```
StreamChunk* ──► BlockAssembler.push() ──► blocks() / message()
                                           + usage
                                           + finish
```

**为什么需要 assembler**：

- LLM 流是 token 级的碎片（`text-delta`、`tool-call-delta`），但 agent 循环需要完整的块（`ContentBlock`）。
- 工具调用参数在流中是分片的 JSON 串，需要拼接成完整串。
- 多个块可能交错输出（文本和工具调用交替），需要按 `index` 分组。

**assembler 是共享的**——所有 provider 适配器都用它。这保证了不同 provider 的流式输出被统一处理，agent 循环不需要关心 provider 差异。

## 4.5 LlmCallConfig —— 调用配置

```ts
import type { ReasoningEffortId } from '@deepseek-ai/dsh-llm'

interface LlmCallConfig {
  provider: string           // provider 路由
  model: string              // 模型 id
  reasoningEffort?: ReasoningEffortId  // 推理强度
  temperature?: number
  maxTokens?: number
  stop?: string[]
}
```

**关键设计**：

- **调用配置从日志 header 重建**。`request/header` 事件记录了 `EpochHeader`——包含 call config + 适配器默认标记 + 渲染提示 + 组装工具 schema。
- **循环从日志 header 构建请求**，而不是每次调用接受新配置。这保证了"同一个 turn 内的请求配置一致"。
- **`agent/request` waterfall 可以替换配置**——但不能修改消息内容（模型可见内容必须走日志渠道）。

## 4.6 LlmAdapter —— Provider 适配器 seam

provider 适配器是可替换的——这是 dsh "不绑定供应商"的核心机制。

```ts ignore-check
abstract class LlmAdapter {
  // 唯一必需方法：流式调用
  abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>
  // provider 展示元数据（按路由）
  providerInfo(provider: string): LlmProviderInfo
  // 可选：provider 作用域的重试策略
  providerRetryPolicy(provider: string): ResolvedRetryPolicy | undefined
  // 可选：列出/解析模型
  listModels(provider: string): Promise<readonly LlmModelInfo[]>
  resolveModel(provider: string, model: string, signal?): Promise<...>
}
```

**注册方式**：

```ts ignore-check
ctx.llm.registerAdapter(['deepseek', 'deepseek-reasoner'], myAdapter)
// 返回 AdapterRegistrationHandle，含原子 replace(providers) 方法
```

**`ctx.llm`（`LlmRuntime`）**是适配器注册表 + 流式调用 API：

- `registerAdapter(providers, adapter)`：注册适配器，返回 handle。
- `prepareCall(provider, model, signal?)`：解析路由与模型元数据，返回 `PreparedAdapterCall`——把"选适配器"和"派发流"绑在同一次注册代际上。
- `PreparedLlmCall`：冻结的 config + 重试策略 + `stream(options)` 派发。循环先经 `agent/request` 冻结配置，再派发绑定的 prepared call。

**`llm/stream` waterfall**：监听器可以包装、替换、观测流式调用。例如：

```ts ignore-check
ctx.on('llm/stream', (options, next) => {
  // 记录 metrics
  const stream = next()  // 委托给适配器
  // 包装流
  return wrapStream(stream)
})
```

## 4.7 适配器契约

适配器必须遵守的契约（不遵守会导致运行时错误）：

1. **`usage` 在 `finish` 之前**，`finish` 之后不再有任何内容。
2. **工具调用参数端到端保持原始 JSON 串**——适配器不解析参数，只传递。
3. **两种错误路径归一**：throw 或带内 `finish{kind:'error'|'aborted'}` 都归一为 `LlmFailure`（结构化 code/status/providerRetryAfterMs）。
4. **一次适配器调用 = 一次 provider 尝试**——重试在适配器外面。
5. **空完成是可重试错误**（`EMPTY_RESPONSE`）——正常停止但零内容块归一为失败，不产出空的 assistant 消息。
6. **上下文溢出规范码** `CONTEXT_WINDOW_EXCEEDED`；配额耗尽有 `QUOTA` / `ACCOUNT_QUOTA` 规范码。
7. **每个 provider HTTP 请求带 `attributionHeaders()`**——直连 fetch 的 DeepSeek 适配器和库支撑的 pi-ai 适配器用不同内部机制满足同一契约。

## 4.8 TokenUsage —— token 计量

```ts
interface TokenUsage {
  inputTokens: number        // 仅未缓存输入
  outputTokens: number       // 总输出（含 reasoning）
  totalTokens?: number       // provider 权威总量或由其推导；不可得则缺省
  cacheReadTokens?: number   // 缓存命中
  cacheWriteTokens?: number  // 缓存写入
  reasoningTokens?: number   // 已含于 output
}
```

**设计要点**：

- 计数**互斥**：`inputTokens` 只计未缓存输入，缓存命中单独计——计费输入 = 三者之和。把缓存折进总量的 provider（DeepSeek 的 `prompt_tokens`）由适配器减出来。
- `reasoningTokens` 已含于 `outputTokens`——不是额外加的。
- token 计量是**重放感知**的——`ctx.tokenMeter` 在重放时不会重复计数。

## 4.9 重试策略

`dsh-llm-retry` 插件监听 `agent/request-error` waterfall，提供 provider 作用域的重试策略：

```
agent/request-error (waterfall)
  │
  ├── dsh-llm-retry 检查 failure:
  │     ├── EMPTY_RESPONSE → retry (最多 N 次)
  │     ├── CONTEXT_WINDOW_EXCEEDED → 不 retry（需要压缩）
  │     ├── 429 Too Many Requests → retry + providerRetryAfterMs
  │     ├── 5xx → retry with backoff
  │     └── 其他 → 不 retry
  │
  └── 返回 { kind: 'retry' } 或 undefined（保留原错误）
```

**关键点**：

- 重试策略是**per-provider** 的——`prepareCall()` 把当时注册代际的策略一并冻结。
- `agent/request-error` 在**失败的 step 关闭后、turn 关闭前**运行——listener 可以修复 durable 状态或等待策略工作。
- 返回 `{ kind: 'retry' }` **不调 `next()`**——这是 short-circuit 语义。

## 4.10 现有 Provider 适配器

| 包 | Provider | 说明 |
|---|---|---|
| `dsh-llm-deepseek` | DeepSeek | 直连 DeepSeek wire 协议（凭据由 `llm-deepseek-account` / `llm-deepseek-api-key` 提供） |
| `dsh-llm-pi-ai` | pi-ai | 库支撑的多 provider 适配器 |
| `dsh-llm-replay` | replay | 重放录制响应（测试支撑，`packages/test-support/llm-replay`） |

**添加新 provider** 的步骤见 [cookbook/adding-an-llm-adapter.md](../cookbook/adding-an-llm-adapter.md)。

## 深入追问

- 为什么 `tool-call` 的 `arguments` 是字符串而不是解析后的对象？（提示：端到端 JSON 原文 + 工具自校验）
- `BlockAssembler` 为什么是共享的，而不是每个适配器自己实现？（提示：统一处理 + 减少重复）
- 如果 provider 返回的流中途断开（网络错误），assembler 的状态怎么处理？（提示：throw 归一为 `LlmFailure`）
- `reasoningEffort` 如何影响请求？（提示：`prepareCall` 解析 exact-model 元数据，适配器解释）
- 工具结果为什么是 tool-role 消息而不是 user 消息里的一个块？（提示：`MessageRoleMap` 闭合——每个角色都要有 durable 事件和适配器投影）

下一篇 → [05 · 工具系统与能力 Seam](./05-tools-and-seams.md)
