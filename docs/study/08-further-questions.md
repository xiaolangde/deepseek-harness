← 上一篇：[07 · 设计哲学提炼](./07-design-principles.md)

# 08 · 未来追问清单

前七篇解释了 dsh 的核心设计。这一篇把每个子系统留下的可深挖问题整理成清单，当作练习题——挑感兴趣的回去对照代码，验证你的理解。

## Cordis 框架

- `waterfall` 和 `serial` 都按注册顺序执行，区别在哪？`next()` 委托 vs bail 短路如何影响监听器行为？
- `ctx.isolate(name)` 如何实现"同一个进程里两个 agent 用不同的文件系统后端"？scope label 的 join 语义是什么？
- `ctx.effect()` 返回的 disposer 是 single-shot 的——重复调用返回什么？复合（generator）effect 如何嵌套 teardown 顺序？
- Cordis 的 Loader 如何解析 bare module specifier？`bareModuleBaseUrl` 在什么场景下需要？
- `!!js` 表达式在 `config` 和 `disabled` 字段中的求值时机有什么不同？为什么不能在其他元数据中用 `!!js`？

## 整体架构

- `dsh-agent-loop` 是 `bundle` 角色而不是 `core` 角色——这意味着什么？如果有人写了另一个 `agent-loop` 实现，两个能共存吗？
- `dsh-agent` 和 `dsh-agent-loop` 的分界未来会如何演化？`Agent` 接口会不会吸收更多 `agent-loop` 的职责？
- 为什么 `scope/` 是非 Service 包（库，无 ctx key）？它为什么必须在 `session/` 和 `system-prompt/` 之下？
- 如果一个能力 seam 的三个角色开始独立演进（如 Provider 有了 Service Definition 不支持的新功能），该怎么拆分？

## Turn 流程

- 如果 `agent.followup()` 之后进程立刻崩溃，inbox 里的消息会怎样？inbox 是 durable 投影还是纯内存？
- `steer` 和 `inject` 都是 `next-step` target，区别只在是否唤醒——为什么这个区别重要？`inject` 的"不唤醒"语义在什么场景下有用？
- `agent/pre-step` reject 后不把消息放回 inbox——如果用户想重试，该怎么做？
- `agent/turn-stopping` 是 serial 事件，没有 `next()`——它如何实现"监听器 object 就再跑一个 step"？
- `concludesTurn` 和 `agent/turn-stopping` 的优先级关系是什么？如果工具结果 `concludesTurn` 但同时有 racing steering，会发生什么？

## 会话日志

- `assistant/chunk` 和 `assistant/message` 都要存——这双份存储的代价是什么？能否只存一份？
- Surface 的 `replace` 操作会不会破坏 append-only 不变量？replace 指向的旧事件还在日志里吗？
- 如果一个 turn 有几百个 step，`deriveMessages()` 的缓存机制如何工作？缓存何时失效？
- 压缩（compaction）如何与 surface replace 配合？压缩后的 replace 操作能被再 replace 吗？
- `request/header` 事件记录了 `EpochHeader`——它包含什么？为什么"请求可从日志重建"很重要？
- `session/end-seed` 事件的用途是什么？fork 和 resume 如何使用种子边界？

## LLM 流式

- `tool-call` 的 `arguments` 为什么是字符串而不是解析后的对象？这对工具验证有什么影响？
- `BlockAssembler` 为什么是共享的？如果某个 provider 的流式格式特殊（如非标准 thinking 块），assembler 如何处理？
- `LlmCallConfig` 的 `reasoningEffort` 如何映射到不同 provider 的 API？DeepSeek 和 pi-ai 的映射一样吗？
- `llm/stream` waterfall 的监听器可以做什么？它能替换流吗？能修改请求吗？
- `agent/request-error` waterfall 在什么时机运行？它和 `agent/request` 的关系是什么？
- `PreparedLlmCall` 的"保持同一适配器注册跨日志与派发"是什么意思？为什么需要这个？

## 工具系统

