← 上一篇：[00 · Cordis 插件框架速通](./00-cordis-primer.md)

# 01 · 整体定位与分层

## 1.1 DeepSeek Harness 是什么

DeepSeek Harness（简称 dsh）是一个**基于插件架构的 AI 编码 Agent 运行时**，定位上对标 Claude Code、Codex CLI、OpenCode 这类产品，差异点是：

- **一切皆插件**：模型适配器、工具注册表、会话日志、agent 循环本身——每个部分都是 Cordis 插件，都可以从配置中替换。没有特权核心需要 patch。
- **事件溯源会话**：会话历史是 append-only 的事件日志（`SessionEvent`），LLM 消息历史从日志派生，不单独存储。崩溃后可恢复。
- **能力 Seam**：文件系统、子进程、shell、LSP、子代理等能力都通过 Service Definition / Provider / Consumer 三角色 seam 暴露，换一个 provider 就改变整个产品行为。
- **Profile / Bundle 组合**：运行时是一个从有序层（bundle → profile patch → home patch → overlay）组合出的插件树，`dsh --profile web` 和 `dsh --profile headless` 是同一个代码库的不同组合。

理解 dsh 的关键，不是"它有哪些功能"，而是**它如何把"AI Agent 运行时"做成一个完全可配置、可替换的插件系统**。

## 1.2 包布局概览

仓库根 `packages/` 下有约 40 个包组、上百个包。初学只需要关注核心脊柱和几个关键能力族：

```
core/           —— 产品 API 脊柱（每个组合都会启动）
  scope/          作用域上下文注册原语（库，无 ctx key）
  session/        事件溯源会话日志（ctx.sessions）
  system-prompt/  提示词与工具 schema 组装（ctx.systemPrompt）
  tools/          工具注册表与执行管线（ctx.tools）
  agent/          Agent 接口与注册表（ctx.agents）
  agent-loop/     默认 agent 驱动器（ctx.agentLoop）

llm/            —— LLM 能力族
  llm/            流式词汇与适配器 seam（ctx.llm）
  llm-deepseek/   DeepSeek 直连适配器
  token-meter/    token 计量（ctx.tokenMeter）

fs/             —— 文件系统能力族
  fs/             Service Definition（ctx.fs）
  fs-local/       本地实现
  fs-sandbox/     沙箱强制实现

shell/          —— shell 能力族
  shell/          执行器契约（ctx.shell）
  bash-local/     本地 bash 执行

subprocess/     —— 子进程能力族
  subprocess/     Service Definition（ctx.subprocess）
  subprocess-local/ 本地进程树

session/        —— 持久会话数据平面
  session-persistence/     持久化 seam（ctx.sessionPersistence）
  session-persistence-jsonl/  JSONL 后端
  session-persistence-sqlite/ SQLite 后端

boot/           —— 启动胶水
  app-boot/       profile/bundle 组合 + 启动序列

bundle/         —— profile 插件层
  base/           每个 profile 首层（模型/工具/持久化/沙箱/审批）
  web-app/        浏览器界面层
  headless/       一次性任务模式
```

其余像 `lsp/`、`skill/`、`compaction/`、`subagent/`、`web/`、`sandbox/`、`terminal/`、`jobs/`、`workflow/`、`sdk/`、`acp/`、`hooks/`、`interaction/` 等是能力族或集成层，初学可以略过，需要时再查 [packages/README.md](../../packages/README.md)。

## 1.3 依赖方向（核心约束）

[AGENTS.md](../../AGENTS.md) 中有一条铁律：扩展插件依赖 Service Definition（如 `dsh-agent`）而非具体实现（如 `dsh-agent-loop`），所以循环可替换。

画出来是这样：

```
                    ┌───────────────────────┐
                    │   cordis (vendored)   │  ← 插件框架
                    └───────────────────────┘
                               ▲
                    ┌──────────┴──────────┐
                    │      core spine      │
                    │  session / tools /   │
                    │  agent / agent-loop  │
                    │  system-prompt       │
                    └──────────┬──────────┘
                               ▲
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────┴───────┐ ┌──────┴───────┐ ┌──────┴───────┐
     │  llm seam      │ │  fs seam     │ │  shell seam  │
     │  (ctx.llm)     │ │  (ctx.fs)    │ │  (ctx.shell) │
     └────────┬───────┘ └──────┬───────┘ └──────┬───────┘
              │                │                │
     ┌────────┴───────┐ ┌──────┴───────┐ ┌──────┴───────┐
     │  providers     │ │  providers   │ │  providers   │
     │  llm-deepseek  │ │  fs-local    │ │  bash-local  │
     │  llm-pi-ai     │ │  fs-sandbox  │ │  pwsh-local  │
     └────────────────┘ └──────────────┘ └──────────────┘
              │                │                │
              └────────────────┼────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │   bundle / boot     │  ← 组合层
                    │  base / web-app /   │
                    │  headless           │
                    └─────────────────────┘
```

**这个分层不是装饰，是有真实后果的：**

1. **`dsh-agent-loop` 可替换**：扩展插件依赖 `dsh-agent`（Service Definition）而非 `dsh-agent-loop`（具体驱动器），所以换一个循环实现不需要改任何工具或 UI 插件。
2. **Provider 可替换**：工具通过 `ctx.fs` / `ctx.shell` / `ctx.subprocess` 使用能力，不导入具体 provider。换一个文件系统后端，所有文件工具自动跟着走。
3. **Core 不感知 HTTP**：dsh 的核心 API 都是 Cordis 服务方法，不依赖任何 HTTP 框架。Web/ACP/SDK 是不同的前端壳。

