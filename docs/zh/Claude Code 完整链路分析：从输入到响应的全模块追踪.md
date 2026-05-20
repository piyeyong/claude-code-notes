## Claude Code 完整链路分析：从输入到响应的全模块追踪

---

### 总体架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         入口层 (Entrypoints)                             │
│  cli.tsx ──→ interactive (REPL) ──→ main.tsx ──→ launchRepl()            │
│          └─→ non-interactive (-p) ──→ ask() one-shot                    │
│          └─→ MCP server mode ──→ server/                                │
│          └─→ Agent SDK mode ──→ QueryEngine directly                    │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                      用户输入处理层 (Input Processing)                    │
│  processUserInput() → slash commands / bash / text prompt                │
│  hooks: executeUserPromptSubmitHooks() → 用户自定义前置钩子              │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                     查询引擎层 (QueryEngine)                             │
│  submitMessage() → system prompt assembly + API query loop               │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                         核心循环层 (query.ts)                            │
│  while(true) { compress → API call → stream → tools → loop or exit }    │
└──────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────┐
│                          响应输出层                                       │
│  yield SDKMessage → React/Ink UI 渲染 or stdout JSON                     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### 1. 入口层：REPL vs CLI vs SDK

| 入口模式 | 文件 | 行为 |
|----------|------|------|
| **Interactive REPL** | [cli.tsx](src/entrypoints/cli.tsx) → [main.tsx](src/main.tsx) | 启动 React/Ink TUI，循环等待用户输入 |
| **Non-interactive** (`-p`) | [cli.tsx](src/entrypoints/cli.tsx) → `ask()` | 单次 QueryEngine 实例，结果输出到 stdout |
| **MCP Server** | [src/server/](src/server/) | 作为 MCP 协议服务端，接受外部调用 |
| **Agent SDK** | [src/entrypoints/agentSdkTypes.ts](src/entrypoints/agentSdkTypes.ts) | 直接实例化 QueryEngine，yield SDKMessage |

**REPL 主循环** ([main.tsx](src/main.tsx)):
```
init → loadPlugins → loadSkills → loadMCPClients → createAppState
  → while(用户输入) {
      processUserInput() → QueryEngine.submitMessage() → render response
    }
```

---

### 2. 用户输入处理 (processUserInput)

**文件**: [src/utils/processUserInput/processUserInput.ts](src/utils/processUserInput/processUserInput.ts)

```
用户文本
  ├─ /command → findCommand() → 执行本地命令 (不调 API)
  ├─ !bash   → 本地 shell 执行，输出展示
  └─ 普通文本 → 创建 UserMessage
       ├─ 附加图片 (paste/drag)
       ├─ 附加 IDE selection
       ├─ executeUserPromptSubmitHooks() → 前置 hook 验证
       └─ shouldQuery = true → 进入 QueryEngine
```

---

### 3. Commands 系统 (Slash Commands)

**文件**: [src/commands.ts](src/commands.ts)

87 个命令注册在静态数组中。关键命令：
- `/compact` → 手动触发上下文压缩
- `/clear` → 清空对话
- `/memory` → 查看/编辑记忆
- `/config` → 修改设置
- `/agents` → 管理 agent 定义

命令分两类：
- **LocalCommand**: 纯本地执行，不调 API（`/clear`, `/config`）
- **PromptCommand**: 生成消息后继续调 API（`/compact` fork 一个摘要 agent）

---

### 4. QueryEngine — 会话管理

**文件**: [src/QueryEngine.ts](src/QueryEngine.ts)

```typescript
class QueryEngine {
  mutableMessages: Message[]       // 完整对话历史
  readFileState: FileStateCache    // 文件读取缓存（turn 级）
  totalUsage: NonNullableUsage     // 累计 token 消耗

  async *submitMessage(prompt): AsyncGenerator<SDKMessage> {
    // 1. 解析模型、thinking 配置
    // 2. fetchSystemPromptParts() → 组装系统 prompt
    // 3. 创建 ToolUseContext (工具权限、abort、MCP 客户端)
    // 4. 附加 memory attachments
    // 5. 调用 query() 生成器 → 流式 yield 结果
    // 6. 持久化 session transcript
  }
}
```

---

### 5. Context 系统 (System Prompt 组装)

**文件**: [src/utils/queryContext.ts](src/utils/queryContext.ts) + [src/constants/prompts.ts](src/constants/prompts.ts)

系统 prompt 由多个 **section** 拼接，每个 section 有缓存策略：

