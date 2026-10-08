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

- `dsh-agent-loop` 是默认驱动器而不是唯一驱动器——如果有人写了另一个 `agent-loop` 实现，两个能共存吗？配置如何选择？
- `dsh-agent` 和 `dsh-agent-loop` 的分界未来会如何演化？`Agent` 接口会不会吸收更多 `agent-loop` 的职责？
- 为什么 `scope/` 是非 Service 包（库，无 ctx key）？它为什么必须在 `session/` 和 `system-prompt/` 之下？
- 如果一个能力 seam 的三个角色开始独立演进（如 Provider 有了 Service Definition 不支持的新功能），该怎么拆分？
- host 平面和 client 平面如何分工？为什么工具注册表留在 host 平面，而工具呈现模式由 agent preset 选择？

## Turn 流程

- 如果 `agent.followup()` 之后进程立刻崩溃，inbox 里的消息会怎样？inbox 是 durable 投影还是纯内存？
- `steer` 和 `inject` 都是 `next-step` target，区别只在是否唤醒——为什么这个区别重要？`inject` 的"不唤醒"语义在什么场景下有用？
- `agent/pre-step` reject 后不把消息放回 inbox——如果用户想重试，该怎么做？
- `startsRequestSeries` 什么时候被设置？"请求系列"对 `request/header` 的记录策略有什么影响？
- `agent/turn-stopping` 是 serial 事件，没有 `next()`——它如何实现"监听器 object 就再跑一个 step"？
- `concludesTurn` 和 `agent/turn-stopping` 的优先级关系是什么？如果工具结果 `concludesTurn` 但同时有 racing steering，会发生什么？

## 会话日志

- `assistant/message` 为什么内嵌流记录（`stream: AssistantStreamRecord[]`）而不是每个 chunk 一个事件？"压缩但不合并 delta 边界"是什么意思？
- `assistant/attempt` 保留哪四类未提交尝试？它为什么不产生模型历史？
- Surface 的 `replace` 操作会不会破坏 append-only 不变量？replace 指向的旧事件还在日志里吗？
- 如果一个 turn 有几百个 step，`deriveMessages()` 的增量投影如何工作？surface content generation 何时失效？
- 压缩（compaction）如何与 surface replace 配合？压缩后的 replace 操作能被再 replace 吗？
- `request/header` 事件记录了 `EpochHeader`——它包含什么？为什么"请求可从日志重建"很重要？
- `session/end-seed` 事件的用途是什么？fork 和 resume 如何使用种子边界？`isSeeded` 为什么在 header 上而精确继承长度在存储元数据里？
- `ctx.sessionProjections` 的投影单元如何增量折叠事件？它和 `deriveMessages()` 的缓存策略有什么不同？

## LLM 流式

- `tool-call` 的 `arguments` 为什么是字符串而不是解析后的对象？这对工具验证有什么影响？
- `BlockAssembler` 为什么是共享的？如果某个 provider 的流式格式特殊（如非标准 thinking 块），assembler 如何处理？
- `LlmCallConfig` 的 `reasoningEffort` 如何映射到不同 provider 的 API？DeepSeek 和 pi-ai 的映射一样吗？
- `llm/stream` waterfall 的监听器可以做什么？它能替换流吗？能修改请求吗？
- `agent/request-error` waterfall 在什么时机运行？它和 `agent/request` 的关系是什么？
- `prepareCall()` 的"同一次注册代际"是什么意思？为什么取消发生在 prepareCall 阶段时 system 和 user 消息都不落日志？
- 工具结果为什么是 tool-role 消息？`MessageRoleMap` 为什么是闭合的而 `ContentBlockMap` 是 merge-extensible 的？

## 工具系统

- `ToolGuard` 只能减权限——这个单调性如何保证？有没有运行时断言？
- `tools/execute` 是 around-dispatch waterfall——它的 `next()` 返回什么？监听器能替换执行结果吗？
- `finalizeContent` 必须 total（不抛异常）——如果它实际上抛了会怎样？
- `additionalContexts` 在 `tool/result` 后注入为 `user/message`——这些消息的 `source` 是什么？
- `presentCall` / `presentResult` 是纯函数——如果工具想在 UI 上显示动态状态（如进度条），该怎么做？
- PTC 模式的 `tools/ptc-dispatch-log` waterfall 是什么？它如何暴露每个结算子派发？为什么只有外层精选结果进入模型历史？
- `deferLoading` 的工具 schema 和 `tool-addition` 块如何配合实现工具的延迟加载？

