
---

## System Prompt Implementation Deep Dive: src/constants/prompts.ts/getSystemPrompt

### Function Signature

```typescript
async function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[],
): Promise<string[]>
```

Returns `string[]` (multiple text blocks), which are then each appended with `cache_control` by `buildSystemPromptBlocks()`.

---

### Three Paths

| Path | Condition | Content |
|------|-----------|---------|
| **Simple** | `CLAUDE_CODE_SIMPLE=1` | One-line intro + CWD + date |
| **Proactive** | feature flag + proactive active | Compact autonomous agent prompt |
| **Normal interaction** | default | Full prompt (detailed below) |

---

### Two-Phase Structure of the Normal Path

```
┌─────────────────────────────────────────────┐
│  Static Sections (globally cacheable)        │
│  ─ intro, system, doing tasks, actions,     │
│    tools, tone/style, output efficiency     │
├─ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ────────────┤
│  Dynamic Sections (registry-managed)        │
│  ─ session_guidance, memory, env_info,      │
│    language, output_style, mcp_instructions,│
│    scratchpad, frc, summarize, budget, ...  │
└─────────────────────────────────────────────┘
```

**Static sections**: Pure functions — same input always produces same output. No memoization needed because they are deterministic; caching happens at the API layer via KV cache.

**Dynamic sections**: Registered via `systemPromptSection()` / `DANGEROUS_uncachedSystemPromptSection()`:

```typescript
// Cached version: computed once, reused for the entire session
systemPromptSection('memory', () => loadMemoryPrompt())

// Uncached version: recomputed every turn, will bust the prompt cache
DANGEROUS_uncachedSystemPromptSection('mcp_instructions', () => ..., 'reason')
```