```
System Prompt 组成:
┌─────────────────────────────────────────────┐
│ ① 核心身份 prompt (角色、行为规则)           │  ← cache: global, long TTL
│ ② 工具列表 (40+ tools 的 schema)            │  ← cache: global
│ ③ Skill 描述 (可用 skill 名+说明)           │  ← cache: session
│ ④ MCP 工具说明 (MCP server 注册的工具)       │  ← cache: session
│ ⑤ Memory prompt (MEMORY.md + 相关记忆)       │  ← cache: ephemeral
│ ⑥ 环境信息 (OS, CWD, git, model)            │  ← cache: ephemeral
│ ⑦ 自定义 prompt (CLAUDE.md, append)         │  ← cache: session
│ ⑧ 输出样式、语言设置                        │  ← cache: ephemeral
└─────────────────────────────────────────────┘

User Context (注入到首条 user message 前):
  - gitStatus, IDE selection, 通知, 附件

System Context (追加到系统 prompt 末尾):
  - 当前日期, session hints, coordinator info
```

---

### 6. Memory 系统

**文件**: [src/memdir/memdir.ts](src/memdir/memdir.ts)

```
加载流程:
  loadMemoryPrompt()
    → 读取 MEMORY.md (索引文件, max 200 行)
    → 读取引用的记忆文件 (.claude/memory/*.md)
    → truncateEntrypointContent() 截断到 25KB
    → 注入 system prompt section

相关记忆搜索:
  startRelevantMemoryPrefetch()   ← 在 query 循环开始时异步启动
    → findRelevantMemories()      ← 语义匹配当前对话内容
    → 作为 AttachmentMessage 注入
```

记忆类型：user / feedback / project / reference — 存储为 frontmatter markdown 文件。

---

### 7. MCP 系统

**文件**: [src/services/mcp/client.ts](src/services/mcp/client.ts)

```
MCP 生命周期:
  settings.json 中配置 MCP servers
    → 启动时创建 MCPServerConnection (stdio/sse/http/ws)
    → client.listTools() → 注册为 MCPTool
    → client.listResources() → 注册为 ReadMcpResourceTool

MCPTool 执行:
  query loop 中模型调用 mcp__server__toolname
    → MCPTool.execute()
      → client.callTool(name, args)
      → 返回 tool_result (text/image/resource)

MCP 在 prompt 中:
  getMcpInstructionsSection() 列出所有 MCP 工具的 name + description
  OR delta attachments (增量更新, 避免 prompt 膨胀)
```

---

### 8. Tool 系统

**文件**: [src/Tool.ts](src/Tool.ts) + [src/services/tools/toolOrchestration.ts](src/services/tools/toolOrchestration.ts)

```
Tool 接口:
  name, inputSchema (Zod), execute(), isReadOnly, userFacingName
  
工具注册 (tools.ts):
  BashTool, FileReadTool, FileEditTool, FileWriteTool,
  GlobTool, GrepTool, WebSearchTool, WebFetchTool,
  AgentTool, SkillTool, MCPTool, TodoWriteTool,
  NotebookEditTool, ScheduleCronTool ...

执行流 (toolOrchestration.ts → runTools()):
  ① 收集所有 tool_use blocks
  ② 分类: read-only → 并发执行 | write → 串行执行
  ③ 每个工具执行前: canUseTool() 权限检查
     → allow / deny / ask_user (弹出确认)
  ④ tool.execute(input, context) → 返回 ToolResult
  ⑤ 大结果走 toolResultStorage 存文件，替换为引用

StreamingToolExecutor:
  模型流式输出过程中，tool_use block 一到达就开始执行
  → 与模型生成剩余内容并行
  → 显著减少 turn 延迟
```

---

### 9. Skill 系统

**文件**: [src/skills/loadSkillsDir.ts](src/skills/loadSkillsDir.ts)

```
Skill = 可发现的命令式 prompt + 可选 shell 脚本

来源:
  .claude/skills/ (项目级)
  ~/.claude/skills/ (用户级)
  managed policy dirs (企业级)
  plugin skills

加载:
  Markdown 文件解析 frontmatter (name, description, allowed_tools)
  → 注册到 command 池
  → 列入 system prompt 的 skill 列表

调用:
  模型调用 SkillTool → 加载 skill markdown → 注入为 prompt → 执行
  OR 用户 /skill-name → 直接触发
```

---

### 10. Hooks 系统

**文件**: [src/utils/hooks.ts](src/utils/hooks.ts)

