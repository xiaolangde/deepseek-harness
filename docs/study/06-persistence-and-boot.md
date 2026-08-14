← 上一篇：[05 · 工具系统与能力 Seam](./05-tools-and-seams.md)

# 06 · 持久化与启动组合

这一篇讲两件事：会话日志如何持久化到磁盘，以及 dsh 运行时如何从配置组合出插件树。

## 6.1 持久化 Seam

`ctx.sessionPersistence`（`SessionPersistence`）是持久化的 Service Definition。它定义了 locate / create / append / prepare / load / inspect / readFrom / list / listSnapshots 等方法——全部基于 `SessionEvent`，没有平行的持久化事件类型。

两个可互换后端实现同一契约：

| 后端 | 存储 | 特点 |
|---|---|---|
| `dsh-session-persistence-jsonl` | 每会话一个 JSONL 文件 | 追加只逻辑日志；默认 Zstandard 压缩帧；崩溃安全原子写 |
| `dsh-session-persistence-sqlite` | 共享 SQLite 库 | 每 `SessionEvent` 一行；单调 `SCHEMA_VERSION` pragma |

两者通过共享的 `runPersistenceContract` 套件验证——保证行为一致。

## 6.2 flush 检查点机制

会话日志的 `append()` 是同步的——事件立刻进入内存，不阻塞在 I/O 上。持久化是异步的：

```
Session.append(event)
  │
  ├── 同步：追加到内存日志
  │     └── session/event (emit, fire-and-forget) → UI/SDK
  │
  └── 异步：持久化插件复制到 per-session 控制器
        │
        ├── 首个待处理事件 → 启动固定批处理窗口
        ├── 后续事件 → 加入当前批次（不重置截止）
        └── 窗口过期 → 持久化批次 → session/flush

session/flush (awaited parallel)
  └── 通知所有持久化插件"现在是安全检查点"
```

**关键设计**：

- **`session/event` 不阻塞生产者**——持久化插件把事件复制进控制器，然后异步写入。
- **批处理窗口是固定的**——后续事件加入不重置截止时间，避免"永远在积压"。
- **`session/flush` 是 awaited parallel**——调用方可以等待持久化完成（如关闭前检查点）。
- **被拒的后台写保留事件并暂停自动重试**——不会丢数据，但需要外部干预。

## 6.3 崩溃恢复

进程崩溃后重启，重载会话日志时可能发现**未关闭的 turn**——有 `turn/start` 但没有 `turn/end`。

**dsh 不截断日志**——原因：

1. 单个 turn 可能很大，截断会丢大量数据。
2. 事件已经持久化了，它们是已发生的事实。

**恢复策略**：用合成 `turn/end {reason:{kind:'interrupted'}}` 关闭孤儿 turn。

```
崩溃前的日志:
  seq 1: turn/start {turn: 5}
  seq 2: step/start {turn:5, step:1}
  seq 3: user/message
  seq 4: assistant/chunk
  ... (没有 turn/end)

恢复后的日志:
  seq 1: turn/start {turn: 5}
  seq 2: step/start {turn:5, step:1}
  seq 3: user/message
  seq 4: assistant/chunk
  seq 5: turn/end {turn:5, reason:{kind:'interrupted'}}  ← 合成
```

**`interrupted` 是唯一非循环发出的 `TurnEndReason`**——它只在崩溃恢复时出现，不会在正常运行中产生。

**恢复流程**：

1. `inspect(id)`：构造不可变逻辑 Session，不发布不写——只用于检查日志状态。
2. `prepare(id)`：预留 Session，提交待修复（合成 `interrupted` 事件），返回可处置发布句柄。
3. 修复仅用于冷会话——活跃 id 的 `load` 等待内存快照持久且仅在平衡时返回。

## 6.4 SessionHeader —— 与日志分离存储

`SessionHeader` 是会话的元数据头，与事件日志分离存储：

```ts
interface SessionHeader {
  version: number            // 格式版本
  id: SessionId
  createdAt: number
  cwd: string                // 工作目录
  parentSession?: SessionId  // 分叉来源
  seedLength?: number        // 种子长度
  origin: SessionOrigin      // 起源分类
  delegationDepth?: number   // 委托深度
  agentPreset?: string       // 使用的 preset
}
```

**为什么分离**：

- 头信息是会话级的元数据，不是事件——不需要 append-only。
- 头信息需要频繁读（列表会话时），分离存储更高效。
- `version` 不匹配时拒绝加载（不迁移）——`SESSION_FORMAT_VERSION` 在 `0`，无兼容承诺。

## 6.5 Profile 与 Bundle —— 启动组合

dsh 运行时不是一个"编译好的程序"，而是一个**从有序层组合出的插件树**。

```
组合顺序（从下到上）:
  1. bundle 层（按 profile 声明的顺序）
     ├── dsh-base         （核心：模型/工具/持久化/沙箱/审批）
     ├── dsh-web-app      （web 界面层，如果是 web profile）
     └── dsh-headless     （一次性任务层，如果是 headless profile）
  2. profile 的 cordis.patch.yml
  3. home 级 cordis.patch.yml
  4. --patch overlay（命令行传入）
```

