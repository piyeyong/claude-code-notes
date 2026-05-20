
---

## System Prompt 实现详解 src/constants/prompts.ts/getSystemPrompt

### 函数签名

```typescript
async function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[],
): Promise<string[]>
```

返回 `string[]`（多个 text block），最终由 `buildSystemPromptBlocks()` 逐块附加 `cache_control`。

---

### 三条路径

| 路径 | 条件 | 内容 |
|------|------|------|
| **Simple** | `CLAUDE_CODE_SIMPLE=1` | 一行介绍 + CWD + 日期 |
| **Proactive** | feature flag + proactive active | 精简自主 agent prompt |
| **正常交互** | 默认 | 完整 prompt（下面详析） |

---

### 正常路径的两阶段结构

```
┌─────────────────────────────────────────────┐
│  Static Sections（全局可缓存）               │
│  ─ intro, system, doing tasks, actions,     │
│    tools, tone/style, output efficiency     │
├─ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ────────────┤
│  Dynamic Sections（注册表管理）              │
│  ─ session_guidance, memory, env_info,      │
│    language, output_style, mcp_instructions,│
│    scratchpad, frc, summarize, budget, ...  │
└─────────────────────────────────────────────┘
```

**Static sections**：纯函数，输入不变则输出不变。没有 memoization——因为它们本身就是确定性的，缓存在 API 层的 KV cache 里。

**Dynamic sections**：通过 `systemPromptSection()` / `DANGEROUS_uncachedSystemPromptSection()` 注册：

```typescript
// 缓存版：计算一次，整个 session 复用
systemPromptSection('memory', () => loadMemoryPrompt())

// 非缓存版：每 turn 重新计算，会 bust prompt cache
DANGEROUS_uncachedSystemPromptSection('mcp_instructions', () => ..., 'reason')
```