```
Hook 执行时机:
  ┌─ userPromptSubmit ──→ 用户提交前 (验证/拦截)
  ├─ preSampling ───────→ API 调用前
  ├─ postSampling ──────→ API 响应后
  ├─ preCompact ────────→ 压缩前
  ├─ postCompact ───────→ 压缩后
  └─ stopFailure ───────→ 停止/错误时

执行方式:
  spawn subprocess → 传 JSON stdin → 读 stdout/stderr
  hook 可返回: { block: true, message: "..." } 阻止操作
```

---

### 11. API Service (Claude API 调用)

**文件**: [src/services/api/claude.ts](src/services/api/claude.ts)

```
callModel() 请求体:
  {
    model: "claude-sonnet-4-...",
    system: [{ type: "text", text: "...", cache_control }],
    messages: [...normalized messages...],
    tools: [...tool schemas...],
    max_tokens: 16384,
    stream: true,
    metadata: { user_id },
    betas: ["prompt-caching-2024-07-31", ...]
  }

流式响应:
  message_start → content_block_start → content_block_delta → 
  content_block_stop → message_delta → message_stop

错误处理:
  - 429 rate limit → retry with backoff
  - 413 prompt too long → reactive compact or error
  - 500/529 → fallback model switch
  - withRetry() 包装所有 API 调用
```

---

### 12. Compaction 系统 (上下文压缩)

**文件**: [src/services/compact/](src/services/compact/)

```
触发时机:
  autoCompact: tokenCount > threshold (约 80% context window)
  reactive compact: API 返回 413 时紧急压缩
  manual: /compact 命令
  snip: 历史裁剪 (feature-gated)
  microcompact: 单条工具结果过大时压缩

autoCompact 流程:
  ① calculateTokenWarningState() 判断是否超阈值
  ② 选取 compact boundary (保留最近 N 条)
  ③ fork 一个摘要 agent (用小模型)
  ④ 生成对话摘要 → buildPostCompactMessages()
  ⑤ 替换原消息 → 继续 query loop

压缩层次 (从轻到重):
  snip (裁剪旧消息) → microcompact (压缩单结果) 
  → context collapse (折叠段落) → autocompact (全量摘要)
  → reactive compact (紧急 413 恢复)
```

---

### 13. Plugin 系统

**文件**: [src/utils/plugins/pluginLoader.ts](src/utils/plugins/pluginLoader.ts)

```
Plugin 结构:
  plugin.json (manifest: name, version, description)
  commands/     → 注册为 slash commands
  agents/       → 注册为 agent 定义
  hooks/        → 合并到 hook 配置
  skills/       → 合并到 skill 池

加载方式:
  ① settings.json 中声明 plugins 列表
  ② --plugin-dir CLI 参数 (session-only)
  ③ marketplace plugins (plugin@marketplace)
  
生命周期:
  loadAllPlugins() → validate manifests → merge into registries
```

---

### 14. Agent Tool (子代理)

**文件**: [src/tools/AgentTool/AgentTool.tsx](src/tools/AgentTool/AgentTool.tsx)

```
Agent 调用流程:
  模型决定调用 AgentTool(description, prompt, subagent_type)
    → loadAgentsDir() 查找 .claude/agents/ 定义
    → runAgent():
        ├─ 创建新 QueryEngine 实例 (独立 context)
        ├─ 可选: worktree 隔离 (git worktree)
        ├─ 可选: 后台执行 (run_in_background)
        ├─ 工具集可裁剪 (subagent_type 决定可用工具)
        └─ 独立 abort controller
    → yield 进度更新
    → 返回 agent 输出作为 tool_result

子代理特性:
  - 独立的 mutableMessages (不共享父对话)
  - 共享 file state cache
  - maxTurns 限制防止无限循环
  - agentId 标识，用于 session 持久化
```

---

### 15. State 管理

**文件**: [src/state/AppStateStore.ts](src/state/AppStateStore.ts)

```
AppState 核心字段:
  messages: Message[]                    // 对话历史
  toolPermissionContext: {
    mode: PermissionMode                 // plan | normal | autonomous
    additionalWorkingDirectories: Map    // 允许的额外目录
  }
  mcp: {
    clients: MCPServerConnection[]       // 活跃 MCP 连接
    tools: MCPToolDefinition[]           // 已注册的 MCP 工具
  }
  fastMode: boolean                      // 快速模式
  effortValue: EffortValue               // 努力程度
  fileStateCache: FileStateCache         // 文件状态缓存

状态流转:
  QueryEngine 通过 getAppState()/setAppState() 读写
  React 组件通过 Context 订阅渲染
```

---

### 16. 完整时序图

