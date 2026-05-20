# Claude Code 源码架构深度分析

> 基于 Claude Code v2.1.88 反编译源码的全模块架构追踪

---

## 目录

1. [整体架构与消息全流程](#1-整体架构与消息全流程)
2. [Query Loop — 核心循环引擎](#2-query-loop--核心循环引擎)
3. [API 层与 SSE 流式处理](#3-api-层与-sse-流式处理)
4. [StreamingToolExecutor — 工具并发执行器](#4-streamingtoolexecutor--工具并发执行器)
5. [Tool Permission — 权限系统](#5-tool-permission--权限系统)
6. [Context 系统 — 上下文构建](#6-context-系统--上下文构建)
7. [Memory 系统 — 跨会话记忆](#7-memory-系统--跨会话记忆)
8. [Session Memory — 会话内记忆](#8-session-memory--会话内记忆)

---

## 1. 整体架构与消息全流程

### 1.1 模块架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLI / IDE Extension                       │
│                          cli.tsx (entrypoint)                    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                       main.tsx (Application Loop)                │
│   ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────────┐  │
│   │ Config   │  │  Auth    │  │  MDM/     │  │  System      │  │
│   │ Loading  │  │  Flow    │  │  Policy   │  │  Prompt      │  │
│   └──────────┘  └──────────┘  └───────────┘  └──────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     REPL.tsx → onQueryImpl()                     │
│   ┌──────────────┐  ┌───────────────┐  ┌─────────────────────┐  │
│   │ getUserCtx() │  │ getSystemCtx()│  │ prependUserContext()│  │
│   └──────┬───────┘  └──────┬────────┘  └──────────┬──────────┘  │
│          └─────────────────┼──────────────────────┘              │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                   query.ts — async function* query()             │
│                                                                  │
│  ┌─────────┐   ┌───────────┐   ┌──────────────┐   ┌──────────┐ │
│  │callModel│──▶│SSE Stream │──▶│Tool Executor │──▶│Next Turn │ │
│  │ (L659)  │   │Processing │   │(permissions, │   │State     │ │
│  └─────────┘   │(L742-835) │   │ execution)   │   │(L1715)   │ │
│                └───────────┘   └──────────────┘   └──────────┘ │
│                                                                  │
│  Recovery: reactive_compact | max_tokens_escalation | hooks      │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                services/api/claude.ts — queryModel()             │
│                                                                  │
│  HTTP POST → SSE Stream → content_block_start/delta/stop         │
│  → AssistantMessage assembly → yield to query loop               │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 一条消息从输入到输出的完整生命周期

```
用户输入
  │
  ▼
┌──────────────────────────────────┐
│ 1. REPL.tsx: onQueryImpl()       │
│    - 获取 userContext (CLAUDE.md, │
│      memory files, currentDate)  │
│    - 获取 systemContext (git      │
│      status, cache breaker)      │
│    - prependUserContext() 注入    │
│      为第一条 <system-reminder>   │
│      user message                │
└────────────┬─────────────────────┘
             ▼
┌──────────────────────────────────┐
│ 2. query.ts: query() 进入循环    │
│    - 初始化 State (10个字段)     │
│    - 检查 blocking_limit        │
│    - 执行 queryCheckpoint()     │
└────────────┬─────────────────────┘
             ▼
┌──────────────────────────────────┐
│ 3. deps.callModel() (L659)       │
│    - 构建 API 请求               │
│    - 发送 HTTP POST 到 Claude    │
│    - 通过 SSE 接收流式响应       │
└────────────┬─────────────────────┘
             ▼
┌──────────────────────────────────┐
│ 4. SSE 事件处理 (L742-835)       │
│    - 逐个 content_block 组装     │
│    - 每个 content_block_stop     │
│      立即 yield AssistantMessage │
│    - 检测 tool_use blocks        │
│      → needsFollowUp = true     │
└────────────┬─────────────────────┘
             ▼
        ┌────┴────┐
        │有tool_use?│
        └────┬────┘
       Yes   │   No
    ┌────────┘    └─────────┐
    ▼                       ▼
┌───────────────┐   ┌───────────────┐
│5a. Permission │   │5b. 退出判断    │
│   检查5层漏斗  │   │  - completed  │
│   → 执行工具  │   │  - stop_hook  │
│   → 收集结果  │   │  - max_turns  │
└───────┬───────┘   └───────────────┘
        ▼
┌───────────────┐
│6. Attachment   │
│   注入 (L1565) │
│   - 文件内容   │
│   - IDE选择    │
│   - Memory预取 │
└───────┬───────┘
        ▼
┌───────────────┐
│7. 构造下一轮   │
│   State (L1715)│
│   → 回到步骤2  │
└───────────────┘
```

### 1.3 调用链总览

```
cli.tsx
  → main.tsx (bootstrap: config, auth, MDM)
    → REPL.tsx (React/Ink terminal UI)
      → onQueryImpl()
        → getUserContext()  ← claudemd.ts: getMemoryFiles() → getClaudeMds()
        → getSystemContext() ← context.ts: getGitStatus()
        → prependUserContext() ← api.ts: 注入 <system-reminder>
        → query()  ← query.ts: AsyncGenerator
          → deps.callModel() → claude.ts: queryModel() → SSE stream
          → StreamingToolExecutor → Tool.execute() → permission check
          → next turn / exit
```

---

## 2. Query Loop — 核心循环引擎

**文件**: `src/query.ts`

### 2.1 设计理念

Query Loop 是 Claude Code 的核心 — 一个 `async function* query()` AsyncGenerator，每次 yield 一个 `QueryYield` 对象供 UI 渲染，通过无限循环实现多轮对话：用户消息 → 模型调用 → 工具执行 → 下一轮。

### 2.2 State 类型 — 循环状态

```typescript
// L555-558
type State = {
  messages: Message[]                        // 完整对话历史
  toolUseContext: ToolUseContext              // 工具执行上下文
  autoCompactTracking: AutoCompactTracking   // 压缩追踪
  turnCount: number                          // 当前轮次
  maxOutputTokensRecoveryCount: number       // max_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean       // 是否已尝试响应式压缩
  pendingToolUseSummary: string | undefined  // 待处理工具摘要
  maxOutputTokensOverride: number | undefined // max_tokens 覆盖值
  stopHookActive: boolean                    // stop hook 是否激活
  transition: { reason: string }             // 状态转换原因
}
```

### 2.3 循环流程详解

```
                    ┌──────────────────────────┐
                    │    query() 入口           │
                    │    初始化 State           │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  while (true) 主循环      │
                    └────────────┬─────────────┘
                                 │
              ┌──────────────────▼──────────────────┐
              │ ① 检查 blocking_limit (L635-648)    │
              │    tokenCount >= blockingLimit?      │
              │    → return { reason: 'blocking_    │
              │       limit' }                       │
              └──────────────────┬──────────────────┘
                                 │ 未超限
              ┌──────────────────▼──────────────────┐
              │ ② callModel() (L659-684)            │
              │    for await (msg of callModel())   │
              │    SSE 流式接收 AssistantMessage     │
              └──────────────────┬──────────────────┘
                                 │
              ┌──────────────────▼──────────────────┐
              │ ③ 消息处理 (L742-835)               │
              │    - backfill tool inputs            │
              │    - withhold recoverable errors     │
              │    - push to assistantMessages[]     │
              │    - 检测 tool_use blocks            │
              │      → needsFollowUp = true          │
              │      → toolUseBlocks.push(block)     │
              └──────────────────┬──────────────────┘
                                 │
                      ┌──────────▼──────────┐
                      │  needsFollowUp?     │
                      └──────┬─────┬────────┘
                        No   │     │  Yes
              ┌──────────────▼┐   ┌▼──────────────────┐
              │ 无工具分支     │   │ 有工具分支          │
              │ (L1062-1357) │   │ (L1360-1620)      │
              └──────────────┘   └────────────────────┘
```

### 2.4 十大退出条件

| # | 条件 | 触发位置 | 说明 |
|---|------|---------|------|
| 1 | `blocking_limit` | L635-648 | token 数超过硬限制，需手动压缩 |
| 2 | `image_error` | L960-975 | 图片处理错误 |
| 3 | `model_error` | L977-997 | API 调用错误（非可恢复） |
| 4 | `aborted_streaming` | L1015-1051 | 流式传输过程中被中止 |
| 5 | `completed` (无工具) | L1085 | end_turn，无 tool_use，正常完成 |
| 6 | `completed` (prompt-too-long 恢复后) | L1166 | 响应式压缩后重新完成 |
| 7 | `stop_hook_prevented` | L1267-1306 | Stop hook 阻止继续 |
| 8 | `aborted_tools` | L1490-1495 | 工具执行被中止 |
| 9 | `hook_stopped` | L1500-1520 | Hook 停止了执行 |
| 10 | `max_turns` | L1704-1710 | 达到最大轮次限制 |

### 2.5 六大继续条件

| # | 条件 | 说明 |
|---|------|------|
| 1 | `collapse_drain_retry` | context-collapse 分阶段排空后重试 |
| 2 | `reactive_compact_retry` | prompt-too-long 触发压缩后重试 |
| 3 | `max_output_tokens_recovery` | max_tokens 命中后恢复（8k→64k 升级） |
| 4 | `stop_hook_blocking` | stop hook 阻塞等待解决后继续 |
| 5 | `token_budget_continuation` | Token 预算允许继续 |
| 6 | `next_turn` | 正常的下一轮对话 |

### 2.6 恢复机制流程

```
API 返回错误
    │
    ├─ prompt_too_long (413)?
    │   ├─ 有未排空的 context-collapse?
    │   │   └─ → collapse_drain_retry
    │   ├─ 未尝试过 reactive compact?
    │   │   └─ → reactive_compact_retry (全量压缩)
    │   └─ 已尝试 → return model_error
    │
    ├─ max_tokens (stop_reason)?
    │   ├─ 首次? → 升级 8k → 64k (ESCALATED_MAX_TOKENS)
    │   ├─ 未达 MAX_OUTPUT_TOKENS_RECOVERY_LIMIT?
    │   │   └─ → 注入 meta message, max_output_tokens_recovery
    │   └─ 超限 → 正常完成
    │
    └─ 其他错误 → return model_error
```

### 2.7 Attachment 注入机制 (L1565-1620)

在工具执行完成后、构造下一轮 State 之前，query loop 通过 `getAttachmentMessages()` 注入额外的上下文：

```typescript
// L1565-1590
const attachmentMessages = getAttachmentMessages(
  null,                          // input: null (非用户直接输入)
  queuedCommandsSnapshot,        // 排队的命令
  [...messagesForQuery, ...assistantMessages, ...toolResults]
)
for await (const msg of attachmentMessages) {
  toolResults.push(msg)          // 注入到 toolResults 中
}

// L1599-1614: Memory 预取消费
const memoryAttachments = filterDuplicateMemoryAttachments(
  memoryPrefetchPromise,
  readFileState
)
// 作为 attachment messages 注入
```

**Attachment 类型**：
- `FileAttachment` — 文件内容
- `selected_lines_in_ide` — IDE 中选中的代码
- `edited_text_file` / `edited_image_file` — 编辑过的文件
- `directory` — 目录列表
- `PDFReference` — PDF 引用
- `CompactFileReference` — 压缩文件引用

---

## 3. API 层与 SSE 流式处理

**文件**: `src/services/api/claude.ts`

### 3.1 SSE (Server-Sent Events) 协议

SSE 是一种基于 HTTP 的单向流式协议。Claude API 通过 SSE 传输模型的增量响应，客户端逐事件接收并组装。

### 3.2 事件处理流程

```
HTTP POST /messages (stream: true)
    │
    ▼
┌────────────────────────────────────────┐
│         SSE Event Stream               │
│                                        │
│  message_start ──────────────────┐     │
│    └─ 保存 partialMessage 元数据  │     │
│       (id, model, usage)        │     │
│                                  ▼     │
│  content_block_start ──────────────┐   │
│    └─ 初始化 contentBlocks[index]  │   │
│       text → ''                   │   │
│       tool_use → { input: '' }    │   │
│       thinking → { thinking: '' } │   │
│                                   ▼   │
│  content_block_delta (多次) ────────┐  │
│    └─ 累积增量数据                  │  │
│       text_delta → text +=         │  │
│       input_json_delta → input +=  │  │
│       thinking_delta → thinking += │  │
│                                    ▼  │
│  content_block_stop ─────────────────┐│
│    └─ 【关键】组装 AssistantMessage  ││
│       从 partialMessage + 单个       ││
│       contentBlock 构建              ││
│       → yield 立即返回给 query loop  ││
│                                      ▼│
│  message_delta ──────────────────────┐│
│    └─ 写回 stop_reason + final usage ││
│       到最后一个已 yield 的 message   ││
│                                      ▼│
│  message_stop ─────────────────────── │
│    └─ 流结束                          │
└────────────────────────────────────────┘
```

### 3.3 关键设计决策：逐块 yield

```typescript
// claude.ts L2171-2211
case 'content_block_stop': {
  const block = contentBlocks[part.index]
  // 组装独立的 AssistantMessage
  const message: AssistantMessage = {
    ...partialMessage,           // 元数据 (id, model, etc.)
    content: [block],            // 只包含当前这一个 block
    role: 'assistant',
  }
  newMessages.push(message)
  yield message                  // 立即 yield，不等其他 blocks
}
```

**为什么不等所有 blocks 完成？**
- 流式 UI 渲染：每个 block 完成就可以显示
- 工具并行执行：tool_use block 一完成就可以开始执行
- 减少延迟：用户更快看到响应

### 3.4 Usage 追踪与成本计算

```typescript
// L1983: 初始 usage
case 'message_start':
  updateUsage(part.message.usage)

// L2214-2256: 最终 usage
case 'message_delta':
  updateUsage(part.usage)
  const costUSD = calculateUSDCost(resolvedModel, usage)
  addToTotalSessionCost(costUSD, usage, options.model)
```

---

## 4. StreamingToolExecutor — 工具并发执行器

**文件**: `src/services/tools/StreamingToolExecutor.ts`

### 4.1 设计理念

StreamingToolExecutor 实现了一个精心设计的并发模型：**只读工具可以并行执行，写工具必须串行执行**，确保文件系统一致性的同时最大化吞吐量。

### 4.2 TrackedTool 生命周期

```
┌─────────┐    addTool()    ┌───────────┐   processQueue()  ┌───────────┐
│ queued  │ ──────────────▶ │ executing │ ───────────────▶  │ completed │
└─────────┘                 └───────────┘                   └─────┬─────┘
                                                                  │
                                                       getCompletedResults()
                                                                  │
                                                            ┌─────▼─────┐
                                                            │  yielded  │
                                                            └───────────┘
```

```typescript
// L21-32
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: 'queued' | 'executing' | 'completed' | 'yielded'
  isConcurrencySafe: boolean      // true = 只读工具（可并行）
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

### 4.3 并发控制逻辑

```typescript
// L129-135: canExecuteTool()
canExecuteTool(isConcurrencySafe: boolean): boolean {
  // 无工具在执行 → 可以执行
  // 或者: 所有正在执行的工具都是只读 AND 候选工具也是只读
  return this.executingCount === 0 ||
    (this.allExecutingAreConcurrencySafe && isConcurrencySafe)
}
```

```
示例执行顺序：

工具队列: [Read₁, Read₂, Write₁, Read₃, Write₂]

时间线:
t0: Read₁ ──▶ 并行
    Read₂ ──▶ 并行
    (Write₁ 等待 — 不是 concurrencySafe)

t1: Read₁ ✓
    Read₂ ✓
    Write₁ ──▶ 串行 (独占)

t2: Write₁ ✓
    Read₃ ──▶ 执行
    (Write₂ 等待)

t3: Read₃ ✓
    Write₂ ──▶ 串行

t4: Write₂ ✓ — 全部完成
```

### 4.4 Promise.race 事件驱动消费

```typescript
// L472-482: getRemainingResults()
async *getRemainingResults() {
  while (hasIncompleteTools()) {
    // 收集当前所有执行中的 Promise
    const executingPromises = this.tools
      .filter(t => t.status === 'executing' && t.promise)
      .map(t => t.promise!)

    // 进度通知的 Promise
    const progressPromise = new Promise<void>(resolve => {
      this.progressAvailableResolve = resolve
    })

    // 竞赛：任何一个完成就唤醒
    await Promise.race([...executingPromises, progressPromise])

    // yield 所有已完成的结果
    yield* this.getCompletedResults()
  }
}
```

**设计精髓**：不用轮询，不用回调，通过 `Promise.race` 实现纯事件驱动的结果消费。任何工具完成或进度更新都会立即唤醒消费者。

### 4.5 三层 Abort 层级

```
┌─────────────────────────────────────┐
│ Layer 1: Query-level AbortController│  ← 整个 query 循环的控制
│   (toolUseContext.abortController)  │
│                                     │
│  ┌──────────────────────────────┐   │
│  │ Layer 2: Sibling Abort       │   │  ← 兄弟工具级别
│  │   (siblingAbortController)   │   │     Bash 错误时取消所有兄弟
│  │                              │   │
│  │  ┌───────────────────────┐   │   │
│  │  │ Layer 3: Tool Abort   │   │   │  ← 单个工具级别
│  │  │  (toolAbortController)│   │   │     权限拒绝时冒泡到 query
│  │  └───────────────────────┘   │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘

取消传播路径:
  Tool abort → 权限拒绝? → 冒泡到 query controller
  Bash error → sibling abort → 取消所有兄弟工具
  用户中止 → query abort → 级联到 sibling → 级联到所有 tools
```

---

## 5. Tool Permission — 权限系统

### 5.1 五层权限漏斗

```
用户请求执行工具
    │
    ▼
┌──────────────────────────────────┐
│ Layer 1: Orchestration           │
│   query loop 决定是否调用工具     │
│   (needsFollowUp 检测)           │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 2: useCanUseTool()         │
│   React hook 层，UI 交互         │
│   处理用户确认/拒绝对话框         │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 3: hasPermissionsToUseTool()│
│   规则匹配引擎                    │
│   遍历所有规则来源，匹配优先级     │
│   policy → flag → user →         │
│   project → local → session →    │
│   command                        │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 4: Tool.checkPermissions() │
│   工具自身的权限校验              │
│   如: Bash 检查命令安全性        │
│   FileWrite 检查路径权限         │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 5: Classifier              │
│   AI 分类器 (可选)                │
│   对高风险操作做额外判断          │
│   如: 检测是否在做恶意操作        │
└──────────────┬───────────────────┘
               ▼
        allow / deny / ask
```

### 5.2 权限模式 (PermissionMode)

| 模式 | 行为 |
|------|------|
| `default` | 标准模式，按规则决定，不匹配则询问用户 |
| `plan` | 只允许只读工具 |
| `acceptEdits` | 自动接受文件编辑，其他操作仍需确认 |
| `bypassPermissions` | 跳过所有权限检查（危险） |
| `dontAsk` | 不匹配则拒绝（非交互模式） |
| `auto` | 自动模式，智能决定 |

### 5.3 规则来源优先级

```
优先级 (高 → 低):
  policy     — 组织级策略 (MDM)
    ▼
  flag       — CLI 参数 (--dangerously-skip-permissions)
    ▼
  user       — 用户级设置 (~/.claude/settings.json)
    ▼
  project    — 项目级设置 (.claude/settings.json)
    ▼
  local      — 本地设置 (.claude/settings.local.json)
    ▼
  session    — 会话级规则 (运行时动态添加)
    ▼
  command    — 命令级规则 (单次命令参数)
```

### 5.4 ToolPermissionContext 结构

```typescript
// Tool.ts L123-138
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource    // 白名单
  alwaysDenyRules: ToolPermissionRulesBySource     // 黑名单
  alwaysAskRules: ToolPermissionRulesBySource      // 必问列表
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
}>
```

---

## 6. Context 系统 — 上下文构建

**文件**: `src/context.ts`, `src/utils/api.ts`

### 6.1 上下文构建流程

```
onQueryImpl() 调用
    │
    ├──▶ getUserContext()          ← memoized
    │     │
    │     ├─ getMemoryFiles()     ← 7 种来源层级加载
    │     │   ├─ Managed (策略)
    │     │   ├─ User (~/.claude/)
    │     │   ├─ Project (CWD 向上遍历 .claude/)
    │     │   ├─ Local (.claude/settings.local.json)
    │     │   ├─ --add-dir (额外目录)
    │     │   ├─ AutoMem (MEMORY.md)
    │     │   └─ TeamMem (团队记忆)
    │     │
    │     ├─ filterInjectedMemoryFiles()
    │     ├─ getClaudeMds()        ← 拼接所有文件为单个字符串
    │     └─ return { claudeMd, currentDate }
    │
    ├──▶ getSystemContext()        ← memoized
    │     │
    │     └─ getGitStatus()
    │         ├─ git branch --show-current
    │         ├─ git rev-parse --abbrev-ref origin/HEAD
    │         ├─ git status --short (截断 2000 字符)
    │         ├─ git log -n5 --oneline
    │         └─ git config user.name
    │         (5 个命令 Promise.all 并行执行)
    │
    └──▶ prependUserContext(messages, context)
          │
          └─ 在 messages 数组开头插入一条 user message:
             <system-reminder>
             # claudeMd
             {CLAUDE.md 内容 + 记忆文件}
             # currentDate
             Today's date is 2026/05/20.
             # gitStatus
             {git 状态信息}
             </system-reminder>
```

### 6.2 getUserContext 详解

```typescript
// context.ts L155-189
const getUserContext = memoize(async (): Promise<{[k: string]: string}> => {
  // 禁用条件检查
  if (env.CLAUDE_CODE_DISABLE_CLAUDE_MDS ||
      (isBareMode() && !hasAddDirs())) {
    return { currentDate: getFormattedDate() }
  }

  // 加载所有记忆文件
  const memoryFiles = await getMemoryFiles()

  // 过滤已注入的记忆
  const filtered = filterInjectedMemoryFiles(memoryFiles)

  // 拼接为字符串
  const claudeMd = getClaudeMds(filtered)

  // 缓存并返回
  setCachedClaudeMdContent(claudeMd)
  return {
    ...(claudeMd && { claudeMd }),     // 条件展开：仅当非空时包含
    currentDate: getFormattedDate(),
  }
})
```

### 6.3 prependUserContext 注入

```typescript
// api.ts L449-474
function prependUserContext(
  messages: Message[],
  context: { [k: string]: string },
): Message[] {
  // 跳过测试环境
  if (process.env.NODE_ENV === 'test') return messages

  // 构建内容字符串
  const contextEntries = Object.entries(context)
    .map(([key, value]) => `# ${key}\n${value}`)
    .join('\n\n')

  // 包装为 <system-reminder> user message
  const contextMessage: UserMessage = {
    role: 'user',
    content: `<system-reminder>\n${contextEntries}\n</system-reminder>`,
  }

  return [contextMessage, ...messages]
}
```

---

## 7. Memory 系统 — 跨会话记忆

**文件**: `src/utils/claudemd.ts`

### 7.1 双轨记忆架构

```
┌─────────────────────────────────────────────────────┐
│                   Memory System                      │
│                                                      │
│  ┌──────────────────┐    ┌──────────────────────┐   │
│  │  Auto Memory     │    │  Session Memory      │   │
│  │  (跨会话持久化)   │    │  (会话内上下文恢复)   │   │
│  │                  │    │                      │   │
│  │  ~/.claude/      │    │  ~/.claude/projects/ │   │
│  │  projects/{proj}/│    │  {cwd}/{sessionId}/  │   │
│  │  memory/         │    │  session-memory/     │   │
│  │                  │    │  summary.md          │   │
│  │  • MEMORY.md     │    │                      │   │
│  │    (索引文件)    │    │  • 9 个 section      │   │
│  │  • *.md          │    │  • forked agent 生成 │   │
│  │    (记忆内容)    │    │  • 压缩时注入恢复    │   │
│  └──────────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 7.2 getMemoryFiles() — 7 层记忆加载

```typescript
// claudemd.ts L790-1075
const getMemoryFiles = memoize(async (): Promise<MemoryFileInfo[]> => {
  const files: MemoryFileInfo[] = []

  // 1. Managed (组织策略 — 始终加载)
  files.push(...await getManagedMemoryFiles())

  // 2. User (用户级 — ~/.claude/CLAUDE.md 等)
  if (isUserSettingsEnabled()) {
    files.push(...await getUserMemoryFiles())
  }

  // 3. Project (项目级 — 从 CWD 向上遍历)
  //    worktree 场景下跳过主仓库目录的 Project 文件
  files.push(...await getProjectMemoryFiles())

  // 4. Local (本地级 — .claude/settings.local.json)
  files.push(...await getLocalMemoryFiles())

  // 5. Additional directories (--add-dir 参数)
  files.push(...await getAddDirMemoryFiles())

  // 6. AutoMem (MEMORY.md — 自动记忆索引)
  if (isAutoMemoryEnabled()) {
    files.push(...await getAutoMemoryFiles())  // 限制 200 行 / 25KB
  }

  // 7. TeamMem (团队记忆 — feature gate)
  if (isTeamMemEnabled()) {
    files.push(...await getTeamMemoryFiles())
  }

  return files
})
```

### 7.3 MEMORY.md 生命周期

```
┌──────────────────────────────────────┐
│ 生成阶段                             │
│                                      │
│ Claude 自身通过 Write 工具生成        │
│ 路径: ~/.claude/projects/{proj}/     │
│       memory/MEMORY.md               │
│                                      │
│ 格式:                                │
│ - [Title](file.md) — 描述           │
│ - [Title](file.md) — 描述           │
│ (限制 200 行 / 25KB)                 │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ 加载阶段                             │
│                                      │
│ getUserContext()                      │
│   → getMemoryFiles()                 │
│     → 第 6 层 AutoMem 加载           │
│       → 读取 MEMORY.md 内容         │
│   → getClaudeMds()                   │
│     → 拼接到 claudeMd 字符串         │
│   → prependUserContext()             │
│     → 作为 <system-reminder> 注入    │
│                                      │
│ 相关记忆预取:                        │
│   Sonnet 从 MEMORY.md 选最相关 5 个  │
│   文件 → 非阻塞加载 → 作为           │
│   attachment 注入到 query loop       │
└──────────────────────────────────────┘
```

---

## 8. Session Memory — 会话内记忆

**文件**: `src/services/SessionMemory/sessionMemory.ts`, `prompts.ts`, `sessionMemoryUtils.ts`, `sessionMemoryCompact.ts`

### 8.1 触发条件决策树

```typescript
// sessionMemory.ts L134-181
function shouldExtractMemory(messages: Message[]): boolean
```

```
shouldExtractMemory(messages)
    │
    ├─ 未初始化?
    │   └─ currentTokens < 10,000? → return false
    │      否则 → markInitialized()
    │
    ├─ tokenGrowth < 5,000 (since last extraction)?
    │   └─ return false (token 阈值未满足)
    │
    ├─ toolCallsSinceLastUpdate >= 3?
    │   └─ Yes → shouldExtract = true (token + tool 双阈值)
    │
    ├─ 最后一轮无 tool calls?
    │   └─ Yes → shouldExtract = true (自然对话断点)
    │
    └─ shouldExtract?
        ├─ Yes → 记录 lastMemoryMessageUuid, return true
        └─ No  → return false
```

**配置阈值**:

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `minimumMessageTokensToInit` | 10,000 | 首次提取最低 token 数 |
| `minimumTokensBetweenUpdate` | 5,000 | 两次提取间最低 token 增长 |
| `toolCallsBetweenUpdates` | 3 | 两次提取间最低工具调用数 |

### 8.2 提取流程

```
post-sampling hook 触发
    │
    ▼
┌─────────────────────────────────┐
│ shouldExtractMemory(messages)?  │──No──▶ 跳过
└──────────────┬──────────────────┘
               │ Yes
               ▼
┌─────────────────────────────────┐
│ sequential() 包装保证串行       │
│                                 │
│ 1. setupSessionMemoryFile()     │
│    创建/读取笔记文件             │
│    路径: ~/.claude/projects/    │
│    {cwd}/{sessionId}/           │
│    session-memory/summary.md    │
│                                 │
│ 2. buildSessionMemoryUpdatePrompt()│
│    构建更新指令                   │
│                                 │
│ 3. runForkedAgent()             │
│    启动 Sonnet 子代理            │
│    工具限制: 仅 FileEdit         │
│    目标: 更新 summary.md        │
│                                 │
│ 4. updateLastSummarizedMessageId│
│    记录进度                     │
└─────────────────────────────────┘
```

### 8.3 笔记模板 — 9 个固定 Section

```markdown
# Session Title
_标题_

# Current State
_当前工作状态 — 压缩恢复后最关键的 section_

# Task specification
_用户的原始任务描述_

# Files and Functions
_涉及的文件、函数、模块_

# Workflow
_工作流程和步骤_

# Errors & Corrections
_遇到的错误和修正_

# Codebase and System Documentation
_代码库和系统文档_

# Learnings
_学到的经验_

# Key results
_关键成果_

# Worklog
_工作日志_
```

每个 section 限制 2000 tokens。总文件限制 12,000 tokens。

### 8.4 权限限制 — createMemoryFileCanUseTool

```typescript
// sessionMemory.ts L460-481
function createMemoryFileCanUseTool(memoryPath: string): CanUseToolFn {
  return async (tool, input) => {
    if (tool.name === FILE_EDIT_TOOL_NAME &&
        input.file_path === memoryPath) {
      return { behavior: 'allow' }    // 仅允许编辑笔记文件
    }
    return {
      behavior: 'deny',
      message: 'Only file edits on the memory file are allowed'
    }
  }
}
```

### 8.5 Compaction 集成

```
Context 接近上限 → 触发 autocompact
    │
    ▼
┌─────────────────────────────────┐
│ trySessionMemoryCompaction()     │
│ (sessionMemoryCompact.ts L514)  │
│                                 │
│ 1. waitForSessionMemoryExtraction()
│    等待最多 15 秒               │
│                                 │
│ 2. 读取 session memory 笔记     │
│                                 │
│ 3. 计算保留消息边界              │
│    - 找到 lastSummarizedId 位置  │
│    - 至少保留 10K tokens         │
│    - 至少保留 5 个 text blocks   │
│    - 最多保留 40K tokens         │
│                                 │
│ 4. truncateSessionMemoryForCompact()
│    截断超长 section (>8000 字符) │
│                                 │
│ 5. 注入截断后的笔记到            │
│    post-compact 消息中           │
│                                 │
│ 6. 验证 post-compact token 数   │
│    超过阈值? → 回退 legacy 压缩  │
└─────────────────────────────────┘

Sequence:
  autocompact trigger
    → waitForSessionMemoryExtraction() [max 15s]
    → read summary.md
    → calculateMessagesToKeepIndex()
    → truncateSessionMemoryForCompact()
    → inject into compacted messages
    → validate threshold
    → return CompactionResult | null (fallback)
```

### 8.6 /summary 命令 — 手动触发

```typescript
// sessionMemory.ts L387-453
async function manuallyExtractSessionMemory(
  messages: Message[],
  toolUseContext: ToolUseContext,
): Promise<ManualExtractionResult> {
  // 绕过所有阈值检查
  // 直接执行 setupSessionMemoryFile → runForkedAgent
  // 返回 { success, memoryPath?, error? }
}
```

### 8.7 自定义模板

用户可以在 `~/.claude/session-memory/config/` 下放置自定义文件：
- `template.md` — 自定义 section 模板
- `prompt.md` — 自定义提取指令

```typescript
// prompts.ts L86-104
async function loadSessionMemoryTemplate(): Promise<string> {
  const customPath = join(homedir(), '.claude/session-memory/config/template.md')
  try {
    return await readFile(customPath, 'utf-8')
  } catch {
    return DEFAULT_SESSION_MEMORY_TEMPLATE
  }
}
```

---

## 附录：关键 TypeScript 模式

### A.1 条件属性展开

```typescript
// 仅当 claudeMd 非空时才包含该属性
return {
  ...(claudeMd && { claudeMd }),
  currentDate: getFormattedDate(),
}
```

### A.2 AsyncGenerator 循环引擎

```typescript
async function* query(initialState: State): AsyncGenerator<QueryYield> {
  let state = initialState
  while (true) {
    // ... 处理逻辑
    yield { type: 'update', ... }  // UI 渲染
    // return 退出 或 state = nextState 继续
  }
}
```

### A.3 Memoize 缓存模式

```typescript
const getUserContext = memoize(async () => { ... })
// 缓存结果直到显式清除
// 通过 getUserContext.cache.clear?.() 清除
```

### A.4 sequential() 串行保护

```typescript
const extractSessionMemory = sequential(async (context) => {
  // 保证同一时间只有一个提取任务在运行
})
```

---

*文档生成时间: 2026-05-20*
*基于 Claude Code v2.1.88 反编译源码*