解析逻辑在 [systemPromptSections.ts:43](src/constants/systemPromptSections.ts#L43)：

```typescript
async function resolveSystemPromptSections(sections) {
  const cache = getSystemPromptSectionCache()
  return Promise.all(sections.map(async s => {
    if (!s.cacheBreak && cache.has(s.name)) return cache.get(s.name)
    const value = await s.compute()
    setSystemPromptSectionCacheEntry(s.name, value)
    return value
  }))
}
```

---

### Boundary Marker 的作用

```typescript
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : [])
```

下游 `splitSysPromptPrefix()`（api.ts）按此标记分割：
- Marker 之前 → `scope: 'global'`（跨 org 共享 KV cache）
- Marker 之后 → `scope: null`（仅本会话 prefix-match）

---

### 并发加载

```typescript
const [skillToolCommands, outputStyleConfig, envInfo] = await Promise.all([
  getSkillToolCommands(cwd),
  getOutputStyleConfig(),
  computeSimpleEnvInfo(model, additionalWorkingDirectories),
])
```

---

### 流程图

I/O 密集操作并行执行，减少 latency。

```
┌─────────────────────────────┐
│     getSystemPrompt()       │
│  tools, model, dirs, mcp    │
└──────────────┬──────────────┘
               │
               ▼
        ┌──────────────┐   Yes   ┌──────────────────┐
        │ SIMPLE mode? ├────────►│ Return 1-line     │
        └──────┬───────┘         │ minimal prompt    │
               │ No              └──────────────────┘
               ▼
   ┌───────────────────────┐
   │   Promise.all([       │
   │     skillToolCommands,│
   │     outputStyleConfig,│
   │     envInfo           │
   │   ])                  │
   └───────────┬───────────┘
               ▼
   ┌───────────────────────┐
   │ settings, enabledTools│
   └───────────┬───────────┘
               ▼
        ┌──────────────┐   Yes   ┌──────────────────────┐
        │ PROACTIVE     ├───────►│ Return proactive      │
        │ path active?  │        │ prompt (no cache      │
        └──────┬────────┘        │ layering)             │
               │ No              └──────────────────────┘
               ▼
   ┌───────────────────────────┐
   │ Build dynamicSections[]   │
   │  - session_guidance       │
   │  - memory                 │
   │  - env_info_simple        │
   │  - language, output_style │
   │  - mcp_instructions ⚠     │◄── DANGEROUS_uncached
   │  - scratchpad, frc        │
   │  + conditional spreads    │
   │    (anchors, budget,brief)│
   └───────────┬───────────────┘
               ▼
   ┌───────────────────────────┐
   │ resolveSystemPromptSections│
   │  cached? → return cache   │
   │  else   → compute + cache │
   │  uncached → always compute│
   └───────────┬───────────────┘
               ▼
   ┌───────────────────────────────────────┐
   │ Assemble final string[]              │
   │                                       │
   │  ┌─ Static (global cache scope) ───┐ │
   │  │ intro                           │ │
   │  │ system                          │ │
   │  │ doingTasks (conditional)        │ │
   │  │ actions                         │ │
   │  │ tools                           │ │
   │  │ toneStyle                       │ │
   │  │ outputEfficiency                │ │
   │  └────────────────────────────────┘ │
   │  ═══ BOUNDARY MARKER ═══            │
   │  ┌─ Dynamic (session cache) ──────┐ │
   │  │ resolvedDynamicSections        │ │
   │  └────────────────────────────────┘ │
   │                                       │
   │  .filter(null)                        │
   └───────────────────┬──────────────────┘
                       ▼
              ┌────────────────┐
              │ Return string[]│
              └────────────────┘

```


---

## Prompt Cache 命中机制详解

Claude Code 使用 Anthropic API 的 **prompt caching** 功能来避免重复处理 system prompt。核心原理是：**API 会缓存标记了 `cache_control` 的文本前缀，只要后续请求的前缀字节完全相同，就能命中缓存。**

---

### 缓存的分层架构

System prompt 被 `splitSysPromptPrefix()` ([src/utils/api.ts:321](src/utils/api.ts#L321)) 拆分成多个 block，每个 block 有不同的缓存策略：

```
API 请求体:
  system: [
    { text: "billing header",      cache_control: null        },  ← 不缓存
    { text: "CLI prefix",          cache_control: {scope:'org'}}, ← org 级缓存
    { text: "静态内容(大块)",       cache_control: {scope:'global'}}, ← 全局缓存 ★
    { text: "动态内容(小块)",       cache_control: null        },  ← 不缓存
  ]
```

---

### 关键设计：BOUNDARY MARKER 分割静态/动态

在 [prompts.ts:573](src/constants/prompts.ts#L573)：

```typescript
return [
  // --- 静态内容 (cacheable) ---
  getSimpleIntroSection(),          // 身份、角色
  getSimpleSystemSection(),         // 系统指令
  getSimpleDoingTasksSection(),     // 编码规则
  getActionsSection(),              // 行为守则
  getUsingYourToolsSection(),       // 工具使用说明
  getSimpleToneAndStyleSection(),   // 语气风格

  // === BOUNDARY MARKER ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),

  // --- 动态内容 (uncacheable / org-scoped) ---
  ...resolvedDynamicSections,       // memory, env, MCP, skills...
]
```

**BOUNDARY 之前**的内容 → `scope: 'global'`（跨用户/跨 org 共享缓存）
**BOUNDARY 之后**的内容 → 不设 cache scope（每次重新处理）

---

### 三种缓存范围 (scope)

| Scope | 含义 | 命中条件 |
|-------|------|----------|
| `global` | 全局共享 | 所有用户相同版本的静态 prompt → 命中 |
| `org` | 组织级 | 同一 org 下请求前缀相同 → 命中 |
| `null` | 不缓存 | 每次重新计算 token |

---

### 缓存命中的具体条件

1. **字节级前缀匹配** — API 从 system blocks 的第一个字节开始比较，遇到不同就停止缓存命中。所以：
   - 静态部分（角色说明、工具使用规则）对所有 Claude Code 用户完全相同 → **global 缓存命中**
   - 动态部分（memory、MCP tools、env info）每用户不同 → 不缓存

2. **`systemPromptSection()` 的会话级 memo** ([systemPromptSections.ts:20](src/constants/systemPromptSections.ts#L20))：
   ```typescript
   // 计算一次，session 内缓存直到 /clear 或 /compact
   if (!s.cacheBreak && cache.has(s.name)) {
     return cache.get(s.name)  // 不重新计算 → 字节不变 → API 缓存命中
   }
   ```
   这确保动态 section 内容在 turn 之间不变 → org 级缓存也能命中。

3. **`DANGEROUS_uncachedSystemPromptSection`** — 标记 `cacheBreak: true` 的 section（如 MCP instructions）每 turn 重新计算，会**破坏缓存**。这就是为什么 MCP 指令迁移到了 delta attachments。

---

### Message 级缓存

除了 system prompt，**messages 数组的尾部**也有缓存标记 ([claude.ts:3078](src/services/api/claude.ts#L3078))：

```
messages: [
  { role: "user", content: [...] },
  { role: "assistant", content: [...], cache_control: {...} },  ← 标记最后一条
  { role: "user", content: "新消息" }  ← 只有这条是新的
]
```

每次请求只有最新的 user message 是"新"内容。之前所有历史消息的 token 通过 cache hit 免费读取（`cache_read_input_tokens`）。

---

### Beta Header Latching — 防止缓存抖动

[claude.ts:1405](src/services/api/claude.ts#L1405)：

```typescript
// Sticky-on latches: 一旦发送某 beta header，整个 session 持续发送
// 防止 mid-session toggle 改变 server 端 cache key
let afkHeaderLatched = getAfkModeHeaderLatched()
let fastModeHeaderLatched = getFastModeHeaderLatched()
let cacheEditingHeaderLatched = getCacheEditingHeaderLatched()
```

beta header 组合是 cache key 的一部分。如果 header 集合变化 → 缓存失效。所以用 latch 确保一旦激活就不回退。

---

### 总结：缓存命中的保障设计

```
保障手段                           目的
─────────────────────────────────────────────────────────
① BOUNDARY 分割静态/动态           静态部分 global 共享，不受动态内容影响
② systemPromptSection memo        动态 section session 内不变 → 字节稳定
③ DANGEROUS_ 标记 volatile         明确标识哪些会破坏缓存
④ MCP → delta attachments         避免 MCP 连接/断开破坏 prompt 缓存
⑤ Beta header latching            session 内 header 集合不变
⑥ Message-level cache_control     历史消息命中缓存，只处理新增部分
⑦ Deferred tools → delta          工具池变化不破坏 system prompt 缓存
```

**实际效果**: 一个典型的多 turn 对话中，约 50-70K tokens 的 system prompt + 历史消息通过 `cache_read_input_tokens` 命中，只需为新增内容付费（约 1/10 成本）。


---

## 架构设计分析

### 好处

| 设计决策 | 好处 |
|----------|------|
| **Static/Dynamic 二分** | Static 部分字节稳定 → API 全局 KV cache 命中率极高。所有用户共享同一 prefix → 巨大的缓存效率 |
| **`systemPromptSection` memoization** | Dynamic 部分在 session 内计算一次即缓存，避免每 turn 重新算（memory 加载、skill 发现等有 I/O） |
| **`DANGEROUS_` 命名** | 明确标识 cache-busting sections，形成 code review 时的"代价可见性"——添加新 uncached section 需要解释 reason |
| **返回 `string[]` 而非拼接字符串** | 下游可以逐块设置不同 `cache_control` scope，实现细粒度缓存策略 |
| **Feature-gated conditional sections** | 编译时 DCE 移除无关代码；运行时不同用户看到不同 prompt，不必维护多套 |
| **Simple mode 早期退出** | `--bare` / CI 环境零开销，不加载 memory/skills/MCP |
| **Proactive 独立路径** | 自主 agent 不需要交互式指令（tone/style、doing tasks），减少 token 浪费 |

### Trade-offs

| 问题 | 影响 |
|------|------|
| **单体函数 + 隐式耦合** | `getSystemPrompt` 知道所有 section 的存在和顺序。新增 section 必须改这个函数，没有注册机制 |
| **Memoization 粒度粗** | `/clear` 和 `/compact` 清除 *所有* section cache。如果只有 memory 变了，其他 section 也需重算（虽然大多很快） |
| **`DANGEROUS_uncached` 只有一个** | 目前只有 `mcp_instructions` 是 uncached。但如果未来多个 section 需要 uncached，每 turn 的 cache bust 影响会叠加，且当前架构没有"部分 cache"的概念 |
| **Boundary 位置硬编码** | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 只有一个分割点。如果需要三级缓存（global / org / session），当前结构不够 |
| **测试困难** | 函数依赖大量全局状态（`getCwd`, `getInitialSettings`, `getSystemPromptSectionCache` 等），单测需要 mock 很多东西 |
| **Prompt 大小不可预测** | 没有 token 预算检查。MCP instructions、memory、skill commands 都是开放长度，极端情况下 system prompt 可能超出预期 |

### 改进建议

1. **注册式 section 架构**：将 sections 改为声明式注册（类似 tool 注册），每个 section 自带 `order`、`cachePolicy`、`scope`，`getSystemPrompt` 只做 resolve + assemble，不硬编码列表。

2. **分层缓存 invalidation**：按 section 名精准失效，而非全局 clear。例如 `/memory` 命令只 invalidate `'memory'` section。

3. **Token budget guard**：在 assemble 完成后检查总 token 数，超限时按优先级截断低优先级 sections（如 numeric_length_anchors、summarize_tool_results）。

4. **多级 boundary**：支持 N 个 boundary marker，映射到不同 cache scope，为未来的 org-level / team-level 共享 prompt 留空间。

5. **MCP instructions delta 模式推广**：当前只有 MCP 用了 delta attachment（避免 cache bust）。其他易变内容（如 memory 更新）也可以用 attachment delta 模式，保持 system prompt 稳定。