```
┌─────┐     ┌────────────┐    ┌─────────────┐    ┌────────┐    ┌──────┐
│User │     │processInput│    │QueryEngine  │    │query() │    │Claude│
└──┬──┘     └─────┬──────┘    └──────┬──────┘    └───┬────┘    └──┬───┘
   │  text/cmd    │                   │               │            │
   │─────────────→│                   │               │            │
   │              │ hooks (validate)  │               │            │
   │              │──→ hook process   │               │            │
   │              │                   │               │            │
   │              │ UserMessage       │               │            │
   │              │──────────────────→│               │            │
   │              │                   │ system prompt  │            │
   │              │                   │ + memory       │            │
   │              │                   │ + MCP tools    │            │
   │              │                   │ + context      │            │
   │              │                   │───────────────→│            │
   │              │                   │               │ messages   │
   │              │                   │               │───────────→│
   │              │                   │               │            │
   │              │                   │               │  stream    │
   │              │                   │               │←───────────│
   │  stream      │                   │    yield      │            │
   │←─────────────│───────────────────│←──────────────│            │
   │              │                   │               │            │
   │              │                   │  tool_use?    │            │
   │              │                   │               │──→ runTools()
   │              │                   │               │    ├ canUseTool()
   │  permission? │                   │               │    ├ tool.execute()
   │←─────────────│───────────────────│←──────────────│    │
   │  allow/deny  │                   │               │    │
   │─────────────→│──────────────────→│──────────────→│    │
   │              │                   │               │    ↓
   │              │                   │               │ tool_result
   │              │                   │               │───────────→│
   │              │                   │               │  (loop)    │
   │              │                   │               │←───────────│
   │              │                   │               │            │
   │              │                   │  no tool_use  │            │
   │              │                   │  = turn end   │            │
   │  final text  │                   │    yield      │            │
   │←─────────────│───────────────────│←──────────────│            │
   │              │                   │               │            │
   │              │                   │ persist session│            │
   │              │                   │ post hooks     │            │
```

---

### 17. 模块间依赖关系

```
                    ┌──────────┐
                    │ cli.tsx  │
                    └────┬─────┘
                         ↓
                    ┌──────────┐
                    │ main.tsx │ ←── State (AppStateStore)
                    └────┬─────┘
                         ↓
              ┌──────────────────────┐
              │    QueryEngine.ts    │
              └──────────┬───────────┘
                         │
         ┌───────────────┼───────────────────┐
         ↓               ↓                   ↓
┌─────────────┐  ┌──────────────┐   ┌──────────────────┐
│queryContext │  │   query.ts   │   │ processUserInput │
│ (prompts)   │  │ (agentic     │   │ (commands,       │
│             │  │  loop)       │   │  hooks, input)   │
└──────┬──────┘  └───────┬──────┘   └──────────────────┘
       │                 │
       ↓                 ↓
┌────────────┐   ┌───────────────────┐
│ memdir     │   │ services/api      │ ←── retry, fallback, cache
│ (memory)   │   │ (Claude API call) │
└────────────┘   └────────┬──────────┘
                          ↓
                 ┌─────────────────────┐
                 │ toolOrchestration   │
                 └────────┬────────────┘
                          │
    ┌─────────┬───────────┼──────────┬──────────┐
    ↓         ↓           ↓          ↓          ↓
┌───────┐ ┌───────┐ ┌─────────┐ ┌────────┐ ┌───────┐
│Bash   │ │File   │ │AgentTool│ │MCPTool │ │Skill  │
│Tool   │ │Tools  │ │(子agent)│ │(MCP)   │ │Tool   │
└───────┘ └───────┘ └─────────┘ └────────┘ └───────┘
                          ↑          ↑
                          │          │
                   ┌──────┘    ┌─────┘
                   │           │
              ┌────────┐  ┌──────────┐
              │Plugins │  │MCP Client│
              └────────┘  └──────────┘
```

---

### 关键洞察

1. **AsyncGenerator 贯穿全链路** — 从 `query()` 到 `QueryEngine.submitMessage()` 到 UI，全部用 `yield` 流式传递，实现逐字渲染
2. **工具循环是核心** — 模型每次返回 `tool_use` 就触发一轮执行+再次调用 API，直到纯文本响应
3. **压缩是多层次的** — snip → microcompact → collapse → autocompact → reactive compact，从轻到重逐级降级
4. **权限是统一入口** — 所有工具执行前经过 `canUseTool()`，支持 allow/deny/ask 三态
5. **子代理是递归结构** — AgentTool 创建新 QueryEngine，拥有独立对话但共享文件缓存