Resolution logic is in [systemPromptSections.ts:43](src/constants/systemPromptSections.ts#L43):

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

### Role of the Boundary Marker

```typescript
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : [])
```

Downstream `splitSysPromptPrefix()` (api.ts) splits on this marker:
- Before the marker → `scope: 'global'` (KV cache shared across orgs)
- After the marker → `scope: null` (session-level prefix-match only)

---

### Concurrent Loading

```typescript
const [skillToolCommands, outputStyleConfig, envInfo] = await Promise.all([
  getSkillToolCommands(cwd),
  getOutputStyleConfig(),
  computeSimpleEnvInfo(model, additionalWorkingDirectories),
])
```

---

### Flow Diagram

I/O-intensive operations are executed in parallel to reduce latency.

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

## Prompt Cache Hit Mechanism Deep Dive

Claude Code uses Anthropic API's **prompt caching** feature to avoid reprocessing the system prompt. The core principle is: **the API caches text prefixes marked with `cache_control`, and any subsequent request whose prefix bytes are identical will result in a cache hit.**

---

### Layered Cache Architecture

The system prompt is split by `splitSysPromptPrefix()` ([src/utils/api.ts:321](src/utils/api.ts#L321)) into multiple blocks, each with a different caching strategy:

```
API request body:
  system: [
    { text: "billing header",       cache_control: null        },  ← not cached
    { text: "CLI prefix",           cache_control: {scope:'org'}}, ← org-level cache
    { text: "static content (large)", cache_control: {scope:'global'}}, ← global cache ★
    { text: "dynamic content (small)", cache_control: null      },  ← not cached
  ]
```

---

### Key Design: BOUNDARY MARKER Splits Static/Dynamic

In [prompts.ts:573](src/constants/prompts.ts#L573):

```typescript
return [
  // --- Static content (cacheable) ---
  getSimpleIntroSection(),          // identity, role
  getSimpleSystemSection(),         // system instructions
  getSimpleDoingTasksSection(),     // coding rules
  getActionsSection(),              // behavioral guidelines
  getUsingYourToolsSection(),       // tool usage instructions
  getSimpleToneAndStyleSection(),   // tone and style

  // === BOUNDARY MARKER ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),

  // --- Dynamic content (uncacheable / org-scoped) ---
  ...resolvedDynamicSections,       // memory, env, MCP, skills...
]
```

Content **before BOUNDARY** → `scope: 'global'` (cache shared across users/orgs)  
Content **after BOUNDARY** → no cache scope set (reprocessed every time)

---

### Three Cache Scopes

| Scope | Meaning | Hit Condition |
|-------|---------|---------------|
| `global` | Globally shared | Same static prompt across all users on the same version → hit |
| `org` | Organization-level | Requests with identical prefix within the same org → hit |
| `null` | Not cached | Tokens recomputed every time |

---

### Specific Conditions for Cache Hits

1. **Byte-level prefix matching** — The API compares system blocks from the first byte; it stops matching as soon as a difference is found. Therefore:
   - Static sections (role description, tool usage rules) are identical for all Claude Code users → **global cache hit**
   - Dynamic sections (memory, MCP tools, env info) differ per user → not cached

2. **Session-level memoization via `systemPromptSection()`** ([systemPromptSections.ts:20](src/constants/systemPromptSections.ts#L20)):
   ```typescript
   // Computed once, cached within the session until /clear or /compact
   if (!s.cacheBreak && cache.has(s.name)) {
     return cache.get(s.name)  // no recomputation → bytes unchanged → API cache hit
   }
   ```
   This ensures dynamic section content does not change between turns → org-level cache can also hit.

3. **`DANGEROUS_uncachedSystemPromptSection`** — Sections marked with `cacheBreak: true` (e.g., MCP instructions) are recomputed every turn and **bust the cache**. This is why MCP instructions were migrated to delta attachments.

---

### Message-Level Caching

In addition to the system prompt, **the tail of the messages array** also carries cache markers ([claude.ts:3078](src/services/api/claude.ts#L3078)):

```
messages: [
  { role: "user", content: [...] },
  { role: "assistant", content: [...], cache_control: {...} },  ← mark the last entry
  { role: "user", content: "new message" }  ← only this is new
]
```

In each request, only the latest user message is "new" content. All previous history tokens are read for free via cache hits (`cache_read_input_tokens`).

---

### Beta Header Latching — Preventing Cache Thrashing

[claude.ts:1405](src/services/api/claude.ts#L1405):

```typescript
// Sticky-on latches: once a beta header is sent, it is sent for the entire session
// Prevents mid-session toggles from changing the server-side cache key
let afkHeaderLatched = getAfkModeHeaderLatched()
let fastModeHeaderLatched = getFastModeHeaderLatched()
let cacheEditingHeaderLatched = getCacheEditingHeaderLatched()
```

The combination of beta headers is part of the cache key. If the header set changes → cache invalidation. Latching ensures that once activated, a header is never rolled back.

---

### Summary: Cache Hit Guarantee Mechanisms

```
Mechanism                              Purpose
─────────────────────────────────────────────────────────
① BOUNDARY splits static/dynamic      Static part globally shared, unaffected by dynamic content
② systemPromptSection memo            Dynamic sections unchanged within session → stable bytes
③ DANGEROUS_ marks volatile sections  Explicitly identifies what will bust the cache
④ MCP → delta attachments             MCP connect/disconnect does not bust prompt cache
⑤ Beta header latching                Header set remains constant within a session
⑥ Message-level cache_control         History messages hit cache; only new additions are processed
⑦ Deferred tools → delta              Tool pool changes do not bust system prompt cache
```

**Practical effect**: In a typical multi-turn conversation, roughly 50–70K tokens of system prompt + history messages are served via `cache_read_input_tokens`, meaning only new content needs to be paid for (approximately 1/10 of the cost).


---

## Architecture Design Analysis

### Benefits

| Design Decision | Benefit |
|----------------|---------|
| **Static/Dynamic split** | Static portion has byte-stable content → extremely high global KV cache hit rate. All users share the same prefix → massive cache efficiency |
| **`systemPromptSection` memoization** | Dynamic portions are computed once per session and cached, avoiding recomputation on every turn (memory loading, skill discovery, etc. involve I/O) |
| **`DANGEROUS_` naming** | Explicitly identifies cache-busting sections, creating "cost visibility" during code review — adding a new uncached section requires providing a reason |
| **Returns `string[]` instead of a concatenated string** | Downstream code can set different `cache_control` scopes per block, enabling fine-grained caching strategies |
| **Feature-gated conditional sections** | Compile-time DCE removes irrelevant code; at runtime different users see different prompts without needing to maintain multiple versions |
| **Simple mode early exit** | `--bare` / CI environments have zero overhead — no memory/skills/MCP loading |
| **Proactive as a separate path** | Autonomous agents do not need interactive instructions (tone/style, doing tasks), reducing wasted tokens |

### Trade-offs

| Issue | Impact |
|-------|--------|
| **Monolithic function + implicit coupling** | `getSystemPrompt` is aware of the existence and order of all sections. Adding a new section requires modifying this function; there is no registration mechanism |
| **Coarse-grained memoization** | `/clear` and `/compact` flush *all* section caches. If only memory changed, other sections also need to be recomputed (though most are fast) |
| **Only one `DANGEROUS_uncached` section** | Currently only `mcp_instructions` is uncached. If multiple sections need to be uncached in the future, the per-turn cache bust impact will compound, and the current architecture has no concept of "partial caching" |
| **Boundary position is hardcoded** | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` has only one split point. If three-tier caching (global / org / session) is needed, the current structure is insufficient |
| **Difficult to test** | The function depends on a large amount of global state (`getCwd`, `getInitialSettings`, `getSystemPromptSectionCache`, etc.), requiring extensive mocking in unit tests |
| **Prompt size is unpredictable** | There is no token budget check. MCP instructions, memory, and skill commands are all open-ended in length; in extreme cases the system prompt may exceed expected limits |

### Improvement Suggestions

1. **Registry-based section architecture**: Convert sections to a declarative registration model (similar to tool registration), where each section carries its own `order`, `cachePolicy`, and `scope`. `getSystemPrompt` would only perform resolve + assemble, without hardcoding the list.

2. **Layered cache invalidation**: Invalidate sections by name instead of global clear. For example, the `/memory` command would only invalidate the `'memory'` section.

3. **Token budget guard**: After assembly, check the total token count and, if over the limit, truncate lower-priority sections by priority (e.g., `numeric_length_anchors`, `summarize_tool_results`).

4. **Multiple-level boundaries**: Support N boundary markers mapped to different cache scopes, leaving room for future org-level / team-level shared prompts.

5. **Generalize MCP instructions delta pattern**: Currently only MCP uses delta attachments (to avoid cache busting). Other frequently changing content (e.g., memory updates) could also use the attachment delta pattern to keep the system prompt stable.
