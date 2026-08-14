← 上一篇：[06 · 持久化与启动组合](./06-persistence-and-boot.md)

# 07 · 设计哲学提炼

前六篇都在讲"是什么"。这一篇退一步，提炼 dsh 设计中**可复用的原则**——这些原则不依赖 dsh 的具体实现，可以指导你设计任何 Agent 系统。

每条原则配一个 dsh 中的实例，便于回溯。

## 7.1 一切皆插件，无特权核心

**原则**：系统没有不可替换的核心。每个部分——包括循环本身——都是插件，都可以从配置中替换。

**dsh 实例**：

- 模型适配器是插件（`ctx.llm`）
- 工具注册表是插件（`ctx.tools`）
- 会话日志是插件（`ctx.sessions`）
- **agent 循环本身是插件**（`ctx.agentLoop`）

**为什么**：

1. **可定制性**：不同部署场景需要不同行为——headless 任务 vs 交互式 web vs 自动化 ACP。插件组合让同一个代码库支持所有场景。
2. **可测试性**：测试时可以替换任意部分——用 mock LLM、用内存文件系统、用重放适配器。
3. **演进性**：新能力通过挂载插件加入，不需要修改核心代码。旧能力通过禁用插件移除，不需要删代码。

**反例**：把 agent 循环硬编码在"核心"里——每次想改行为都要 fork 代码库。

## 7.2 事件溯源：模型可见 ⟺ 已记录

**原则**：模型看到的任何东西都必须能从持久化日志重建。运行时断言守护这条不变量。

**dsh 实例**：

- 用户输入通过 `agent.followup()` 进入 inbox，最终产生 `user/message` 事件
- 注入上下文通过 `agent.inject()` 产生 `user/message` 事件（`source` 非 `'user'`）
- 请求配置通过 `agent/request` waterfall，记录 `request/header` 事件
- 工具调用先记录 `tool/call` 事件，再执行
- 工具结果记录 `tool/result` 事件

**为什么**：

1. **可重放**：如果模型行为异常，你能从日志精确重建"模型当时看到了什么"。
2. **可审计**：每一步都有记录，可以追溯为什么模型做了某个决策。
3. **崩溃恢复**：日志是持久化的，崩溃后重载就能恢复到崩溃前的状态。

**反例**：直接修改 LLM 请求对象——不记录、不可重放、不可审计。

## 7.3 先记录、后执行

**原则**：有副作用的操作（尤其是工具调用）先写入日志，再执行。这样即使执行中崩溃，恢复时知道有过这次尝试。

**dsh 实例**：

- `tool/call` 事件在工具执行前就写入日志
- 如果执行中崩溃，恢复时看到 `tool/call` 但没有 `tool/result`——知道执行被中断
- 恢复时用合成 `turn/end {reason:{kind:'interrupted'}}` 关闭孤儿 turn

**为什么**：

1. **不丢事实**：工具调用是一个事实——"模型决定调用这个工具"。这个事实不该因为执行失败而丢失。
2. **可诊断**：恢复后能看到"模型调用了 read 但执行被中断"，而不是"什么都没发生"。
3. **幂等性基础**：如果工具是幂等的，恢复后可以安全重试；如果不是，至少知道上次尝试过。

**反例**：先执行再记录——执行中崩溃就完全不知道发生过什么。

## 7.4 Capability Seam：三角色分离

**原则**：可交换的能力分成三个角色——Service Definition（声明接口）、Provider（实现）、Consumer（使用）。三者独立演进。

**dsh 实例**：

- 文件系统：`dsh-fs`（定义） / `dsh-fs-local` / `dsh-fs-sandbox`（实现） / `dsh-tool-fs`（消费）
- 换一个 provider（从 local 到 sandbox），所有文件工具自动跟着走，不改任何工具代码

**为什么**：

1. **执行环境可切换**：本地 → 远程沙箱 → E2B，只需要换 provider 配置。
2. **实现可替换**：测试时用 mock provider；生产用真实 provider。
3. **消费者不耦合实现**：工具代码只依赖 `ctx.fs` 接口，不知道背后是哪个 provider。

**反例**：工具直接 `import { readFile } from 'fs'`——绑定了本地文件系统，无法切换。

## 7.5 Waterfall：协作式拦截

**原则**：需要多个插件协作的决策点用 waterfall 事件——每个监听器可以包装、替换、观察，通过 `next()` 委托。

**dsh 实例**：

- `agent/pre-step`：hook 可以 reject 或改写消息，然后 `next()` 委托
- `tools/pre-execute`：权限检查可以 allow/deny/ask，然后 `next()` 委托
- `llm/stream`：可以包装流式调用，记录 metrics 或修改请求