## 1.4 三种角色：Service Definition / Provider / Consumer

dsh 的每个能力 seam 都包含三个角色，**缺一不可**：

| 角色 | 职责 | 示例（文件系统） |
|---|---|---|
| **Service Definition** | 声明接口、类型、事件词汇 | `dsh-fs`（`ctx.fs`） |
| **Service Provider** | 实现接口 | `dsh-fs-local`、`dsh-fs-sandbox` |
| **Consumer** | 使用能力（通常是模型工具） | `dsh-tool-fs`（`read`/`write`/`edit` 工具） |

**为什么三个角色分开**：

- Service Definition 是**类型契约**——它定义"文件系统能做什么"，不关心怎么做。
- Provider 是**具体实现**——可以有多个，通过配置选择。换一个 provider 就改变整个产品的执行环境（如从本地切到远程沙箱）。
- Consumer 是**使用者**——通常是注册到 `ctx.tools` 的模型工具，它通过 `ctx.fs` 调用文件系统，不知道也不关心背后是哪个 provider。

**关键效果**：文件系统和子进程 provider 共享一个执行世界，所以把它们指向远程沙箱时，Bash、终端、LSP 都跟着走，不需要为每个工具写 provider 分支。

## 1.5 一句话总结每个核心包

| 包 | 它做什么 | 它不做什么 |
|---|---------|-----------|
| **cordis** | 插件框架：Service / Context / Events / Effect | 不包含任何 dsh 业务逻辑 |
| **session** | append-only 事件日志，LLM 历史从日志派生 | 不做持久化 I/O（那是 `session-persistence`） |
| **system-prompt** | 组装提示词段落 + 工具 schema | 不决定模型行为 |
| **tools** | 工具注册表 + 执行管线（pre-execute → execute → post-execute） | 不实现具体工具 |
| **agent** | Agent 接口、注册表、`agent/*` 事件词汇 | 不驱动循环 |
| **agent-loop** | 默认驱动器：claim → pre-step → request → stream → tools → turn-stopping | 不定义 Agent 接口 |
| **llm** | 流式词汇 + 适配器注册表 + `llm/stream` waterfall | 不实现具体 provider |
| **fs / shell / subprocess** | 能力 seam 的 Service Definition | 不做具体 I/O |

## 1.6 入口路径速查

读源码时，按这个表去定位：

| 想看什么 | 去哪 |
|---|---|
| Agent 循环驱动器 | [packages/core/agent-loop/src/index.ts](../../packages/core/agent-loop/src/index.ts) |
| Agent 接口定义 | [packages/core/agent/src/types.ts](../../packages/core/agent/src/types.ts) |
| 会话事件日志 | [packages/core/session/src/types.ts](../../packages/core/session/src/types.ts) |
| 会话存储 API | [packages/core/session/src/index.ts](../../packages/core/session/src/index.ts) |
| 工具注册表 | [packages/core/tools/src/index.ts](../../packages/core/tools/src/index.ts) |
| 工具定义 DSL | [packages/core/tools/src/schema.ts](../../packages/core/tools/src/schema.ts) |
| 系统提示组装 | [packages/core/system-prompt/src/index.ts](../../packages/core/system-prompt/src/index.ts) |
| LLM 流式类型 | [packages/llm/llm/src/types.ts](../../packages/llm/llm/src/types.ts) |
| LLM 适配器注册 | [packages/llm/llm/src/index.ts](../../packages/llm/llm/src/index.ts) |
| 持久化 seam | [packages/session/session-persistence/src/index.ts](../../packages/session/session-persistence/src/index.ts) |
| 启动组合 | [packages/boot/app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts) |
| 架构权威文档 | [docs/architecture.md](../architecture.md) |

## 1.7 关于"无特权核心"

dsh 最颠覆性的设计是**没有特权核心**：

- 模型适配器是一个插件（注册到 `ctx.llm`）
- 工具注册表是一个插件（`ctx.tools`）
- 会话日志是一个插件（`ctx.sessions`）
- **agent 循环本身也是一个插件**（`ctx.agentLoop`）

这意味着：你可以用配置替换 agent 循环，而不需要 fork 代码库。你可以换掉模型适配器、换掉工具集、换掉持久化后端，甚至换掉整个文件系统——全部通过 `cordis.yml` 配置完成。

**扩展方式是"挂载插件"，不是"修改核心"**：想加一个新能力？写一个插件，注册到 `ctx.tools` 或声明一个新的 `ctx` key，它就出现在 agent 的能力里。想拦截一个请求？监听 `agent/request` waterfall。想阻止一个 turn？监听 `agent/turn-stopping`。

## 深入追问

- 为什么 `agent-loop` 是 `bundle` 角色而不是 `core` 角色？（提示：它是唯一具体实现，可替换）
- 如果两个 provider 注册了同一个 `ctx.fs`，会发生什么？（提示：scope 隔离 + first-wins）
- `dsh-agent` 和 `dsh-agent-loop` 的分界未来会如何演化？

下一篇 → [02 · 一次 Turn 的旅程](./02-turn-journey.md)
