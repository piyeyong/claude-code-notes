
---

## Query Loop `query()` Implementation Deep Dive

### Overall Structure

[query.ts:219](src/query.ts#L219) is an **AsyncGenerator** that yields events to be consumed by the upper layer (REPL / SDK). The core is a `while(true)` loop inside `queryLoop()` (line 241).

```
query()
  └─ yield* queryLoop()
       └─ while(true) {
            1. Compact (snip → microcompact → collapse → autocompact)
            2. Call API (callModel streaming)
            3. Collect tool_use blocks
            4. Execute tools
            5. Inject attachments
            6. Decide: continue (has tool_use) or return (done)
          }
```

---

### Six Phases Per Iteration

#### Phase 1: Compact (lines 396–543)

```
messages → snip → microcompact → context_collapse → autocompact → messagesForQuery
```

Compression proceeds level by level, with each level independently deciding whether to trigger. The order is fixed: snip removes ancient messages, microcompact compresses tool results, collapse folds intermediate segments, autocompact performs a full summarization.

#### Phase 2: API Call (lines 654–863)

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt, tools, signal, options: { model, ... }
})) {
  // Collect assistant messages and tool_use blocks from the stream
  if (streamingToolExecutor) {
    streamingToolExecutor.addTool(toolBlock)  // Start tools as they stream in
  }
}
```

Key point: **StreamingToolExecutor** begins executing completed tool_use blocks in parallel while the API is still streaming output.

#### Phase 3: Error Recovery (lines 1062–1256)

Entered when `needsFollowUp === false` (the model did not request a tool):
- **prompt-too-long** → context collapse drain → reactive compact → report error
- **max_output_tokens** → escalate to 64k → inject "resume" message to continue → report error
- **stop hooks** → execute user hooks; hooks can prevent termination or inject errors to make the model retry

#### Phase 4: Tool Execution (lines 1360–1408)

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()  // Fetch remaining results
  : runTools(toolUseBlocks, ...)                  // Traditional serial/parallel execution

for await (const update of toolUpdates) {
  yield update.message    // Send tool_result to REPL for display
  toolResults.push(...)   // Collect for the next API call
}
```

#### Phase 5: Attachments Injection (lines 1580–1628)

After tool execution, inject additional context:
- **Queued commands** (commands entered by the user while AI was thinking)
- **Memory prefetch** (relevant memory files)
- **Skill discovery prefetch** (recommended skills)
- **File change notifications** (file changes made by other tools)

#### Phase 6: Loop Decision (lines 1704–1728)

```typescript
if (maxTurns && nextTurnCount > maxTurns) return { reason: 'max_turns' }

state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  turnCount: nextTurnCount,
  transition: { reason: 'next_turn' },
  ...
}
// continue → next iteration of while(true)
```

---

### How Tool / MCP / Skill / SubAgent Plug In

From `query()`'s perspective, all of these are **unified Tools**:

| Type | Implementation | How query sees it |
|------|------|---------------|
| **Regular Tool** (Bash, FileRead, etc.) | Implements `Tool` base class in `src/tools/*/` | `tool_use` block → executed by `runTools()` |
| **MCP Tool** | MCP client registered as a Tool; calls remote server via JSON-RPC at runtime | Same as above — transparent to query |
| **Skill** | `SkillTool` receives skill name → loads skill prompt → injects into messages | `tool_use` block with name=`Skill` |
| **SubAgent** | `AgentTool` → fork context → recursively calls `query()` | `tool_use` block with name=`Agent` → internally creates a new `query()` generator |

```
query() [main thread]
  → model returns tool_use: { name: "Agent", input: { prompt: "..." } }
    → AgentTool.execute()
      → Creates new toolUseContext + messages
      → Calls query() [sub-agent, recursive]
        → Sub-agent has its own while(true) loop
      → Returns result as tool_result
```

