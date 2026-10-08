← 上一篇：[05 · 工具系统与能力 Seam](./05-tools-and-seams.md)

# 06 · 持久化与启动组合

这一篇讲两件事：会话日志如何持久化到磁盘，以及 dsh 运行时如何从配置组合出插件树。

## 6.1 持久化 Seam

`ctx.sessionPersistence`（`SessionPersistence`）是持久化的 Service Definition。它是**句柄式** API：`create(header)` / `open(id, access)` 返回一个 `SessionHandle`，通过它 append / flush / read；另有 `stat(id)` 和 `list()` 做只读头信息观察（不加载事件流）。全部基于 `SessionEvent`，没有平行的持久化事件类型。

当前唯一的后端实现：

| 后端 | 存储 | 特点 |
|---|---|---|
| `dsh-session-persistence-jsonl` | 每会话一个目录，内含版本化代际文件 `session.vN.jsonl[.zstd]` | 追加只逻辑日志；默认 Zstandard 压缩帧；崩溃安全原子写；内核锁租约防止双写 |

后端契约由共享的 `runPersistenceContract` 套件验证（[packages/session/session-persistence/tests/contract.ts](../../packages/session/session-persistence/tests/contract.ts)）。非会话数据（设置、存储域等）走独立的 `storage/` 能力族（`storage-json` / `storage-sqlite`）。

## 6.2 flush 检查点机制

会话日志的 `append()` 是同步的——事件立刻进入内存，不阻塞在 I/O 上。持久化是异步的：

```
Session.append(event)
  │
  ├── 同步：追加到内存日志
  │     └── session/event (emit, fire-and-forget) → UI/SDK
  │
  └── 异步：JSONL 插件深拷贝事件进 per-session 写缓冲
        │
        ├── 首个待处理事件 → 启动有界批处理窗口
        ├── 后续事件 → 加入当前批次（不重置截止）
        └── 窗口过期 → 经串行 mutation chain 持久化批次

session/flush (awaited)
  └── 通知持久化插件"现在是安全检查点"
```

**关键设计**：

- **`session/event` 不阻塞生产者**——持久化插件把事件复制进缓冲，然后异步写入。
- **批处理窗口是有界的**——后续事件加入不重置截止时间，避免"永远在积压"。
- **turn 边界不等待 flush**——语义化的 durability 检查点（模型请求、顶层工具派发、step 完成）由 `dsh-session-checkpoint-policy` 决定何时 flush。
- **失败的后台写保留事件并暂停自动重试**——不会丢数据；下一次 `session/flush` 大声重试。

## 6.3 崩溃恢复

进程崩溃后重启，重载会话日志时可能发现**未关闭的 turn**——有 `turn/start` 但没有 `turn/end`。

**dsh 不截断日志**——原因：

1. 单个 turn 可能很大，截断会丢大量数据。
2. 事件已经持久化了，它们是已发生的事实。

**恢复策略**：用合成 `turn/end {reason:{kind:'interrupted'}}` 关闭孤儿 turn。修复工具（`interruptedTurnClosers`、`ToolCallRecovery`）由 `dsh-session` 提供，未封闭 interrupted 尾部的常规修复是**句柄消费者**（恢复路径）的职责；后端只负责在迁移已封印的历史日志时补上缺失的 interrupted 关闭。

```
崩溃前的日志:
  seq 1: turn/start {turn: 5}
  seq 2: step/start {turn:5, step:1}
  seq 3: user/message
  seq 4: tool/call (没有 tool/result)
  ... (没有 turn/end)

恢复后的日志:
  seq 1-4: 原样保留
  seq 5: tool/result (TOOL_OUTCOME_UNKNOWN 之类的恢复结果)
  seq 6: turn/end {turn:5, reason:{kind:'interrupted'}}  ← 合成
```

**`interrupted` 是唯一非循环发出的 `TurnEndReason`**——它只在崩溃恢复时出现，不会在正常运行中产生。

**恢复流程**：

1. `stat(id)` / `list()`：只读头信息观察，选择每个会话目录的最高代际文件，不加载事件。
2. `open(id, access)`：读打开在内存中解码/迁移，不发布后继代际；**写打开**先编码、校验，再把版本命名的后继代际文件独占发布在未改动的源文件旁边。
3. `ctx.sessions.prepare()`：预留 Session 并提交修复，返回可处置的发布句柄。

## 6.4 SessionHeader 与格式版本 —— 与日志分离存储

`SessionHeader` 是会话的元数据头，与事件日志分离存储：

```ts
import type { SessionId } from '@deepseek-ai/dsh-session'

interface SessionHeader {
  version: 4                   // SESSION_FORMAT_VERSION
  id: SessionId
  createdAt: number
  cwd?: string                 // 工作目录
  parentSession?: SessionId    // 分叉来源
  isSeeded: boolean            // fork 血缘标记（精确继承长度走存储元数据）
  origin?: 'subagent'          // 起源分类
  delegationDepth?: number     // 委托深度
  agentPreset?: string         // 使用的 preset
}
```

