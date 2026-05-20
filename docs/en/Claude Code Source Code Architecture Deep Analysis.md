# Claude Code Source Code Architecture — Deep Analysis

> Comprehensive module-by-module architectural trace based on decompiled Claude Code v2.1.88

---

## Table of Contents

1. [Overall Architecture & End-to-End Message Flow](#1-overall-architecture--end-to-end-message-flow)
2. [Query Loop — Core Loop Engine](#2-query-loop--core-loop-engine)
3. [API Layer & SSE Streaming](#3-api-layer--sse-streaming)
4. [StreamingToolExecutor — Concurrent Tool Execution](#4-streamingtoolexecutor--concurrent-tool-execution)
5. [Tool Permission System](#5-tool-permission-system)
6. [Context System — Context Construction](#6-context-system--context-construction)
7. [Memory System — Cross-Session Persistence](#7-memory-system--cross-session-persistence)
8. [Session Memory — Within-Session Context Recovery](#8-session-memory--within-session-context-recovery)

---

## 1. Overall Architecture & End-to-End Message Flow

### 1.1 Module Architecture Overview

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

### 1.2 Complete Lifecycle of a Single Message

```
User Input
  │
  ▼
┌──────────────────────────────────┐
│ 1. REPL.tsx: onQueryImpl()       │
│    - Fetch userContext (CLAUDE.md,│
│      memory files, currentDate)  │
│    - Fetch systemContext (git     │
│      status, cache breaker)      │
│    - prependUserContext() injects │
│      as first <system-reminder>  │
│      user message                │
└────────────┬─────────────────────┘
             ▼
┌──────────────────────────────────┐
│ 2. query.ts: query() enters loop │
│    - Initialize State (10 fields)│
│    - Check blocking_limit        │
│    - Execute queryCheckpoint()   │
└────────────┬─────────────────────┘
             ▼
┌──────────────────────────────────┐
│ 3. deps.callModel() (L659)       │
│    - Build API request           │
│    - Send HTTP POST to Claude    │
│    - Receive SSE streaming       │
└────────────┬─────────────────────┘
             ▼
┌──────────────────────────────────┐
│ 4. SSE Event Processing          │
│    (L742-835)                    │
│    - Assemble content blocks     │
│    - Each content_block_stop     │
│      immediately yields          │
│      AssistantMessage            │
│    - Detect tool_use blocks      │
│      → needsFollowUp = true     │
└────────────┬─────────────────────┘
             ▼
        ┌────┴─────┐
        │tool_use? │
        └────┬─────┘
       Yes   │   No
    ┌────────┘    └─────────┐
    ▼                       ▼
┌───────────────┐   ┌───────────────┐
│5a. Permission │   │5b. Exit check │
│   5-layer     │   │  - completed  │
│   funnel      │   │  - stop_hook  │
│   → execute   │   │  - max_turns  │
│   → collect   │   └───────────────┘
└───────┬───────┘
        ▼
┌───────────────┐
│6. Attachment   │
│   injection    │
│   (L1565)      │
│   - File data  │
│   - IDE select │
│   - Memory     │
│     prefetch   │
└───────┬───────┘
        ▼
┌───────────────┐
│7. Build next   │
│   State (L1715)│
│   → back to 2  │
└───────────────┘
```

### 1.3 Call Chain Overview

```
cli.tsx
  → main.tsx (bootstrap: config, auth, MDM)
    → REPL.tsx (React/Ink terminal UI)
      → onQueryImpl()
        → getUserContext()  ← claudemd.ts: getMemoryFiles() → getClaudeMds()
        → getSystemContext() ← context.ts: getGitStatus()
        → prependUserContext() ← api.ts: inject <system-reminder>
        → query()  ← query.ts: AsyncGenerator
          → deps.callModel() → claude.ts: queryModel() → SSE stream
          → StreamingToolExecutor → Tool.execute() → permission check
          → next turn / exit
```

---

## 2. Query Loop — Core Loop Engine

**File**: `src/query.ts`

### 2.1 Design Philosophy

The Query Loop is Claude Code's heart — an `async function* query()` AsyncGenerator that yields `QueryYield` objects for UI rendering, implementing multi-turn conversations through an infinite loop: user message → model call → tool execution → next turn.

### 2.2 State Type — Loop State

```typescript
// L555-558
type State = {
  messages: Message[]                        // Full conversation history
  toolUseContext: ToolUseContext              // Tool execution context
  autoCompactTracking: AutoCompactTracking   // Compaction tracking
  turnCount: number                          // Current turn number
  maxOutputTokensRecoveryCount: number       // max_tokens recovery count
  hasAttemptedReactiveCompact: boolean       // Whether reactive compact tried
  pendingToolUseSummary: string | undefined  // Pending tool summary
  maxOutputTokensOverride: number | undefined // max_tokens override
  stopHookActive: boolean                    // Stop hook active flag
  transition: { reason: string }             // State transition reason
}
```

### 2.3 Detailed Loop Flow

```
                    ┌──────────────────────────┐
                    │    query() entry          │
                    │    Initialize State       │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │  while (true) main loop   │
                    └────────────┬─────────────┘
                                 │
              ┌──────────────────▼──────────────────┐
              │ ① Check blocking_limit (L635-648)   │
              │    tokenCount >= blockingLimit?      │
              │    → return { reason: 'blocking_    │
              │       limit' }                       │
              └──────────────────┬──────────────────┘
                                 │ Not exceeded
              ┌──────────────────▼──────────────────┐
              │ ② callModel() (L659-684)            │
              │    for await (msg of callModel())   │
              │    SSE streaming AssistantMessages   │
              └──────────────────┬──────────────────┘
                                 │
              ┌──────────────────▼──────────────────┐
              │ ③ Message processing (L742-835)     │
              │    - Backfill tool inputs            │
              │    - Withhold recoverable errors     │
              │    - Push to assistantMessages[]     │
              │    - Detect tool_use blocks          │
              │      → needsFollowUp = true          │
              │      → toolUseBlocks.push(block)     │
              └──────────────────┬──────────────────┘
                                 │
                      ┌──────────▼──────────┐
                      │  needsFollowUp?     │
                      └──────┬─────┬────────┘
                        No   │     │  Yes
              ┌──────────────▼┐   ┌▼──────────────────┐
              │ No-tool path  │   │ Tool execution     │
              │ (L1062-1357)  │   │ path (L1360-1620) │
              └───────────────┘   └────────────────────┘
```

### 2.4 Ten Exit Conditions

| # | Condition | Location | Description |
|---|-----------|----------|-------------|
| 1 | `blocking_limit` | L635-648 | Token count exceeds hard limit, manual compaction needed |
| 2 | `image_error` | L960-975 | Image processing error |
| 3 | `model_error` | L977-997 | API call error (non-recoverable) |
| 4 | `aborted_streaming` | L1015-1051 | Streaming aborted mid-transfer |
| 5 | `completed` (no tools) | L1085 | end_turn, no tool_use, normal completion |
| 6 | `completed` (post-recovery) | L1166 | Completed after reactive compact |
| 7 | `stop_hook_prevented` | L1267-1306 | Stop hook prevents continuation |
| 8 | `aborted_tools` | L1490-1495 | Tool execution aborted |
| 9 | `hook_stopped` | L1500-1520 | Hook stopped execution |
| 10 | `max_turns` | L1704-1710 | Maximum turn limit reached |

### 2.5 Six Continue Conditions

| # | Condition | Description |
|---|-----------|-------------|
| 1 | `collapse_drain_retry` | Context-collapse staged drain, then retry |
| 2 | `reactive_compact_retry` | Prompt-too-long triggers full compaction, then retry |
| 3 | `max_output_tokens_recovery` | max_tokens hit → escalate 8k→64k |
| 4 | `stop_hook_blocking` | Stop hook blocking resolved, continue |
| 5 | `token_budget_continuation` | Token budget allows continuation |
| 6 | `next_turn` | Normal next conversation turn |

### 2.6 Recovery Mechanisms

```
API returns error
    │
    ├─ prompt_too_long (413)?
    │   ├─ Pending context-collapse drains?
    │   │   └─ → collapse_drain_retry
    │   ├─ Haven't tried reactive compact?
    │   │   └─ → reactive_compact_retry (full summary compaction)
    │   └─ Already tried → return model_error
    │
    ├─ max_tokens (stop_reason)?
    │   ├─ First time? → escalate 8k → 64k (ESCALATED_MAX_TOKENS)
    │   ├─ Under MAX_OUTPUT_TOKENS_RECOVERY_LIMIT?
    │   │   └─ → inject meta message, max_output_tokens_recovery
    │   └─ Over limit → normal completion
    │
    └─ Other error → return model_error
```

### 2.7 Attachment Injection (L1565-1620)

After tool execution completes and before constructing next-turn State, the query loop injects additional context via `getAttachmentMessages()`:

```typescript
// L1565-1590
const attachmentMessages = getAttachmentMessages(
  null,                          // input: null (not direct user input)
  queuedCommandsSnapshot,        // queued commands
  [...messagesForQuery, ...assistantMessages, ...toolResults]
)
for await (const msg of attachmentMessages) {
  toolResults.push(msg)          // Inject into toolResults
}

// L1599-1614: Memory prefetch consumption
const memoryAttachments = filterDuplicateMemoryAttachments(
  memoryPrefetchPromise,
  readFileState
)
// Injected as attachment messages
```

**Attachment Types**:
- `FileAttachment` — file content
- `selected_lines_in_ide` — code selected in IDE
- `edited_text_file` / `edited_image_file` — edited files
- `directory` — directory listing
- `PDFReference` — PDF reference
- `CompactFileReference` — compact file reference

---

## 3. API Layer & SSE Streaming

**File**: `src/services/api/claude.ts`

### 3.1 SSE (Server-Sent Events) Protocol

SSE is a unidirectional HTTP-based streaming protocol. The Claude API transmits incremental model responses via SSE, with the client receiving and assembling events one by one.

### 3.2 Event Processing Pipeline

```
HTTP POST /messages (stream: true)
    │
    ▼
┌────────────────────────────────────────┐
│         SSE Event Stream               │
│                                        │
│  message_start ──────────────────┐     │
│    └─ Save partialMessage meta   │     │
│       (id, model, usage)        │     │
│                                  ▼     │
│  content_block_start ──────────────┐   │
│    └─ Initialize contentBlocks     │   │
│       [index]:                    │   │
│       text → ''                   │   │
│       tool_use → { input: '' }    │   │
│       thinking → { thinking: '' } │   │
│                                   ▼   │
│  content_block_delta (repeated) ────┐  │
│    └─ Accumulate incremental data   │  │
│       text_delta → text +=          │  │
│       input_json_delta → input +=   │  │
│       thinking_delta → thinking +=  │  │
│                                     ▼  │
│  content_block_stop ──────────────────┐│
│    └─ [KEY] Assemble AssistantMessage ││
│       from partialMessage + single    ││
│       contentBlock                    ││
│       → yield immediately to query    ││
│                                       ▼│
│  message_delta ───────────────────────┐│
│    └─ Write back stop_reason +        ││
│       final usage to last yielded msg ││
│                                       ▼│
│  message_stop ────────────────────────│
│    └─ Stream ends                     │
└────────────────────────────────────────┘
```

### 3.3 Key Design Decision: Per-Block Yield

```typescript
// claude.ts L2171-2211
case 'content_block_stop': {
  const block = contentBlocks[part.index]
  // Assemble independent AssistantMessage
  const message: AssistantMessage = {
    ...partialMessage,           // Metadata (id, model, etc.)
    content: [block],            // Only THIS single block
    role: 'assistant',
  }
  newMessages.push(message)
  yield message                  // Yield immediately, don't wait for other blocks
}
```

**Why not wait for all blocks?**
- **Streaming UI**: Each block can be displayed as soon as it completes
- **Parallel tool execution**: tool_use blocks can start executing immediately
- **Reduced latency**: Users see responses faster

### 3.4 Usage Tracking & Cost Calculation

```typescript
// L1983: Initial usage
case 'message_start':
  updateUsage(part.message.usage)

// L2214-2256: Final usage
case 'message_delta':
  updateUsage(part.usage)
  const costUSD = calculateUSDCost(resolvedModel, usage)
  addToTotalSessionCost(costUSD, usage, options.model)
```

---

## 4. StreamingToolExecutor — Concurrent Tool Execution

**File**: `src/services/tools/StreamingToolExecutor.ts`

### 4.1 Design Philosophy

StreamingToolExecutor implements a carefully designed concurrency model: **read-only tools execute in parallel, write tools execute serially**, ensuring filesystem consistency while maximizing throughput.

### 4.2 TrackedTool Lifecycle

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
  isConcurrencySafe: boolean      // true = read-only (parallelizable)
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

### 4.3 Concurrency Control Logic

```typescript
// L129-135: canExecuteTool()
canExecuteTool(isConcurrencySafe: boolean): boolean {
  // No tools executing → can execute
  // OR: all executing tools are read-only AND candidate is read-only
  return this.executingCount === 0 ||
    (this.allExecutingAreConcurrencySafe && isConcurrencySafe)
}
```

```
Example execution order:

Tool queue: [Read₁, Read₂, Write₁, Read₃, Write₂]

Timeline:
t0: Read₁ ──▶ parallel
    Read₂ ──▶ parallel
    (Write₁ waits — not concurrencySafe)

t1: Read₁ ✓
    Read₂ ✓
    Write₁ ──▶ serial (exclusive)

t2: Write₁ ✓
    Read₃ ──▶ execute
    (Write₂ waits)

t3: Read₃ ✓
    Write₂ ──▶ serial

t4: Write₂ ✓ — all done
```

### 4.4 Promise.race Event-Driven Consumption

```typescript
// L472-482: getRemainingResults()
async *getRemainingResults() {
  while (hasIncompleteTools()) {
    // Collect all currently executing Promises
    const executingPromises = this.tools
      .filter(t => t.status === 'executing' && t.promise)
      .map(t => t.promise!)

    // Progress notification Promise
    const progressPromise = new Promise<void>(resolve => {
      this.progressAvailableResolve = resolve
    })

    // Race: any completion wakes the consumer
    await Promise.race([...executingPromises, progressPromise])

    // Yield all completed results
    yield* this.getCompletedResults()
  }
}
```

**Design elegance**: No polling, no callbacks — pure event-driven result consumption via `Promise.race`. Any tool completion or progress update immediately wakes the consumer.

### 4.5 Three-Layer Abort Hierarchy

```
┌─────────────────────────────────────┐
│ Layer 1: Query-level AbortController│  ← Entire query loop control
│   (toolUseContext.abortController)  │
│                                     │
│  ┌──────────────────────────────┐   │
│  │ Layer 2: Sibling Abort       │   │  ← Sibling tool level
│  │   (siblingAbortController)   │   │     Bash error cancels all siblings
│  │                              │   │
│  │  ┌───────────────────────┐   │   │
│  │  │ Layer 3: Tool Abort   │   │   │  ← Individual tool level
│  │  │  (toolAbortController)│   │   │     Permission denial bubbles to query
│  │  └───────────────────────┘   │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘

Abort propagation paths:
  Tool abort → permission denied? → bubble to query controller
  Bash error → sibling abort → cancel all sibling tools
  User abort → query abort → cascade to sibling → cascade to all tools
```

---

## 5. Tool Permission System

### 5.1 Five-Layer Permission Funnel

```
Tool execution requested
    │
    ▼
┌──────────────────────────────────┐
│ Layer 1: Orchestration           │
│   Query loop decides whether to  │
│   invoke tools (needsFollowUp)   │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 2: useCanUseTool()         │
│   React hook layer, UI           │
│   interaction, user confirm/deny │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 3: hasPermissionsToUseTool()│
│   Rule matching engine           │
│   Traverses all rule sources     │
│   policy → flag → user →         │
│   project → local → session →    │
│   command                        │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 4: Tool.checkPermissions() │
│   Tool's own permission checks   │
│   e.g., Bash checks command      │
│   safety, FileWrite checks paths │
└──────────────┬───────────────────┘
               ▼
┌──────────────────────────────────┐
│ Layer 5: Classifier              │
│   AI classifier (optional)       │
│   Extra judgment for high-risk   │
│   operations                     │
└──────────────┬───────────────────┘
               ▼
        allow / deny / ask
```

### 5.2 Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Standard mode, follow rules, ask user if no match |
| `plan` | Only allow read-only tools |
| `acceptEdits` | Auto-accept file edits, other ops still need confirmation |
| `bypassPermissions` | Skip all permission checks (dangerous) |
| `dontAsk` | Deny if no match (non-interactive mode) |
| `auto` | Automatic mode, smart decisions |

### 5.3 Rule Source Priority

```
Priority (high → low):
  policy     — Organization-level policies (MDM)
    ▼
  flag       — CLI arguments (--dangerously-skip-permissions)
    ▼
  user       — User-level settings (~/.claude/settings.json)
    ▼
  project    — Project-level settings (.claude/settings.json)
    ▼
  local      — Local settings (.claude/settings.local.json)
    ▼
  session    — Session-level rules (runtime dynamic)
    ▼
  command    — Command-level rules (single-command arguments)
```

### 5.4 ToolPermissionContext Structure

```typescript
// Tool.ts L123-138
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource    // Whitelist
  alwaysDenyRules: ToolPermissionRulesBySource     // Blacklist
  alwaysAskRules: ToolPermissionRulesBySource      // Always-ask list
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
}>
```

---

## 6. Context System — Context Construction

**Files**: `src/context.ts`, `src/utils/api.ts`

### 6.1 Context Construction Pipeline

```
onQueryImpl() invoked
    │
    ├──▶ getUserContext()          ← memoized
    │     │
    │     ├─ getMemoryFiles()     ← 7-source hierarchical loading
    │     │   ├─ Managed (policy)
    │     │   ├─ User (~/.claude/)
    │     │   ├─ Project (walk CWD upward through .claude/)
    │     │   ├─ Local (.claude/settings.local.json)
    │     │   ├─ --add-dir (additional directories)
    │     │   ├─ AutoMem (MEMORY.md)
    │     │   └─ TeamMem (team memory)
    │     │
    │     ├─ filterInjectedMemoryFiles()
    │     ├─ getClaudeMds()        ← concatenate all files into single string
    │     └─ return { claudeMd, currentDate }
    │
    ├──▶ getSystemContext()        ← memoized
    │     │
    │     └─ getGitStatus()
    │         ├─ git branch --show-current
    │         ├─ git rev-parse --abbrev-ref origin/HEAD
    │         ├─ git status --short (truncated at 2000 chars)
    │         ├─ git log -n5 --oneline
    │         └─ git config user.name
    │         (5 commands via Promise.all in parallel)
    │
    └──▶ prependUserContext(messages, context)
          │
          └─ Prepend a user message at array start:
             <system-reminder>
             # claudeMd
             {CLAUDE.md content + memory files}
             # currentDate
             Today's date is 2026/05/20.
             # gitStatus
             {git status info}
             </system-reminder>
```

### 6.2 getUserContext Implementation

```typescript
// context.ts L155-189
const getUserContext = memoize(async (): Promise<{[k: string]: string}> => {
  // Disable conditions
  if (env.CLAUDE_CODE_DISABLE_CLAUDE_MDS ||
      (isBareMode() && !hasAddDirs())) {
    return { currentDate: getFormattedDate() }
  }

  // Load all memory files
  const memoryFiles = await getMemoryFiles()

  // Filter already-injected memories
  const filtered = filterInjectedMemoryFiles(memoryFiles)

  // Concatenate into string
  const claudeMd = getClaudeMds(filtered)

  // Cache and return
  setCachedClaudeMdContent(claudeMd)
  return {
    ...(claudeMd && { claudeMd }),     // Conditional spread: include only if non-empty
    currentDate: getFormattedDate(),
  }
})
```

### 6.3 prependUserContext Injection

```typescript
// api.ts L449-474
function prependUserContext(
  messages: Message[],
  context: { [k: string]: string },
): Message[] {
  // Skip in test environment
  if (process.env.NODE_ENV === 'test') return messages

  // Build content string
  const contextEntries = Object.entries(context)
    .map(([key, value]) => `# ${key}\n${value}`)
    .join('\n\n')

  // Wrap as <system-reminder> user message
  const contextMessage: UserMessage = {
    role: 'user',
    content: `<system-reminder>\n${contextEntries}\n</system-reminder>`,
  }

  return [contextMessage, ...messages]
}
```

---

## 7. Memory System — Cross-Session Persistence

**File**: `src/utils/claudemd.ts`

### 7.1 Dual-Track Memory Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Memory System                      │
│                                                      │
│  ┌──────────────────┐    ┌──────────────────────┐   │
│  │  Auto Memory     │    │  Session Memory      │   │
│  │  (cross-session  │    │  (within-session     │   │
│  │   persistence)   │    │   context recovery)  │   │
│  │                  │    │                      │   │
│  │  ~/.claude/      │    │  ~/.claude/projects/ │   │
│  │  projects/{proj}/│    │  {cwd}/{sessionId}/  │   │
│  │  memory/         │    │  session-memory/     │   │
│  │                  │    │  summary.md          │   │
│  │  • MEMORY.md     │    │                      │   │
│  │    (index file)  │    │  • 9 sections        │   │
│  │  • *.md          │    │  • Forked agent gen  │   │
│  │    (content)     │    │  • Injected on       │   │
│  │                  │    │    compaction         │   │
│  └──────────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 7.2 getMemoryFiles() — 7-Source Hierarchical Loading

```typescript
// claudemd.ts L790-1075
const getMemoryFiles = memoize(async (): Promise<MemoryFileInfo[]> => {
  const files: MemoryFileInfo[] = []

  // 1. Managed (organization policy — always loaded)
  files.push(...await getManagedMemoryFiles())

  // 2. User (user-level — ~/.claude/CLAUDE.md, etc.)
  if (isUserSettingsEnabled()) {
    files.push(...await getUserMemoryFiles())
  }

  // 3. Project (project-level — walk upward from CWD)
  //    In worktree scenarios, skip Project files from main repo dirs
  files.push(...await getProjectMemoryFiles())

  // 4. Local (local-level — .claude/settings.local.json)
  files.push(...await getLocalMemoryFiles())

  // 5. Additional directories (--add-dir argument)
  files.push(...await getAddDirMemoryFiles())

  // 6. AutoMem (MEMORY.md — auto memory index)
  if (isAutoMemoryEnabled()) {
    files.push(...await getAutoMemoryFiles())  // Limited: 200 lines / 25KB
  }

  // 7. TeamMem (team memory — feature gated)
  if (isTeamMemEnabled()) {
    files.push(...await getTeamMemoryFiles())
  }

  return files
})
```

### 7.3 MEMORY.md Lifecycle

```
┌──────────────────────────────────────┐
│ Generation Phase                     │
│                                      │
│ Generated by Claude itself via the   │
│ Write tool during conversation       │
│ Path: ~/.claude/projects/{proj}/     │
│       memory/MEMORY.md               │
│                                      │
│ Format:                              │
│ - [Title](file.md) — description    │
│ - [Title](file.md) — description    │
│ (Limited to 200 lines / 25KB)        │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ Loading Phase                        │
│                                      │
│ getUserContext()                      │
│   → getMemoryFiles()                 │
│     → Source 6: AutoMem loading      │
│       → Read MEMORY.md content      │
│   → getClaudeMds()                   │
│     → Concatenate into claudeMd      │
│   → prependUserContext()             │
│     → Inject as <system-reminder>    │
│                                      │
│ Relevant memory prefetch:            │
│   Sonnet selects top 5 relevant      │
│   files from MEMORY.md → async load  │
│   → inject as attachments into       │
│   query loop                         │
└──────────────────────────────────────┘
```

---

## 8. Session Memory — Within-Session Context Recovery

**Files**: `src/services/SessionMemory/sessionMemory.ts`, `prompts.ts`, `sessionMemoryUtils.ts`, `sessionMemoryCompact.ts`

### 8.1 Trigger Decision Tree

```typescript
// sessionMemory.ts L134-181
function shouldExtractMemory(messages: Message[]): boolean
```

```
shouldExtractMemory(messages)
    │
    ├─ Not initialized?
    │   └─ currentTokens < 10,000? → return false
    │      Otherwise → markInitialized()
    │
    ├─ tokenGrowth < 5,000 (since last extraction)?
    │   └─ return false (token threshold not met)
    │
    ├─ toolCallsSinceLastUpdate >= 3?
    │   └─ Yes → shouldExtract = true (both thresholds met)
    │
    ├─ No tool calls in last turn?
    │   └─ Yes → shouldExtract = true (natural conversation break)
    │
    └─ shouldExtract?
        ├─ Yes → record lastMemoryMessageUuid, return true
        └─ No  → return false
```

**Configuration Thresholds**:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `minimumMessageTokensToInit` | 10,000 | Minimum tokens for first extraction |
| `minimumTokensBetweenUpdate` | 5,000 | Minimum token growth between extractions |
| `toolCallsBetweenUpdates` | 3 | Minimum tool calls between extractions |

### 8.2 Extraction Pipeline

```
Post-sampling hook fires
    │
    ▼
┌─────────────────────────────────┐
│ shouldExtractMemory(messages)?  │──No──▶ skip
└──────────────┬──────────────────┘
               │ Yes
               ▼
┌─────────────────────────────────┐
│ sequential() wrapper (serial)   │
│                                 │
│ 1. setupSessionMemoryFile()     │
│    Create/read notes file       │
│    Path: ~/.claude/projects/    │
│    {cwd}/{sessionId}/           │
│    session-memory/summary.md    │
│                                 │
│ 2. buildSessionMemoryUpdatePrompt()
│    Build update instructions    │
│                                 │
│ 3. runForkedAgent()             │
│    Launch Sonnet sub-agent      │
│    Tool restriction: FileEdit   │
│    only on summary.md           │
│                                 │
│ 4. updateLastSummarizedMessageId│
│    Record extraction progress   │
└─────────────────────────────────┘
```

### 8.3 Notes Template — 9 Fixed Sections

```markdown
# Session Title
_Title of the session_

# Current State
_Current work state — most critical section for compaction recovery_

# Task specification
_User's original task description_

# Files and Functions
_Files, functions, modules involved_

# Workflow
_Workflow and steps taken_

# Errors & Corrections
_Errors encountered and corrections made_

# Codebase and System Documentation
_Codebase and system documentation discovered_

# Learnings
_Lessons learned_

# Key results
_Key results and outputs_

# Worklog
_Chronological work log_
```

Each section is limited to 2,000 tokens. Total file limit: 12,000 tokens.

### 8.4 Permission Restriction — createMemoryFileCanUseTool

```typescript
// sessionMemory.ts L460-481
function createMemoryFileCanUseTool(memoryPath: string): CanUseToolFn {
  return async (tool, input) => {
    if (tool.name === FILE_EDIT_TOOL_NAME &&
        input.file_path === memoryPath) {
      return { behavior: 'allow' }    // Only allow editing the notes file
    }
    return {
      behavior: 'deny',
      message: 'Only file edits on the memory file are allowed'
    }
  }
}
```

### 8.5 Compaction Integration

```
Context approaching limit → trigger autocompact
    │
    ▼
┌─────────────────────────────────┐
│ trySessionMemoryCompaction()     │
│ (sessionMemoryCompact.ts L514)  │
│                                 │
│ 1. waitForSessionMemoryExtraction()
│    Wait up to 15 seconds        │
│                                 │
│ 2. Read session memory notes    │
│                                 │
│ 3. Calculate message boundary   │
│    - Find lastSummarizedId pos  │
│    - Keep at least 10K tokens   │
│    - Keep at least 5 text blocks│
│    - Keep at most 40K tokens    │
│                                 │
│ 4. truncateSessionMemoryForCompact()
│    Truncate sections >8000 chars│
│                                 │
│ 5. Inject truncated notes into  │
│    post-compact messages        │
│                                 │
│ 6. Validate post-compact tokens │
│    Exceeds threshold?           │
│    → fallback to legacy compact │
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

### 8.6 /summary Command — Manual Trigger

```typescript
// sessionMemory.ts L387-453
async function manuallyExtractSessionMemory(
  messages: Message[],
  toolUseContext: ToolUseContext,
): Promise<ManualExtractionResult> {
  // Bypasses all threshold checks
  // Directly: setupSessionMemoryFile → runForkedAgent
  // Returns { success, memoryPath?, error? }
}
```

### 8.7 Custom Templates

Users can place custom files under `~/.claude/session-memory/config/`:
- `template.md` — custom section template
- `prompt.md` — custom extraction instructions

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

## Appendix: Key TypeScript Patterns

### A.1 Conditional Property Spread

```typescript
// Include property only when claudeMd is non-empty
return {
  ...(claudeMd && { claudeMd }),
  currentDate: getFormattedDate(),
}
```

### A.2 AsyncGenerator Loop Engine

```typescript
async function* query(initialState: State): AsyncGenerator<QueryYield> {
  let state = initialState
  while (true) {
    // ... processing logic
    yield { type: 'update', ... }  // UI rendering
    // return to exit, or state = nextState to continue
  }
}
```

### A.3 Memoize Cache Pattern

```typescript
const getUserContext = memoize(async () => { ... })
// Caches result until explicitly cleared
// Clear via getUserContext.cache.clear?.()
```

### A.4 sequential() Serial Guard

```typescript
const extractSessionMemory = sequential(async (context) => {
  // Ensures only one extraction task runs at a time
})
```

---

*Generated: 2026-05-20*
*Based on decompiled Claude Code v2.1.88*