- `ToolGuard` 只能减权限——这个单调性如何保证？有没有运行时断言？
- `tools/execute` 是 around-dispatch waterfall——它的 `next()` 返回什么？监听器能替换执行结果吗？
- `finalizeContent` 必须 total（不抛异常）——如果它实际上抛了会怎样？
- `additionalContexts` 在 `tool/result` 后注入为 `user/message`——这些消息的 `source` 是什么？
- `presentCall` / `presentResult` 是纯函数——如果工具想在 UI 上显示动态状态（如进度条），该怎么做？
- Code Mode 的 `tools/code-dispatch-log` waterfall 是什么？它如何暴露每个结算子派发？

## 能力 Seam

- 文件系统和子进程 provider "共享一个执行世界"——这个共享在代码层面如何实现？
- `dsh-fs-sandbox` 和 `dsh-sandbox` 的关系是什么？沙箱如何强制文件系统策略？
- 子代理 seam 有这么多 provider（spawn / fork / acp / codex / claude-code / dsh-sdk）——它们的启动时 vs 运行时能力拆分是什么？
- `ctx.userQuestions` 没有 impl——为什么？谁来提供活跃的人类 answer provider？
- `ctx.approval` 的 impl 是 `acp`——这意味着非 ACP 模式下审批由谁处理？

## 持久化与启动

- JSONL 和 SQLite 后端通过 `runPersistenceContract` 套件验证——这个套件测试什么？如何保证两个后端行为一致？
- `session/flush` 是 awaited parallel——如果某个持久化插件写入失败（如磁盘满），会发生什么？
- 崩溃恢复时用合成 `interrupted` 关闭孤儿 turn——这个合成事件写入日志吗？它会影响 `seq` 编号吗？
- Profile 模板（`web`、`headless`）首次使用时自动初始化——这个初始化做了什么？如果模板更新了，已初始化的 profile 会更新吗？
- `healProfilesModuleFallback` 维护扁平的 `node_modules` 目录——为什么需要这个？Cordis Loader 的模块解析如何工作？
- HMR 只重组用户 patch 层，不重组 bundle 层——为什么？如果 bundle 层需要热更新怎么办？

## 设计哲学

- "一切皆插件"的代价是什么？启动性能、间接性、调试难度如何平衡？
- "模型可见 ⟺ 已记录"的运行时断言如何实现？它检查什么？在什么时机检查？
- 如果一个新事件类型忘记加入 `SessionEventMap`，但被 `session.append()` 调用了，会发生什么？
- `assertNever` 在运行时还是编译时检查？如果运行时收到了一个未知的 tag，会怎样？
- dsh 是 pre-release，`SESSION_FORMAT_VERSION` 在 `0`——这意味着什么？格式变更的自由度有多大？

## 跨系统

- 压缩（compaction）如何与 agent 循环交互？它在 `agent/pre-step` 监听器中运行——这意味着压缩发生在什么时机？
- 子代理（subagent）如何继承父代理的能力？`composeFrom` 和 `mount` 的区别是什么？
- 目标（goal）如何驱动续行？`ctx.goals` 和 `agent/*` 事件如何配合？
- 工作流（workflow）在 worker 线程中运行——它如何与主线程的 agent 循环通信？
- MCP 客户端桥如何把外部服务器工具注册到 `ctx.tools`？这些工具经过执行管线吗？

## 怎么用这份清单

1. **挑一个问题**，回到对应的源码目录，尝试自己回答。
2. **验证你的答案**：写一个小测试或 demo，看行为是否符合你的预期。
3. **发现新问题**：在回答过程中发现的新问题，追加到这份清单里。
4. **与正式文档对照**：你的答案和 [architecture.md](../architecture.md) / [subsystems/](../subsystems/README.md) 的描述一致吗？如果不一致，以正式文档为准。

这份清单不是"已知问题列表"——这些是**学习入口**，每个问题都指向一个可以深入理解的设计决策。回答它们的过程，就是理解 dsh 架构的过程。
