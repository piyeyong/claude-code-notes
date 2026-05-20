
---

## Query Loop `query()` 实现详解

### 总体结构

[query.ts:219](src/query.ts#L219) 是一个 **AsyncGenerator**，yield 事件给上层（REPL / SDK）消费。核心是 `queryLoop()`（line 241）内的 `while(true)` 循环。

```
query()
  └─ yield* queryLoop()
       └─ while(true) {
            1. Compact（snip → microcompact → collapse → autocompact）
            2. 调用 API（callModel streaming）
            3. 收集 tool_use blocks
            4. 执行工具
            5. 注入 attachments
            6. 决定：continue（有 tool_use）or return（完成）
          }
```

---

### 每次迭代的六阶段

#### 阶段 1：Compact（lines 396-543）

```
messages → snip → microcompact → context_collapse → autocompact → messagesForQuery
```

逐级压缩，每一级独立判断是否触发。顺序固定：snip 去掉远古消息、microcompact 压缩工具结果、collapse 折叠中间段、autocompact 做全量摘要。

#### 阶段 2：API 调用（lines 654-863）

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt, tools, signal, options: { model, ... }
})) {
  // 流式收集 assistant messages 和 tool_use blocks
  if (streamingToolExecutor) {
    streamingToolExecutor.addTool(toolBlock)  // 边流边启动工具
  }
}
```

关键：**StreamingToolExecutor** 在 API 还在流式输出时就开始并行执行已完成的 tool_use block。

#### 阶段 3：错误恢复（lines 1062-1256）

当 `needsFollowUp === false`（模型没有请求工具）时进入：
- **prompt-too-long** → context collapse drain → reactive compact → 报错
- **max_output_tokens** → escalate 到 64k → 注入 "resume" 消息继续 → 报错
- **stop hooks** → 执行用户 hook，hook 可阻止结束或注入错误让模型重试

#### 阶段 4：工具执行（lines 1360-1408）

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()  // 拿剩余的
  : runTools(toolUseBlocks, ...)                  // 传统串行/并行执行

for await (const update of toolUpdates) {
  yield update.message    // tool_result 发给 REPL 显示
  toolResults.push(...)   // 收集用于下次 API 调用
}
```

#### 阶段 5：Attachments 注入（lines 1580-1628）

工具执行完后，注入额外上下文：
- **Queued commands**（用户在 AI 思考时输入的命令）
- **Memory prefetch**（相关记忆文件）
- **Skill discovery prefetch**（推荐的 skills）
- **File change notifications**（其他工具修改的文件变更）

#### 阶段 6：循环决策（lines 1704-1728）

```typescript
if (maxTurns && nextTurnCount > maxTurns) return { reason: 'max_turns' }

state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  turnCount: nextTurnCount,
  transition: { reason: 'next_turn' },
  ...
}
// continue → while(true) 下一次迭代
```

---

### Tool / MCP / Skill / SubAgent 如何接入

所有这些在 `query()` 看来都是 **统一的 Tool**：

| 类型 | 实现 | query 如何看待 |
|------|------|---------------|
| **普通 Tool**（Bash, FileRead 等） | `src/tools/*/` 实现 `Tool` 基类 | `tool_use` block → `runTools()` 执行 |
| **MCP Tool** | MCP client 注册为 Tool，运行时通过 JSON-RPC 调用远程 server | 同上——对 query 透明 |
| **Skill** | `SkillTool` 接收 skill name → 加载 skill prompt → 注入到消息中 | `tool_use` block name=`Skill` |
| **SubAgent** | `AgentTool` → fork context → 递归调用 `query()` | `tool_use` block name=`Agent` → 内部创建新 `query()` generator |

```
query() [main thread]
  → model returns tool_use: { name: "Agent", input: { prompt: "..." } }
    → AgentTool.execute()
      → 创建新的 toolUseContext + messages
      → 调用 query() [sub-agent, 递归]
        → 子 agent 有自己的 while(true) 循环
      → 返回结果作为 tool_result
```

