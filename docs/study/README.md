# DeepSeek Harness 架构学习材料

这是一份围绕 [DeepSeek Harness](../architecture.md) 源码的代码 Agent 架构学习材料。

目标：**由浅入深、易懂而不失深度**。不罗列所有细节，只把真正承载设计思想的核心抽象讲透，并保留未来追问的入口。

## 前置知识

在开始之前，你需要了解：

1. **TypeScript 基础**：类型系统、泛型、`async/await`、declaration merging
2. **Cordis 框架**：DeepSeek Harness 的核心框架。如果你没接触过，**先读 [00 · Cordis 插件框架速通](./00-cordis-primer.md)**（~15 分钟），它覆盖了源码中最常见的模式
3. **事件溯源基本概念**：dsh 用 append-only 事件日志做会话持久化，了解"事件"和"投影"即可
4. **LLM 流式 API**：了解 OpenAI 兼容的 streaming 响应格式

如果你已经熟悉 Cordis 的 Service / Context / Event / Effect 模式，可以直接跳过第 00 篇。

## 这份材料不写什么

- 不逐文件搬运源码——只在关键处引用最小代码片段
- 不覆盖每一个内置工具 / Provider / 能力 seam 的实现细节
- 不重述 [architecture.md](../architecture.md) 已经写明的架构约束，而是解释"为什么这样设计"
- 不写 API 使用文档——那是 [docs/user/](../user/index.md) 的事

## 阅读地图

按顺序读，**第 02 篇是骨架**，把它读完后其余各篇都是横向深入。

| # | 标题 | 你会得到什么 | 推荐顺序 |
|---|------|--------------|---------|
| 00 | [Cordis 插件框架速通](./00-cordis-primer.md) | Service / Context / Event / Effect 等核心模式 | **0（不熟悉 Cordis 必读）** |
| 01 | [整体定位与分层](./01-overview.md) | dsh 是什么、为什么这样切包、插件如何组合 | 1 |
| 02 | [一次 Turn 的旅程](./02-turn-journey.md) | 从用户输入到模型回答再到工具调用的完整时序 | **2（必读）** |
| 03 | [会话日志与事件溯源](./03-session-log.md) | SessionEvent、deriveMessages、Surface 模型、turn/step | 3 |
| 04 | [LLM 流式与 Provider 适配](./04-llm-streaming.md) | StreamChunk、ContentBlock、LlmAdapter、重试策略 | 4 |
| 05 | [工具系统与能力 Seam](./05-tools-and-seams.md) | ToolDefinition、执行管线、Service Definition / Provider / Consumer | 5 |
| 06 | [持久化与启动组合](./06-persistence-and-boot.md) | JSONL/SQLite 持久化、崩溃恢复、Profile / Bundle / Boot | 6 |
| 07 | [设计哲学提炼](./07-design-principles.md) | 抽象出可复用的设计原则 | 7 |
| 08 | [未来追问清单](./08-further-questions.md) | 每个子系统留下的可深挖问题 | 翻阅 |

## 快速上手：先跑起来再看文档

读完第 01-02 篇后（甚至之前），建议先把项目跑起来，建立直观感受：

```bash
# 1. 安装依赖
pnpm install

# 2. 启动 headless 模式跑一个任务（需要 DEEPSEEK_API_KEY）
pnpm dsh --profile headless "hello, explain what you are"

# 3. 或者启动 web 界面
pnpm dsh --profile web
# → 浏览器打开后，创建会话、发消息、观察工具调用
```

**观察要点**：
- headless 模式下 agent 跑完即退出——这就是"一次性任务"
- web 模式下会话是持久的——刷新页面后历史还在
- 如果工具调用需要授权，会看到审批提示——这就是"交互平面"

## 怎么对照源码读

1. 仓库根目录先读：[AGENTS.md](../../AGENTS.md)（编码规范 + 核心约束）、[docs/architecture.md](../architecture.md)（架构权威）。
2. 每一篇都会在文中插入形如 `packages/core/agent-loop/src/index.ts` 的路径，**遇到就跳过去扫一眼**，不必逐行读，能定位即可。
3. 遇到不认识的 Cordis 模式（`ctx.effect()`、`ctx.on()`、`Service` 等），回 [00 · Cordis 速通](./00-cordis-primer.md) 查。

## 阅读节奏建议

- **第零遍**（如果你不熟悉 Cordis）：读 [00 · Cordis 速通](./00-cordis-primer.md)，~15 分钟。
- **第一遍**：只读 01、02、07，建立"地形感"。配合上面的"快速上手"把项目跑一遍。
- **第二遍**：按 03→04→05→06 顺序深入。每篇遇到代码时跳到源码扫一眼。
- **第三遍**：把 08 的追问清单当作练习题，挑感兴趣的回去对照代码。

## 关于学习材料与正式文档的关系

这份学习材料是 [docs/architecture.md](../architecture.md) 及其子系统文档的**导读**，不替代它们：

- **正式文档**描述当前状态，是权威；学习材料解释设计思想，是辅助。
- 遇到矛盾时，以正式文档为准。
- 学习材料中的代码片段为概念精简，不保证逐字编译；以源码为最终真相。