## 能力 Seam

- 文件系统、子进程、沙箱 provider "共享一个执行世界"——这个共享在代码层面如何实现？`ssh/` 族的三个 provider 如何保证一致性？
- `dsh-fs-sandbox` 和 `dsh-sandbox` 的关系是什么？沙箱如何强制文件系统策略？
- 子代理 seam 有这么多 provider（spawn / fork / acp / codex / claude-code / dsh-sdk）——它们的启动时 vs 运行时能力拆分是什么？
- `ctx.userQuestions` 和 `ctx.approval` 的 answerer 由谁提供？headless（无 GUI）模式下审批和 ask-user 会发生什么？（提示：fail closed）
- `ctx.workflowEngine` 的 provider 是 `workflow-ptc`——脚本里的 `agent()` 调用如何变成子代理 run？

## 持久化与启动

- JSONL 后端通过 `runPersistenceContract` 套件验证——这个套件测试什么？如果将来加入第二个后端，它如何保证行为一致？
- `session/flush` 是 awaited 的——如果持久化插件写入失败（如磁盘满），会发生什么？
- 崩溃恢复时用合成 `interrupted` 关闭孤儿 turn——这个合成事件写入日志吗？它会影响 `seq` 编号吗？`ToolCallRecovery` 的 `TOOL_NOT_STARTED` 和 `TOOL_OUTCOME_UNKNOWN` 分别对应什么崩溃时机？
- Profile 模板（`acp`、`web`、`headless`、`sdk`、`sdk-minimal`）首次使用时初始化——初始化做了什么？为什么模板更新后已初始化的 profile 不跟随？
- profile 的 `node_modules` 拦截层如何参与模块解析？pnpm 管理的条目为什么需要 real-path 祖先链处理？
- `dsh-hmr` 串行化配置与模块重载——为什么默认只在 web 组合启用，headless/SDK/ACP 禁用？

## 设计哲学

- "一切皆插件"的代价是什么？启动性能、间接性、调试难度如何平衡？
- "模型可见 ⟺ 已记录"的运行时断言如何实现？它检查什么？在什么时机检查？
- 如果一个新事件类型忘记加入 `SessionEventMap`，但被 `session.append()` 调用了，会发生什么？`ignorable` 标记的"过度拒绝好于静默吞掉"是什么意思？
- `assertNever` 在运行时还是编译时检查？如果运行时收到了一个未知的 tag，会怎样？
- `SESSION_FORMAT_VERSION` 已经是 4，且有已发布格式的迁移链——"已发布 Session 数据"的承诺如何约束格式演化？（提示：[docs/session-format-status.md](../session-format-status.md)，已提交代际不可变）

## 跨系统

- 压缩（compaction）如何与 agent 循环交互？`compaction-image-offload` 的 `IMAGE_OFFLOAD_REQUIRED` 重试路径是什么？
- 子代理（subagent）如何继承父代理的能力？`composeFrom` 和 `mount` 的区别是什么？
- 目标（goal）如何驱动续行？`ctx.goals` 和 `agent/*` 事件如何配合？
- webhook（`ctx.webhookRuntime`）如何从外部事件创建 Workspace Session？"trusted rule"防的是什么？
- MCP 客户端桥（`packages/mcp/mcp-client`）如何把外部服务器工具注册到 `ctx.tools`？这些工具经过执行管线吗？
- Agent Teams（`ctx.agentTeams`，experimental）在 subagent seam 之上加了什么？roster / task board / mailbox 各解决什么问题？

## 怎么用这份清单

1. **挑一个问题**，回到对应的源码目录，尝试自己回答。
2. **验证你的答案**：写一个小测试或 demo，看行为是否符合你的预期。
3. **发现新问题**：在回答过程中发现的新问题，追加到这份清单里。
4. **与正式文档对照**：你的答案和 [architecture.md](../architecture.md) / [subsystems/](../subsystems/README.md) 的描述一致吗？如果不一致，以正式文档为准。

这份清单不是"已知问题列表"——这些是**学习入口**，每个问题都指向一个可以深入理解的设计决策。回答它们的过程，就是理解 dsh 架构的过程。