**StreamingToolExecutor** 的并发策略（[toolOrchestration.ts:26](src/services/tools/toolOrchestration.ts#L26)）：

```typescript
for (const { isConcurrencySafe, blocks } of partitionToolCalls(toolUseMessages)) {
  if (isConcurrencySafe) runToolsConcurrently(blocks)  // read-only 工具并行
  else runToolsSerially(blocks)                         // 有副作用的串行
}
```

---

## 架构分析

### 设计优势

| 决策 | 好处 |
|------|------|
| **AsyncGenerator 模式** | 调用者可以逐事件消费（流式 UI 更新）、可以 `.return()` 中断、可以 `for await` 自然结束。比 callback 或 EventEmitter 更可组合 |
| **统一 Tool 抽象** | MCP、Skill、SubAgent 全部是 Tool。`query()` 无需知道工具类型，新增工具零改动 |
| **StreamingToolExecutor** | API 流式输出时提前执行已完成的 tool block，减少 latency（I/O 与计算重叠） |
| **State 对象 + continue** | 每个 `continue` 点构造完整 `State`，清晰表达"从什么状态重新开始"。比 N 个 `let` 变量赋值更不易出错 |
| **多级压缩管道** | 每级独立、按需触发。轻量的 snip/microcompact 几乎每次都跑（便宜），重量的 autocompact 只在阈值触发（一次 API 调用） |
| **Reactive compact（后置）** | 不提前预测 token 数，让 API 自己报 413 → 紧急压缩 → 重试。简单且精确 |
| **Memory/Skill prefetch** | prefetch 在工具执行期间并行完成，到注入时已 settled，零额外延迟 |
| **turnCount + maxTurns** | Agent SDK 需要限制 agent 执行深度，通过简单计数实现 |

### Trade-offs

| 问题 | 影响 |
|------|------|
| **单一 while(true) 循环 1500 行** | 可读性差，8 个 `continue` 出口散布各处（collapse_drain、reactive_compact、max_output_recovery、stop_hook、token_budget、model_fallback...），心智模型复杂 |
| **State 对象 vs 类型安全** | 所有 continue 点手动构造 State，容易漏设字段或设错。每新增一个 state 字段需要改 7+ 处 |
| **压缩管道硬编码顺序** | snip → MC → collapse → autocompact 是固定的。如果想改顺序或并行执行多种压缩，需要重写 |
| **Generator 不可 clone/fork** | SubAgent 递归调用 `query()` 创建新 generator，但主线程的 generator 不能 fork。如果需要"投机执行两个分支"，当前架构不支持 |
| **`deps` 注入仅部分** | `deps.callModel` 和 `deps.microcompact` 可注入，但 snip、collapse、autocompact、hooks 都是直接 import，测试时需要 mock 模块 |
| **Tool 执行与 API 调用耦合在同一循环** | StreamingToolExecutor 提升了性能，但也增加了复杂度（fallback 时需 discard、abort 时需 drain remaining） |
| **Attachment 注入时机固定** | 总是在工具执行后、下次 API 调用前。如果某些 attachment（如 memory）需要更早注入（影响工具决策），当前做不到 |
| **无背压机制** | Generator yield 到 REPL 的 `onQueryEvent` 是同步消费。如果 UI 渲染慢，没有背压信号让 query 暂停 |

### 改进建议

1. **拆分循环体为 pipeline stages**：将 while(true) 内部拆成独立函数（`compressStage()`, `callModelStage()`, `executeToolsStage()`, `attachmentStage()`, `decisionStage()`），每个返回 `State | Terminal`，主循环只做 dispatch。

2. **状态机显式化**：当前的 `transition.reason` 已经是 proto-state-machine。可以用 discriminated union 明确所有合法转换，编译期保证不会漏处理某个 continue 路径。

3. **压缩管道可配置化**：将压缩阶段定义为有序数组 `CompactionStage[]`，每个 stage 自带 `shouldRun()` 和 `run()`。方便 A/B 测试不同压缩策略或顺序。

4. **完整的 DI**：将 snip、collapse、autocompact、hooks 等全部通过 `deps` 注入，消除对全局 import 的直接依赖，提升可测试性。

5. **背压 / 批量 yield**：在高频 yield 场景（streaming token events），可以引入微批次（每 N ms flush 一次），减少 React 重渲染次数。

6. **SubAgent 资源限制**：当前 SubAgent 递归调用 `query()` 没有全局深度/并发限制（只有 `maxTurns`）。建议加入全局并发计数器，防止 agent 无限 fork。

7. **将 "recovery" 逻辑抽离为中间件**：prompt-too-long recovery、max-output-tokens recovery、model fallback 都是"出错 → 变换 state → retry"的模式，可以统一为 `RecoveryMiddleware[]`，循环体只需 `for (const mw of recoveries) { if (mw.matches(error)) state = mw.recover(state); continue }`。



# 附录

## Compact详解

Compact在每次 API 调用前执行，由4个**顺序阶段**组成，每个阶段可能同时运行：

---

### 阶段 1: SNIP（~L400-410）

**Feature gate**: `HISTORY_SNIP`

裁剪最旧的对话历史，设立 compact boundary。记录 `snipTokensFreed`（释放的 token 数），传递给后续 autocompact 做阈值修正。

---

### 阶段 2: MICROCOMPACT（~L412-426）

**无 feature gate**，始终运行。

压缩特定 tool call 的结果内容。若 `CACHED_MICROCOMPACT` 启用，boundary 消息**延迟到 API 流式响应结束后**（~L870-892）才 yield，此时用 API 返回的实际 `cache_deleted_input_tokens` 填充准确的 token 数。

---

### 阶段 3: CONTEXT_COLLAPSE（~L440-447）

**Feature gate**: `CONTEXT_COLLAPSE`

生成消息的"折叠视图"——不修改实际消息数组，而是通过 commit log 在读取时投影。折叠在 autocompact 之前运行，若折叠后 token 已低于阈值，autocompact 变成 no-op。

---

### 阶段 4: AUTOCOMPACT（~L453-543）

**始终调用**，接收前三步处理后的消息。

- **成功**：生成摘要消息，重置 tracking（`compacted: true, turnCounter: 0, consecutiveFailures: 0`），yield 新消息，用压缩后的消息继续查询
- **失败**：传播 `consecutiveFailures`（断路器模式），不替换消息，留给后续 reactive compact 处理

---

### 错误恢复路径

| 路径 | 触发条件 | 行为 |
|------|---------|------|
| `collapse_drain_retry` | 413 错误 + context_collapse 有待处理的折叠 | 先释放折叠内容再重试 |
| `reactive_compact_retry` | 413 错误 + autocompact 失败 | 运行紧急压缩，`hasAttemptedReactiveCompact` 防循环 |
| `max_output_tokens_escalate` | max_output_tokens 命中 | 从 8K 升级到 64K |

### 流程总结

```
messages → getMessagesAfterCompactBoundary()
  → [SNIP] → snipTokensFreed
    → [MICROCOMPACT] → 延迟 boundary
      → [CONTEXT_COLLAPSE] → 投影折叠视图
        → [AUTOCOMPACT] → 成功则替换消息 / 失败则继续
          → API 调用
            → yield 延迟的 microcompact boundary
            → 错误? → reactive_compact / collapse_drain / escalate
```

**关键设计**：4 个阶段**不互斥**，可在同一 turn 内全部执行。`snipTokensFreed` 跨阶段传递确保 autocompact 阈值准确。`hasAttemptedReactiveCompact` 防止 reactive compact 死循环（reset 只在 `next_turn` 正常流转时发生）。


## StreamingToolExecutor 详解

定义在 [StreamingToolExecutor.ts:40](src/services/tools/StreamingToolExecutor.ts#L40)，实现**在 API 流式响应过程中边接收 tool_use 块边执行工具**的并发控制器。

---

### 核心数据结构

每个工具被包装为 `TrackedTool`（L21-32）：

```
id, block, assistantMessage → 工具标识
status: queued → executing → completed → yielded → 生命周期
isConcurrencySafe → 是否可并行
results[] → 执行结果（缓冲，按序 yield）
pendingProgress[] → 进度消息（立即 yield）
contextModifiers[] → 上下文修改器
```

---

### 并发控制模型（L129-134）

`canExecuteTool(isConcurrencySafe)` 的规则：

| 当前执行中的工具 | 新工具类型 | 能否执行 |
|---|---|---|
| 无 | 任意 | **可以** |
| 全是 concurrency-safe | concurrency-safe | **可以** |
| 有任何 non-concurrent | 任意 | **不行** |
| 全是 concurrency-safe | non-concurrent | **不行** |

即：**读只工具可并行，写工具独占执行**。

---

### 工具添加流程 `addTool()`（L76-124）

1. 查找工具定义，不存在则直接生成错误结果标记为 `completed`
2. 解析输入，通过 `isConcurrencySafe()` 判断并发安全性
3. 创建 `TrackedTool`，状态为 `queued`
4. 调用 `processQueue()` 尝试立即启动

---

### 队列处理 `processQueue()`（L140-151）

遍历所有 `queued` 工具：
- 满足并发条件 → 执行
- 不满足且是 non-concurrent → **break**（保序，阻塞后续工具）
- 不满足但是 concurrent → 跳过，继续检查后面的

---

### 工具执行 `executeTool()`（L265-405）

核心流程：

1. **状态切换**：`queued` → `executing`，注册到 `inProgressToolUseIDs`
2. **中止检查**：执行前检查是否已被中止（兄弟错误/用户中断/流式回退），若是则生成合成错误消息
3. **创建子 AbortController**（L301-318）：
   - 继承自 `siblingAbortController`
   - abort 事件**冒泡到父控制器**（除非是 sibling_error 或已 discarded），确保权限拒绝等场景能终止整个 turn
4. **消费 `runToolUse()` 生成器**（L332-382）：
   - 每次迭代检查中止原因
   - 进度消息 → `pendingProgress`（立即通知等待者）
   - 结果消息 → `messages` 缓冲区
   - **仅 Bash 工具错误会取消兄弟**（L359-363）：`hasErrored = true`，abort siblingAbortController。其他工具（Read、WebFetch 等）互相独立
5. **完成**：状态 → `completed`，触发 `processQueue()` 推进队列

---

### 结果产出

**两种获取方式**：

**`getCompletedResults()`**（L412-440）— 同步、非阻塞：
- 按工具添加顺序遍历
- 先 yield 所有 `pendingProgress`（进度不等待）
- `completed` 的工具 yield 结果后标记为 `yielded`
- 遇到正在执行的 non-concurrent 工具 → **break**（保序）

**`getRemainingResults()`**（L453-490）— 异步、阻塞：
- `while(hasUnfinishedTools())` 循环
- 用 `Promise.race([...executingPromises, progressPromise])` 等待任一工具完成或有进度
- 每轮 yield 所有已完成结果

---

### 中止机制（3 层）

```
toolUseContext.abortController  ← 用户中断（ESC）
  └─ siblingAbortController     ← Bash 错误级联
       └─ toolAbortController   ← 单工具级（权限拒绝等）
```

| 场景 | 触发 | 效果 |
|------|------|------|
| 用户按 ESC | `abortController.abort('interrupt')` | `interruptBehavior='cancel'` 的工具收到合成拒绝消息 |
| Bash 工具报错 | `siblingAbortController.abort('sibling_error')` | 所有并行兄弟工具收到"cancelled: parallel tool call errored" |
| 流式回退 | `discard()` 被调用 | 所有待处理工具收到"streaming fallback"错误 |
| 权限拒绝 | `toolAbortController.abort()` | 冒泡到 `abortController`，终止整个 turn |

---

### 在 query.ts 中的使用

```
API 流式响应中:
  收到 tool_use 块 → executor.addTool(block, msg)  // 边收边执行
  每次流式事件后 → executor.getCompletedResults()   // 边执行边yield

API 流式结束后:
  executor.getRemainingResults()  // 等待所有未完成工具
  或 runTools()（非流式回退）
```

**关键设计**：工具在 API 还在输出文本/思考时就开始执行，消除了"等 API 响应完再执行"的串行等待。进度消息实时推送到 UI，结果按序缓冲保证确定性输出。



## Tool Permission 检查的逻辑和层次

整个权限系统是一个**5 层漏斗**，从外到内逐层过滤：

```
┌─────────────────────────────────────────────┐
│ Layer 1: Tool Orchestration (入口)           │
│   toolOrchestration.ts → runToolUse()       │
│   对每个 tool_use block 调用 canUseTool()     │
├─────────────────────────────────────────────┤
│ Layer 2: useCanUseTool Hook (协调层)         │
│   构建 PermissionContext                     │
│   调用 hasPermissionsToUseTool()             │
│   根据结果分发到 UI/自动/拒绝                  │
├─────────────────────────────────────────────┤
│ Layer 3: hasPermissionsToUseTool (核心决策)   │
│   权限模式判断 → 规则匹配 → 工具自身检查        │
├─────────────────────────────────────────────┤
│ Layer 4: Tool.checkPermissions (工具自检)     │
│   每个工具自己的权限逻辑（路径、命令分类等）      │
├─────────────────────────────────────────────┤
│ Layer 5: Bash Classifier / Safety (安全层)   │
│   AI 分类器、危险模式检测、沙箱规则             │
└─────────────────────────────────────────────┘
```

---

### Layer 1: 入口 — Tool Orchestration

[toolOrchestration.ts](src/services/tools/toolOrchestration.ts) 和 [toolExecution.ts](src/services/tools/toolExecution.ts)

- `runTools()` 先用 `partitionToolCalls()` 把工具分成**只读并发批次**和**写操作串行批次**
- 对每个 tool_use block 调用 `canUseTool(tool, input, context, ...)`
- 这是权限检查的唯一入口

---

### Layer 2: useCanUseTool Hook — 协调分发

[useCanUseTool.tsx](src/hooks/useCanUseTool.tsx)

```
canUseTool(tool, input, context, assistantMessage, toolUseID)
  ↓
createPermissionContext()  ← 构建日志/持久化/abort 等上下文
  ↓
hasPermissionsToUseTool()  ← 核心决策
  ↓
结果是什么？
  allow → 直接放行，记录日志
  deny  → 拒绝，记录原因
  ask   → 分三种情况：
    ├─ coordinator worker → handleCoordinatorPermission()
    ├─ swarm worker → handleSwarmWorkerPermission()
    └─ 主线程 → 等待 speculative classifier（2秒）
                → 仍需确认 → handleInteractivePermission() → 弹出 UI
```

---

### Layer 3: hasPermissionsToUseTool — 核心决策引擎

[permissions.ts](src/utils/permissions/permissions.ts)

决策按优先级依次检查：

| 优先级 | 检查项 | 结果 |
|--------|--------|------|
| 1 | **Permission Mode** | `plan` → deny, `bypassPermissions` → allow, `dontAsk` → deny |
| 2 | **Deny 规则** | 匹配 `alwaysDenyRules` → deny |
| 3 | **Allow 规则** | 匹配 `alwaysAllowRules` → allow |
| 4 | **Ask 规则** | 匹配 `alwaysAskRules` → ask（强制弹窗） |
| 5 | **Tool.checkPermissions()** | 工具自身的权限判断 |
| 6 | **Auto 模式** | ask 结果 + auto 模式 → 调 AI classifier 自动决策 |

**规则来源**（按覆盖优先级）：

```
policySettings     ← 组织策略（最高优先级）
flagSettings       ← CLI --allow/--deny 参数
userSettings       ← ~/.claude/settings.json
projectSettings    ← .claude/settings.json
localSettings      ← .claude/settings.local.json
session            ← 会话内临时规则
command            ← /permissions 交互设置
```

---

### Layer 4: Tool.checkPermissions — 工具自检

[Tool.ts:500-503](src/Tool.ts#L500-L503)

每个工具实现自己的 `checkPermissions(input, context)`，返回 `allow` / `ask` / `deny`：

| 工具 | 自检逻辑 |
|------|---------|
| **BashTool** | 解析命令 → 子命令拆分 → 逐个匹配规则 → 路径验证 → 危险模式检测 |
| **FileEditTool / FileWriteTool** | 路径在允许目录内？文件是否存在？ |
| **FileReadTool / GlobTool / GrepTool** | 路径验证（[pathValidation.ts](src/utils/permissions/pathValidation.ts)） |
| **MCPTool** | 返回 `passthrough`，委托通用权限框架处理 |
| **默认** | `allow`（大多数内置工具无需额外检查） |

---

### Layer 5: 安全分类器（Bash 专用）

[bashClassifier.ts](src/utils/permissions/bashClassifier.ts)

这是 **Anthropic 内部专属功能**（开源版 stub 返回 `disabled`）：

```
Bash 命令
  ↓
startSpeculativeClassifierCheck()  ← API 流式时就开始异步分类
  ↓
useCanUseTool 等待最多 2 秒
  ↓
高置信度 allow → 自动放行，跳过弹窗
低置信度 / deny → 仍弹窗让用户决定
```

另外还有：
- [dangerousPatterns.ts](src/utils/permissions/dangerousPatterns.ts) — 硬编码的危险命令模式（`rm -rf /`、`git push --force` 等）
- [sedConstraints.ts](src/utils/permissions/sedConstraints.ts) — sed 命令的特殊约束
- [shellRuleMatching.ts](src/utils/permissions/shellRuleMatching.ts) — shell 命令的 glob 规则匹配

---

### 权限决策结果类型

[types/permissions.ts](src/types/permissions.ts#L241-L246)

```typescript
PermissionDecision =
  | { behavior: 'allow', updatedInput?, decisionReason? }
  | { behavior: 'ask',   message, suggestions?, pendingClassifierCheck? }
  | { behavior: 'deny',  message, decisionReason }
```

`decisionReason` 记录**为什么**做出该决定：

| Reason Type | 含义 |
|-------------|------|
| `rule` | 匹配了某条 allow/deny 规则 |
| `mode` | 当前权限模式决定（plan/bypass/dontAsk） |
| `classifier` | AI 分类器判定 |
| `hook` | 用户配置的 hook 决定 |
| `safetyCheck` | 安全检查拦截 |
| `sandboxOverride` | 沙箱覆盖 |
| `workingDir` | 工作目录违规 |

---

### 一句话总结

权限系统是 **Mode → Rules → Tool 自检 → Classifier** 的层级漏斗：先看全局模式，再匹配用户规则，再让工具自己检查，最后对 Bash 等高风险工具用 AI 分类器兜底。每层都可以直接 allow/deny 短路，不必走完全部层级。