**为什么**：

1. **可组合**：多个监听器可以同时作用于同一个决策点，互不干扰。
2. **有序**：监听器按注册顺序执行，可以预测行为。
3. **可短路**：不需要 `next()` 时可以 short-circuit——如权限拒绝。

**反例**：用全局变量传递决策——不可组合、不可预测、不可短路。

## 7.6 Surface 模型：追加只日志 + 可修正投影

**原则**：日志是 append-only 的（不改不删），但投影层支持"替换"操作——让后续事件修正之前的投影。

**dsh 实例**：

- 事件日志只追加：`user/message`、`assistant/message`、`tool/result` 都是 append
- Surface 事件带 `surfaceOp`：`'append'` 或 `{op:'replace', start, end}`
- 压缩（compaction）产生 replace 操作：把前 20 条消息替换成一条摘要
- 日志本身不改——replace 是投影层概念

**为什么**：

1. **不可变性**：日志是已发生的事实，不改不删。改了就不可审计。
2. **可修正**：投影层允许"修正之前的视图"——如压缩、如工具结果覆盖。
3. **多视图**：同一个日志可以派生出不同视图——模型历史、UI 转录、遥测统计。

**反例**：直接修改日志中的旧事件——破坏不可变性、不可审计、不可重放。

## 7.7 注册是可逆 Effect

**原则**：所有注册（工具、提示词段、监听器、适配器）都通过 `ctx.effect()` 或 `ctx.on()` 安装，卸载时按序回滚。

**dsh 实例**：

- 工具注册：`ctx.effect(() => { ctx.tools.register(myTool); return () => ctx.tools.unregister(myTool) })`
- 事件监听：`ctx.on('tools/pre-execute', handler)` 自动返回 disposer
- agent scoped 注册：`agent.ctx` 上的所有注册在 agent 销毁时自动回滚

**为什么**：

1. **热重载**：修改配置后，旧插件卸载（注册回滚），新插件挂载——不需要重启。
2. **无残留**：插件卸载后不会留下孤立的注册——避免"幽灵工具"或"幽灵监听器"。
3. **作用域隔离**：agent 级注册在 agent 销毁时回滚，不影响其他 agent。

**反例**：注册时不返回 disposer——卸载时不知道该清理什么，留下残留。

## 7.8 Misconfiguration Fails Loud

**原则**：配置错误要尽早、尽可能明确地失败。不静默跳过、不使用默认值掩盖问题。

**dsh 实例**：

- Loader 失败 → stderr 标注具体行 + exit(1)
- bundle 声明缺失 → fail-loud（不静默跳过）
- `SESSION_FORMAT_VERSION` 不匹配 → 拒绝加载（不迁移）
- profile 引用不存在的 entry id → stderr 警告

**为什么**：

1. **可诊断**：错误信息明确指向问题，而不是" silently 不工作"。
2. **安全**：静默跳过可能导致"以为开了沙箱但其实没开"——这是安全问题。
3. **early failure**：启动时失败比运行时失败好——不会在关键操作中途出问题。

**反例**：配置错误时静默回退到默认值——用户以为配置生效了，其实没有。

## 7.9 判别联合 + Declaration Merging

**原则**：可扩展的类型用 `…Map → derived-union` 模式——interface 按 discriminant tag 键控，插件通过 declaration merging 添加变体。

**dsh 实例**：

- `ContentBlockMap` → `ContentBlock`：插件可以添加新块类型
- `SessionEventMap` → `SessionEvent`：插件可以添加新事件类型
- `TurnEndReasonMap` → `TurnEndReason`：插件可以添加新结束原因
- `switch` 以 `assertNever` 结束——typo 的 tag 编译失败

**为什么**：

1. **可扩展**：插件添加新变体不需要修改声明包。
2. **类型安全**：`switch` 穷尽检查，漏掉的 case 编译失败。
3. **模式一致**：所有可扩展联合用同一模式，降低认知负担。

**反例**：用 `string` 标签 + `if/else` 链——不类型安全、不可扩展、容易漏 case。

## 深入追问

- "一切皆插件"的代价是什么？（提示：启动性能 + 间接性 + 调试难度）
- 如果两个插件注册了冲突的 `ctx` key，怎么办？（提示：scope 隔离 + first-wins）
- Surface replace 能否无限嵌套？有没有限制？（提示：replace 指向之前的事件 seq，不能循环）
- `assertNever` 在运行时还是编译时检查？（提示：编译时类型检查 + 运行时 throw）

下一篇 → [08 · 未来追问清单](./08-further-questions.md)