**Profile**：`$DSH_HOME/profiles/<name>/` 下的目录，持有：

- `package.json`：树外插件 `dependencies` + `dsh.profile` manifest（有序 `bundles` 列表）。
- `cordis.patch.yml`：用户自己的 patch 层。

**Bundle**：manifest 声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` 的 npm 包。它是 Cordis 配置行和它们挂载的代码的**分发格式**。

**Patch**：一个 patch 目标一个 entry id，替换其整个 config，或插入新行。上层 patch 覆盖下层。

```
dsh-base/cordis.patch.yml:
  - name: dsh-llm-deepseek
    config: { baseURL: ... }
  - name: dsh-tool-fs
    config: { ... }

profile/cordis.patch.yml:
  - name: dsh-llm-deepseek        ← 覆盖 base 层的同名 entry
    config: { baseURL: ..., timeout: 60000 }
  - name: my-custom-tool           ← 插入新 entry
    config: { ... }
```

## 6.6 Boot 启动序列

`boot()` 函数（`packages/boot/app-boot/src/index.ts`）是启动入口：

```
boot(binName, configPath, patches?, prepare?)
  │
  ├── 1. 加载 .env（loadLayeredEnv）
  │     调用目录文件 > Harness-home 文件 > 继承环境
  │
  ├── 2. 安装 fail-loud Loader 守卫
  │     Loader 失败 → stderr 标注 + exit(1)
  │
  ├── 3. 解析配置路径（快照模式用 cordis.snapshot.yml）
  │
  ├── 4. 创建根上下文
  │     向 Loader 暴露 dshHomePath(...)
  │
  ├── 5. 运行可选 host 准备（prepare）
  │     可用 Loader，提供 launcher 拥有的上下文槽
  │
  ├── 6. 挂载并 await include 树
  │     composeEntries 在空条目列表上应用 patch 层
  │
  ├── 7. 断言条目已加载与已激活
  │     assertEntriesLoaded + assertEntriesActivated
  │
  └── 8. 返回根上下文
        或处置部分上下文并以标注错误拒绝
```

**fail-loud 原则**：dsh 不静默跳过错误。Loader 失败时，在 stderr 标注具体行 + 退出码 1。这符合 dsh 的设计哲学：**misconfiguration fails loud**。

## 6.7 三种运行模式

| 模式 | 命令 | 特点 |
|---|---|---|
| **web** | `dsh --profile web` | 浏览器界面；HTTP 服务器；持久会话 |
| **headless** | `dsh --profile headless "task"` | 一次性任务；无服务器；跑完即退出 |
| **ACP** | `dsh --profile acp` | Agent Client Protocol 自动化服务器 |

**web 模式**的插件树：`dsh-base` + `dsh-web-app`（HTTP 路由 + SPA + WebSocket）。

**headless 模式**的插件树：`dsh-base` + `dsh-headless`（直接执行单个任务，不启动服务器）。

**查看实际插件树**：

```bash
dsh --profile web --dump-config
# 打印实际组合的所有 entry，每个都可以被 patch 替换
```

## 6.8 配置覆盖与条件组合

```yaml
# cordis.yml 里的配置行

# 条件禁用（!!js 表达式在 disabled 字段求值）
- name: dsh-fs-sandbox
  disabled: !!js process.env.SANDBOX !== '1'
  config:
    mode: write-only

# 条件配置（!!js 表达式在 config 字段求值）
- name: dsh-llm-deepseek
  config:
    baseURL: !!js process.env.DEEPSEEK_BASE_URL || 'https://api.deepseek.com'
    timeout: 30000
```

**`!!js` 的语义**（来自 [Cordis Primer](../cordis-primer.md#loader-configuration)）：

- `config` 和 `disabled` 字段中的 `!!js` 表达式在插件激活时求值。
- `config` 在声明注入激活后、对插件自己的 `ctx.serviceName` 求值。
- `disabled` 在每次挂载决策时、对 Loader 上下文求值。
- 其他元数据（`name`、`inject`）保持字面值。
- 环境选择插件时用 overlays（覆盖层），不用 `!!js`。

## 6.9 Profile 模板与 HMR

`web` 和 `headless` 是内置的 profile 模板——首次使用时自动初始化到 `$DSH_HOME/profiles/`。

**HMR（热模块替换）**：`watchUserPatches` 监听用户 patch 文件变化，变化时重组用户 patch 层。这意味着你可以在运行中修改 `cordis.patch.yml`，插件树会自动更新——不需要重启。

**注意**：HMR 只重组用户 patch 层，不重组 bundle 层。Bundle 层是启动时固定组合的。

## 深入追问

- 为什么用 JSONL 而不是直接用 SQLite？（提示：可读性 + 调试 + 人类可检查）
- `session/flush` 是 awaited parallel——如果某个持久化插件拒绝写怎么办？（提示：保留事件 + 暂停自动重试）
- 为什么 `SESSION_FORMAT_VERSION` 在 `0`？（提示：pre-release，无兼容承诺，格式可自由变更）
- `--dump-config` 输出的 entry 列表和实际挂载的插件树有什么关系？（提示：include 自己的解析器 + patch 算法离线组合）

下一篇 → [07 · 设计哲学提炼](./07-design-principles.md)