**StreamingToolExecutor** concurrency strategy ([toolOrchestration.ts:26](src/services/tools/toolOrchestration.ts#L26)):

```typescript
for (const { isConcurrencySafe, blocks } of partitionToolCalls(toolUseMessages)) {
  if (isConcurrencySafe) runToolsConcurrently(blocks)  // read-only tools run in parallel
  else runToolsSerially(blocks)                         // tools with side effects run serially
}
```

---

## Architecture Analysis

### Design Strengths

| Decision | Benefit |
|------|------|
| **AsyncGenerator pattern** | Callers can consume events one at a time (streaming UI updates), can `.return()` to abort, and can use `for await` for natural termination. More composable than callbacks or EventEmitter |
| **Unified Tool abstraction** | MCP, Skill, and SubAgent are all Tools. `query()` does not need to know the tool type; adding new tools requires zero changes |
| **StreamingToolExecutor** | Begins executing completed tool blocks while the API is still streaming, reducing latency (I/O and computation overlap) |
| **State object + continue** | Each `continue` point constructs a complete `State`, clearly expressing "what state to restart from." Less error-prone than assigning N `let` variables |
| **Multi-level compaction pipeline** | Each level is independent and triggered on demand. Lightweight snip/microcompact runs nearly every time (cheap); heavyweight autocompact only triggers at a threshold (one API call) |
| **Reactive compact (post-hoc)** | Does not try to predict token count upfront; lets the API report 413 → emergency compact → retry. Simple and precise |
| **Memory/Skill prefetch** | Prefetch completes in parallel during tool execution; by the time of injection it is already settled, adding zero extra latency |
| **turnCount + maxTurns** | The Agent SDK needs to limit agent execution depth; implemented via simple counting |

### Trade-offs

| Issue | Impact |
|------|------|
| **Single while(true) loop of 1500 lines** | Poor readability; 8 `continue` exit points scattered throughout (collapse_drain, reactive_compact, max_output_recovery, stop_hook, token_budget, model_fallback…); complex mental model |
| **State object vs. type safety** | All continue points manually construct State, making it easy to miss or misset fields. Adding a new state field requires changes in 7+ places |
| **Hardcoded compaction pipeline order** | snip → MC → collapse → autocompact is fixed. Changing the order or running multiple compactions in parallel requires a rewrite |
| **Generator cannot be cloned/forked** | SubAgent recursively calls `query()` to create a new generator, but the main thread's generator cannot be forked. Speculative execution of two branches is not supported by the current architecture |
| **Partial `deps` injection** | `deps.callModel` and `deps.microcompact` are injectable, but snip, collapse, autocompact, and hooks are all direct imports, requiring module mocking in tests |
| **Tool execution and API call coupled in the same loop** | StreamingToolExecutor improves performance but also increases complexity (requires discard on fallback, drain remaining on abort) |
| **Fixed attachment injection timing** | Always after tool execution, before the next API call. If some attachments (e.g., memory) need to be injected earlier (to influence tool decisions), the current design cannot do that |
| **No backpressure mechanism** | The generator's yield to REPL's `onQueryEvent` is synchronously consumed. If UI rendering is slow, there is no backpressure signal to pause query |

### Improvement Suggestions

1. **Split the loop body into pipeline stages**: Break the inside of while(true) into independent functions (`compressStage()`, `callModelStage()`, `executeToolsStage()`, `attachmentStage()`, `decisionStage()`), each returning `State | Terminal`, with the main loop only dispatching.

2. **Make the state machine explicit**: The current `transition.reason` is already a proto-state-machine. Using a discriminated union to clearly define all legal transitions would give compile-time guarantees that no continue path is left unhandled.

3. **Make the compaction pipeline configurable**: Define compaction stages as an ordered array `CompactionStage[]`, where each stage has its own `shouldRun()` and `run()`. This makes it easy to A/B test different compaction strategies or orderings.

4. **Full DI**: Inject snip, collapse, autocompact, hooks, etc. all through `deps`, eliminating direct dependencies on global imports and improving testability.

5. **Backpressure / batched yield**: In high-frequency yield scenarios (streaming token events), introduce micro-batching (flush every N ms) to reduce the number of React re-renders.

6. **SubAgent resource limits**: Currently, SubAgent recursively calls `query()` with no global depth/concurrency limit (only `maxTurns`). Adding a global concurrency counter is recommended to prevent agents from forking indefinitely.

7. **Extract "recovery" logic as middleware**: prompt-too-long recovery, max-output-tokens recovery, and model fallback all follow the pattern of "error → transform state → retry" and can be unified as `RecoveryMiddleware[]`, so the loop body only needs `for (const mw of recoveries) { if (mw.matches(error)) state = mw.recover(state); continue }`.



# Appendix

## Compact Deep Dive

Compact executes before each API call and consists of 4 **sequential phases**, each of which may run concurrently:

---

### Phase 1: SNIP (~L400–410)

**Feature gate**: `HISTORY_SNIP`

Trims the oldest conversation history and establishes a compact boundary. Records `snipTokensFreed` (number of tokens freed) and passes it to the subsequent autocompact for threshold correction.

---

### Phase 2: MICROCOMPACT (~L412–426)

**No feature gate** — always runs.

Compresses the result content of specific tool calls. If `CACHED_MICROCOMPACT` is enabled, the boundary message is **deferred until after the API streaming response ends** (~L870–892), at which point the actual `cache_deleted_input_tokens` returned by the API is used to fill in the accurate token count.

---

### Phase 3: CONTEXT_COLLAPSE (~L440–447)

**Feature gate**: `CONTEXT_COLLAPSE`

Generates a "collapsed view" of messages — does not modify the actual message array, but instead projects it via a commit log at read time. Collapse runs before autocompact; if the token count is already below the threshold after collapsing, autocompact becomes a no-op.

---

### Phase 4: AUTOCOMPACT (~L453–543)

**Always called**, receives messages processed by the first three steps.

- **Success**: Generates a summary message, resets tracking (`compacted: true, turnCounter: 0, consecutiveFailures: 0`), yields the new message, and continues the query with the compacted messages
- **Failure**: Propagates `consecutiveFailures` (circuit breaker pattern), does not replace messages, leaving it for subsequent reactive compact to handle

---

### Error Recovery Paths

| Path | Trigger Condition | Behavior |
|------|---------|------|
| `collapse_drain_retry` | 413 error + context_collapse has pending folds | Release folded content first, then retry |
| `reactive_compact_retry` | 413 error + autocompact failed | Run emergency compaction; `hasAttemptedReactiveCompact` prevents loops |
| `max_output_tokens_escalate` | max_output_tokens hit | Escalate from 8K to 64K |

### Flow Summary

```
messages → getMessagesAfterCompactBoundary()
  → [SNIP] → snipTokensFreed
    → [MICROCOMPACT] → deferred boundary
      → [CONTEXT_COLLAPSE] → projected collapsed view
        → [AUTOCOMPACT] → replace messages on success / continue on failure
          → API call
            → yield deferred microcompact boundary
            → error? → reactive_compact / collapse_drain / escalate
```

**Key design**: The 4 phases are **not mutually exclusive** and can all execute within the same turn. `snipTokensFreed` is passed across phases to ensure the autocompact threshold is accurate. `hasAttemptedReactiveCompact` prevents reactive compact from looping infinitely (reset only occurs during normal `next_turn` transitions).


## StreamingToolExecutor Deep Dive

Defined in [StreamingToolExecutor.ts:40](src/services/tools/StreamingToolExecutor.ts#L40), this implements a **concurrency controller that executes tools as tool_use blocks are received during API streaming**.

---

### Core Data Structure

Each tool is wrapped as a `TrackedTool` (L21–32):

```
id, block, assistantMessage → tool identity
status: queued → executing → completed → yielded → lifecycle
isConcurrencySafe → whether it can run in parallel
results[] → execution results (buffered, yielded in order)
pendingProgress[] → progress messages (yielded immediately)
contextModifiers[] → context modifiers
```

---

### Concurrency Control Model (L129–134)

Rules for `canExecuteTool(isConcurrencySafe)`:

| Currently executing tools | New tool type | Can execute? |
|---|---|---|
| None | Any | **Yes** |
| All concurrency-safe | concurrency-safe | **Yes** |
| Any non-concurrent | Any | **No** |
| All concurrency-safe | non-concurrent | **No** |

In short: **read-only tools can run in parallel; write tools execute exclusively**.

---

### Tool Addition Flow `addTool()` (L76–124)

1. Look up the tool definition; if not found, immediately generate an error result marked as `completed`
2. Parse input; determine concurrency safety via `isConcurrencySafe()`
3. Create `TrackedTool` with status `queued`
4. Call `processQueue()` to attempt immediate execution

---

### Queue Processing `processQueue()` (L140–151)

Iterates over all `queued` tools:
- Concurrency condition met → execute
- Not met and tool is non-concurrent → **break** (preserve order, block subsequent tools)
- Not met but tool is concurrent → skip and continue checking the rest

---

### Tool Execution `executeTool()` (L265–405)

Core flow:

1. **Status transition**: `queued` → `executing`; register in `inProgressToolUseIDs`
2. **Abort check**: Before executing, check whether it has been aborted (sibling error / user interrupt / streaming fallback); if so, generate a synthetic error message
3. **Create child AbortController** (L301–318):
   - Inherits from `siblingAbortController`
   - Abort events **bubble up to the parent controller** (unless it is a sibling_error or already discarded), ensuring scenarios like permission denial can terminate the entire turn
4. **Consume the `runToolUse()` generator** (L332–382):
   - Check abort reason on each iteration
   - Progress messages → `pendingProgress` (immediately notify waiters)
   - Result messages → `messages` buffer
   - **Only Bash tool errors cancel siblings** (L359–363): `hasErrored = true`, abort siblingAbortController. Other tools (Read, WebFetch, etc.) are independent of each other
5. **Completion**: Status → `completed`; trigger `processQueue()` to advance the queue

---

### Result Output

**Two retrieval methods**:

**`getCompletedResults()`** (L412–440) — synchronous, non-blocking:
- Iterates in the order tools were added
- Yields all `pendingProgress` first (progress does not wait)
- Yields results from `completed` tools and marks them as `yielded`
- Encounters an executing non-concurrent tool → **break** (preserve order)

**`getRemainingResults()`** (L453–490) — asynchronous, blocking:
- `while(hasUnfinishedTools())` loop
- Uses `Promise.race([...executingPromises, progressPromise])` to wait for any tool to complete or for progress
- Yields all completed results each round

---

### Abort Mechanism (3 Layers)

```
toolUseContext.abortController  ← user interrupt (ESC)
  └─ siblingAbortController     ← Bash error cascade
       └─ toolAbortController   ← per-tool level (permission denial, etc.)
```

| Scenario | Trigger | Effect |
|------|------|------|
| User presses ESC | `abortController.abort('interrupt')` | Tools with `interruptBehavior='cancel'` receive a synthetic rejection message |
| Bash tool errors | `siblingAbortController.abort('sibling_error')` | All parallel sibling tools receive "cancelled: parallel tool call errored" |
| Streaming fallback | `discard()` is called | All pending tools receive a "streaming fallback" error |
| Permission denial | `toolAbortController.abort()` | Bubbles up to `abortController`, terminating the entire turn |

---

### Usage in query.ts

```
During API streaming:
  Receive tool_use block → executor.addTool(block, msg)  // Execute as blocks arrive
  After each streaming event → executor.getCompletedResults()   // Yield as tools complete

After API streaming ends:
  executor.getRemainingResults()  // Wait for all unfinished tools
  or runTools() (non-streaming fallback)
```

**Key design**: Tools begin executing while the API is still outputting text/thinking, eliminating the serial wait of "wait for API response before executing." Progress messages are pushed to the UI in real time; results are buffered in order to guarantee deterministic output.



## Tool Permission Check Logic and Layers

The entire permission system is a **5-layer funnel**, filtering from the outside in:

```
┌─────────────────────────────────────────────┐
│ Layer 1: Tool Orchestration (entry point)    │
│   toolOrchestration.ts → runToolUse()       │
│   Calls canUseTool() for each tool_use block │
├─────────────────────────────────────────────┤
│ Layer 2: useCanUseTool Hook (coordination)   │
│   Builds PermissionContext                   │
│   Calls hasPermissionsToUseTool()            │
│   Dispatches to UI / auto / deny based on result │
├─────────────────────────────────────────────┤
│ Layer 3: hasPermissionsToUseTool (core decision) │
│   Permission mode check → rule matching → tool self-check │
├─────────────────────────────────────────────┤
│ Layer 4: Tool.checkPermissions (tool self-check) │
│   Each tool's own permission logic (paths, command classification, etc.) │
├─────────────────────────────────────────────┤
│ Layer 5: Bash Classifier / Safety (safety layer) │
│   AI classifier, dangerous pattern detection, sandbox rules │
└─────────────────────────────────────────────┘
```

---

### Layer 1: Entry Point — Tool Orchestration

[toolOrchestration.ts](src/services/tools/toolOrchestration.ts) and [toolExecution.ts](src/services/tools/toolExecution.ts)

- `runTools()` first uses `partitionToolCalls()` to split tools into **read-only concurrent batches** and **write-operation serial batches**
- Calls `canUseTool(tool, input, context, ...)` for each tool_use block
- This is the sole entry point for permission checking

---

### Layer 2: useCanUseTool Hook — Coordination and Dispatch

[useCanUseTool.tsx](src/hooks/useCanUseTool.tsx)

```
canUseTool(tool, input, context, assistantMessage, toolUseID)
  ↓
createPermissionContext()  ← Build logging/persistence/abort context
  ↓
hasPermissionsToUseTool()  ← Core decision
  ↓
What is the result?
  allow → pass through directly, log
  deny  → reject, record reason
  ask   → three cases:
    ├─ coordinator worker → handleCoordinatorPermission()
    ├─ swarm worker → handleSwarmWorkerPermission()
    └─ main thread → wait for speculative classifier (2 seconds)
                → still needs confirmation → handleInteractivePermission() → show UI
```

---

### Layer 3: hasPermissionsToUseTool — Core Decision Engine

[permissions.ts](src/utils/permissions/permissions.ts)

Decisions are checked in priority order:

| Priority | Check | Result |
|--------|--------|------|
| 1 | **Permission Mode** | `plan` → deny, `bypassPermissions` → allow, `dontAsk` → deny |
| 2 | **Deny rules** | Matches `alwaysDenyRules` → deny |
| 3 | **Allow rules** | Matches `alwaysAllowRules` → allow |
| 4 | **Ask rules** | Matches `alwaysAskRules` → ask (force prompt) |
| 5 | **Tool.checkPermissions()** | The tool's own permission judgment |
| 6 | **Auto mode** | ask result + auto mode → call AI classifier to decide automatically |

**Rule sources** (by override priority):

```
policySettings     ← Organization policy (highest priority)
flagSettings       ← CLI --allow/--deny arguments
userSettings       ← ~/.claude/settings.json
projectSettings    ← .claude/settings.json
localSettings      ← .claude/settings.local.json
session            ← Temporary in-session rules
command            ← /permissions interactive settings
```

---

### Layer 4: Tool.checkPermissions — Tool Self-Check

[Tool.ts:500–503](src/Tool.ts#L500-L503)

Each tool implements its own `checkPermissions(input, context)`, returning `allow` / `ask` / `deny`:

| Tool | Self-check logic |
|------|---------|
| **BashTool** | Parse command → split subcommands → match rules one by one → path validation → dangerous pattern detection |
| **FileEditTool / FileWriteTool** | Is the path within an allowed directory? Does the file exist? |
| **FileReadTool / GlobTool / GrepTool** | Path validation ([pathValidation.ts](src/utils/permissions/pathValidation.ts)) |
| **MCPTool** | Returns `passthrough`, delegating to the general permission framework |
| **Default** | `allow` (most built-in tools need no additional checks) |

---

### Layer 5: Safety Classifier (Bash-specific)

[bashClassifier.ts](src/utils/permissions/bashClassifier.ts)

This is an **Anthropic-internal feature** (the open-source version stub returns `disabled`):

```
Bash command
  ↓
startSpeculativeClassifierCheck()  ← Start async classification while API is streaming
  ↓
useCanUseTool waits up to 2 seconds
  ↓
High confidence allow → auto-approve, skip prompt
Low confidence / deny → still prompt the user
```

Also included:
- [dangerousPatterns.ts](src/utils/permissions/dangerousPatterns.ts) — Hardcoded dangerous command patterns (`rm -rf /`, `git push --force`, etc.)
- [sedConstraints.ts](src/utils/permissions/sedConstraints.ts) — Special constraints for sed commands
- [shellRuleMatching.ts](src/utils/permissions/shellRuleMatching.ts) — Glob rule matching for shell commands

---

### Permission Decision Result Types

[types/permissions.ts](src/types/permissions.ts#L241-L246)

```typescript
PermissionDecision =
  | { behavior: 'allow', updatedInput?, decisionReason? }
  | { behavior: 'ask',   message, suggestions?, pendingClassifierCheck? }
  | { behavior: 'deny',  message, decisionReason }
```

`decisionReason` records **why** the decision was made:

| Reason Type | Meaning |
|-------------|------|
| `rule` | Matched an allow/deny rule |
| `mode` | Determined by current permission mode (plan/bypass/dontAsk) |
| `classifier` | Determined by the AI classifier |
| `hook` | Determined by a user-configured hook |
| `safetyCheck` | Intercepted by a safety check |
| `sandboxOverride` | Overridden by sandbox |
| `workingDir` | Working directory violation |

---

### One-Line Summary

The permission system is a hierarchical funnel of **Mode → Rules → Tool self-check → Classifier**: check the global mode first, then match user rules, then let the tool check itself, and finally use the AI classifier as a safety net for high-risk tools like Bash. Each layer can short-circuit with allow/deny directly, without having to traverse all layers.