**为什么分离**：

- 头信息是会话级的元数据，不是事件——不需要 append-only。
- 头信息需要频繁读（列表会话时），分离存储更高效——`stat`/`list` 只解析头。

**格式版本是发布契约**：`SESSION_FORMAT_VERSION` 当前是 `4`。读到**未来**版本直接拒绝；读到**历史**版本则通过相邻迁移链（`session-format-v0-to-v1` … `v3-to-v4`，每包恰好一步 `vN → vN+1`）解码为当前逻辑事件。已提交的代际文件永不移动、覆盖或删除——写路径把迁移结果发布为新的版本命名代际。版本化的格式历史见 [docs/persistence-changes/](../persistence-changes/README.md)。

## 6.5 Profile 与 Bundle —— 启动组合

dsh 运行时不是一个"编译好的程序"，而是一个**从有序层组合出的插件树**。

```
组合顺序（从下到上，应用在空条目列表上）:
  1. bundle 层（按 profile 声明的顺序）
      ├── dsh-base         （核心：模型/工具/持久化/沙箱/审批）
      └── dsh-web-app / dsh-headless / dsh-acp-app / dsh-sdk-app
  2. profile 的 cordis.patch.yml
  3. home 级 cordis.patch.yml（机器本地偏好，高于 per-profile 层）
  4. --patch overlay（命令行传入）
```

**Profile**：`$DSH_HOME/profiles/<name>/` 下的目录，持有：

- `package.json`：树外插件 `dependencies` + `dsh.profile` manifest（有序 `bundles` 列表）。
- `cordis.patch.yml`：用户自己的 patch 层。

**Bundle**：manifest 声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` 的 npm 包。它是 Cordis 配置行和它们挂载的代码的**分发格式**。`dsh-web-app` 的 patch 是一个数组——除主 patch 外还带 standard / ptc / minimal / cordis 等 preset 层。

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

`boot()` 函数（[packages/boot/app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts)）是启动入口：

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

## 6.7 运行模式

只有 `dsh` profile 启动受支持的 Node 应用（`dsh <name>` 是 `dsh --profile <name>` 的简写）：

| 模式 | 命令 | 特点 |
|---|---|---|
| **web** | `dsh web` | 浏览器界面；HTTP 服务器 + WebSocket；持久会话 |
| **headless** | `dsh headless "task"` | 一次性任务；无服务器；跑完即退出 |
| **acp** | `dsh acp` | Agent Client Protocol 自动化服务器 |
| **sdk** | `dsh sdk` | SDK JSON-RPC 服务器（TypeScript / Python SDK 都选它） |
| **sdk-minimal** | `dsh sdk-minimal` | 单 bundle 完整显式 SDK 树，不叠加 dsh-base |
| **desktop** | （由 Electron 应用独占管理） | 桌面壳内启动 Host + 完整 Web 应用 |

**web 模式**的插件树：`dsh-base` + `dsh-web-app`。服务端由 `packages/host/`（webserver、controller 等）承载，浏览器端是一棵客户端插件树（`packages/client/` 的 `ui-*` 插件 + `apps/web` 前端壳）。

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

`acp`、`web`、`headless`、`sdk`、`sdk-minimal` 是内置的 profile 模板——`--from-default-profile` 首次初始化时把模板的 bundle 列表复制到 `$DSH_HOME/profiles/<name>/`，之后不再跟随模板变化。

**HMR（热重载）**：`dsh-hmr` 插件（`ctx.hmr`）监听 profile patch 文件并串行化配置与插件模块重载。YAML 控制开关：base 为 web 启用 config-only HMR；headless、SDK、ACP 禁用；`sdk-minimal` 不含它。profile patch 可以覆盖这些默认。

## 深入追问

- 为什么写打开要把迁移结果发布为**新的代际文件**，而不是原地升级？（提示：已提交代际不可变 + 崩溃安全 + 旧版本可读性）
- `session/flush` 是 awaited 的——如果持久化插件写入失败（如磁盘满），会发生什么？（提示：保留缓冲 + 暂停自动重试 + flush 大声拒绝）
- 崩溃恢复时用合成 `interrupted` 关闭孤儿 turn——这个合成事件写入日志吗？它会影响 `seq` 编号吗？
- Profile 模板首次使用时初始化——如果模板更新了，已初始化的 profile 会更新吗？（提示：只复制一次，无继承元数据）
- profile 的 `node_modules` 拦截层如何参与模块解析？（提示：[packages/boot/app-boot/src/profile.ts](../../packages/boot/app-boot/src/profile.ts) 的 real-path 祖先链）
- 为什么 `dsh-sdk-minimal` 故意不叠加 `dsh-base`？（提示：一个 bundle 拥有完整显式树）

下一篇 → [07 · 设计哲学提炼](./07-design-principles.md)
