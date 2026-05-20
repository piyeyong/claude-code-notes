# Claude Code 如何与大模型通信

通过 mitmproxy 正向代理拦截 Claude Code CLI 发送给 Anthropic API 的请求，可以完整观察其与 LLM 的通信过程。本文基于实际抓包数据（3 个问题，8 次 API 调用）进行分析。

## 1. 环境搭建

### 启动 mitmweb 正向代理

```bash
mitmweb --listen-port 8080
```

启动后浏览器自动打开 `http://127.0.0.1:8081`，可视化查看所有请求。

### 配置 Claude Code 走代理

在启动 Claude Code 的终端中设置环境变量：

```powershell
$env:http_proxy = "http://localhost:8080"
$env:https_proxy = "http://localhost:8080"
$env:no_proxy = ""
```

然后启动 Claude Code：

```powershell
claude
```

此时 Claude Code 发送给 `api.anthropic.com` 的所有请求都会经过 mitmproxy，在 mitmweb 界面中可以查看。

## 2. 一个用户问题触发多少次 API 调用？

抓包发现，**用户每输入一个问题，Claude Code 会发送多次 API 请求**，而不是简单的一问一答。

以下是 3 个问题产生的 8 次请求：

| # | 类型 | 端点 | 用途 | input_tokens | output_tokens |
|---|------|------|------|-------------|---------------|
| 1 | Topic Detection | `/messages` | 检测 "summary this project" 是否为新话题 | 394 | 23 |
| 2 | Token Count | `/messages/count_tokens` | 预检 MCP 工具的 token 消耗 | — | — |
| 3 | **主对话** | `/messages` | 回答 "summary this project in short" | 23,375 | 271 |
| 4 | Topic Detection | `/messages` | 检测 "who are you?" 是否为新话题 | 396 | 23 |
| 5 | **主对话** | `/messages` | 回答 "who are you?" | 23,711 | 64 |
| 6 | Suggestion | `/messages` | 生成 ghost text 建议用户下一句话 | 24,156 | 9 |
| 7 | Topic Detection | `/messages` | 检测 "what tools and skills do you have?" 是否为新话题 | 396 | 24 |
| 8 | **主对话** | `/messages` | 回答 "what tools and skills do you have?" | 23,819 | 419 |

### 调用模式

每个用户问题的典型流程：

```
用户输入
  ├─ ① Topic Detection（轻量调用，~400 tokens，判断是否新话题）
  ├─ ② 主对话（重量级调用，~23K tokens，实际回答问题）
  └─ ③ Suggestion（可选，生成 ghost text 预测用户下一句）
```

此外，会话开始时还有一次 **Token Count** 预检调用。

## 3. 请求的 HTTP 头部

```
POST /api/anthropic/v1/messages?beta=true HTTP/1.1
User-Agent: claude-cli/2.1.62 (external, cli)
anthropic-version: 2023-06-01
anthropic-beta: claude-code-20250219,interleaved-thinking-2025-05-14,prompt-caching-scope-2026-01-05
anthropic-dangerous-direct-browser-access: true
X-Stainless-Runtime: node
X-Stainless-Runtime-Version: v24.3.0
X-Stainless-Package-Version: 0.74.0
```

关键信息：

- **anthropic-beta** 启用了三个 beta 特性：Claude Code 专用协议、交错思考、提示缓存
- **X-Stainless-\*** 表明底层使用 Stainless 生成的 Anthropic TypeScript SDK（v0.74.0）
- **Node.js v24.3.0** 作为运行时

## 4. 请求体结构

### 4.1 主对话请求

```json
{
  "model": "claude-opus-4.6",
  "max_tokens": 32000,
  "stream": true,
  "temperature": 1,
  "system": [ ...3 个 text block... ],
  "tools": [ ...23 个工具定义... ],
  "messages": [ ...完整对话历史... ],
  "metadata": { "user_id": "user_xxx_account__session_xxx" }
}
```

### 4.2 Topic Detection 请求（轻量级）

```json
{
  "model": "claude-opus-4.6",
  "max_tokens": 32000,
  "temperature": 1,
  "stream": true,
  "system": [
    { "text": "x-anthropic-billing-header: ..." },
    { "text": "You are Claude Code, Anthropic's official CLI for Claude." },
    { "text": "Analyze if this message indicates a new conversation topic..." }
  ],
  "tools": [],
  "messages": [{ "role": "user", "content": [{ "type": "text", "text": "summary this project in short" }] }],
  "output_config": {
    "format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "isNewTopic": { "type": "boolean" },
          "title": { "anyOf": [{ "type": "string" }, { "type": "null" }] }
        }
      }
    }
  }
}
```

特点：**无工具、无对话历史、使用 JSON Schema 约束输出**。仅消耗 ~400 tokens。

响应示例：`{"isNewTopic": true, "title": "Project Summary"}`

### 4.3 Token Count 预检请求

```
POST /api/anthropic/v1/messages/count_tokens?beta=true
```

```json
{
  "model": "claude-opus-4.6",
  "messages": [{ "role": "user", "content": "foo" }],
  "tools": [
    { "name": "mcp__ide__getDiagnostics", ... },
    { "name": "mcp__ide__executeCode", ... }
  ]
}
```

只发送 MCP 工具定义和一个占位消息，**不触发模型推理**，返回 `{"input_tokens": 314}`。Claude Code 用此结果来规划上下文预算。

### 4.4 Suggestion（Ghost Text）请求

在 assistant 回复之后，Claude Code 可能会再发一次请求，预测用户下一句话：

- 将完整对话历史 + 一段 `[SUGGESTION MODE]` 提示词作为最后一条 user 消息发送
- 提示词要求模型预测 2-12 个词的用户输入，不能包含评价性词语、疑问句或 Claude 口吻
- 本次抓包中模型输出了：`"how does the agentic loop work"`（仅 9 output tokens）
- 这就是 CLI 中看到的灰色 ghost text 建议

## 5. System Prompt 详解

主对话的 `system` 字段固定为 **3 个 text block**：

| Block | 内容 | 缓存 |
|-------|------|------|
| Block 0 | `x-anthropic-billing-header: cc_version=2.1.62.3d5; cc_entrypoint=cli; cch=cf6d9` | 无 |
| Block 1 | `"You are Claude Code, Anthropic's official CLI for Claude."` | `cache_control: ephemeral` |
| Block 2 | 完整指令集（~15,000 字符），含角色定义、安全策略、代码风格、工具使用规则、Memory 系统、环境信息等 | `cache_control: ephemeral` |

### Block 0：计费标识

嵌在 system prompt 中而非 HTTP 头部。`cch=` 值每次调用都不同（可能是会话/对话哈希），`cc_version` 的后缀也会变化（`e73`、`3d5`、`3e0`、`416`）。

### Block 2：指令集包含的内容

- 角色与行为规范（安全策略、代码风格、任务执行原则）
- 工具使用指南（何时用 Read/Edit/Bash/Glob/Grep、何时用 Agent 子任务）
- Auto Memory 系统（持久化记忆的读写规则）
- 环境信息（工作目录、平台、Shell、模型 ID）
- Git 状态（当前分支、最近提交）
- VSCode 集成信息（编辑器上下文、文件引用格式）

## 6. Tools（23 个工具定义）

主对话请求中携带 23 个工具定义：

| 类别 | 工具 | 说明 |
|------|------|------|
| 文件操作 | `Read`, `Write`, `Edit`, `NotebookEdit` | 读写编辑文件 |
| 搜索 | `Glob`, `Grep` | 文件名匹配、内容正则搜索 |
| 执行 | `Bash` | 执行 shell 命令 |
| 网络 | `WebFetch`, `WebSearch` | 获取网页、搜索 |
| Agent | `Task`, `TaskOutput`, `TaskStop` | 启动/监控/停止子 Agent |
| 任务管理 | `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList` | 结构化任务跟踪 |
| 交互 | `AskUserQuestion`, `Skill` | 向用户提问、调用技能 |
| 规划 | `EnterPlanMode`, `ExitPlanMode` | 进入/退出计划模式 |
| Git | `EnterWorktree` | 创建隔离的 git worktree |
| MCP | `mcp__ide__getDiagnostics`, `mcp__ide__executeCode` | VS Code 语言诊断、Jupyter 执行 |

每个工具定义包含 `name`、`description`（详细的使用说明）和 `input_schema`（JSON Schema 参数定义）。

## 7. Messages（对话历史）

### 首条 User 消息的特殊结构

Claude Code 在第一条 user 消息中注入了 **7 个 content block**，实际用户输入只是最后一个：

| Block | 标签 | 内容 |
|-------|------|------|
| 0 | `<system-reminder>` | Startup hook 输出（项目根目录 `ls` 结果） |
| 1 | `<system-reminder>` | 可用 Skills 列表 |
| 2 | `<system-reminder>` | CLAUDE.md 完整内容（~3,618 字符） |
| 3 | `<local-command-caveat>` | 本地命令告警 |
| 4 | `<command-name>` | 触发的斜杠命令（如 `/plugin`） |
| 5 | `<local-command-stdout>` | 命令输出（此处为空） |
| 6 | （纯文本） | **用户实际输入**："summary this project in short" |

> Skills 列表和 CLAUDE.md 内容之所以放在 user 消息而非 system prompt 中，可能是为了利用 system prompt 的缓存——system prompt 保持不变以命中缓存，而这些动态内容放在 messages 中。

### 后续 User 消息

后续消息简单得多，只包含用户输入文本，并带有 `cache_control: ephemeral`：

```json
{ "role": "user", "content": [{ "type": "text", "text": "who are you?", "cache_control": { "type": "ephemeral" } }] }
```

### 对话历史的增长

每次请求都发送**完整对话历史**：

- 第 1 轮（Q1）：1 条 user 消息
- 第 2 轮（Q2）：3 条消息（user + assistant + user）
- 第 3 轮（Q3）：5 条消息（user + assistant + user + assistant + user）

input_tokens 从 23,375 → 23,711 → 23,819 逐步增长，增量来自历史消息。

## 8. 流式响应（SSE）

所有请求均使用 `stream: true`，响应以 Server-Sent Events 格式返回：

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_xxx","model":"claude-opus-4.6","usage":{...}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"I'm "}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Claude Code..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":64,...}}

event: message_stop
data: {"type":"message_stop"}
```

| 事件 | 说明 |
|------|------|
| `message_start` | 消息开始，含 model、初始 usage |
| `content_block_start` | 内容块开始（text 或 tool_use） |
| `content_block_delta` | 增量文本片段或工具参数 JSON 片段 |
| `content_block_stop` | 内容块结束 |
| `message_delta` | stop_reason、最终 usage |
| `message_stop` | 消息结束 |

## 9. Prompt Caching

### 缓存标记

System prompt 的 Block 1 和 Block 2 带有 `cache_control: {"type": "ephemeral"}`，最后一条 user 消息也带此标记。

### 抓包观察

本次 8 次请求中，**所有调用的 `cache_creation_input_tokens` 和 `cache_read_input_tokens` 均为 0**。可能原因：

1. 使用了 mitmproxy 代理，代理转发可能影响了缓存键的一致性
2. `prompt-caching-scope-2026-01-05` beta 可能对缓存生效条件有特殊限制
3. Topic Detection 调用的 system prompt 结构与主对话不同，无法复用缓存

在正常直连 API 的情况下，缓存应该能在同一会话的多次主对话请求之间命中，避免重复处理 ~23K tokens 的 system prompt。

## 10. Token 用量分析

| 请求类型 | 单次 input_tokens | 单次 output_tokens | 占比 |
|----------|-------------------|--------------------|----- |
| Topic Detection | ~395 | ~23 | 极低 |
| Token Count | 不推理 | 不推理 | 无 |
| 主对话 | ~23,000-24,000 | 64-419 | **主要开销** |
| Suggestion | ~24,000 | ~9 | 高输入低输出 |

**关键发现：**

- 主对话的 input tokens 中，system prompt + tools 定义占绝大部分（~23K），用户消息和对话历史只占几百 tokens
- 如果 Prompt Caching 正常工作，这 ~23K tokens 在同一会话中只需处理一次
- Suggestion 调用虽然 output 极少（9 tokens），但 input 与主对话相当（~24K），成本不容忽视

## 11. 有趣的发现

1. **计费标识嵌在 system prompt 中** — `x-anthropic-billing-header` 作为 system 的第一个 text block，而非 HTTP 头部，使计费元数据跟随模型请求
2. **cc_version 后缀不同** — 同一会话中 Topic Detection 和主对话的 `cc_version` 后缀不同（如 `e73` vs `3d5`），这些可能是构建哈希或调用链标识
3. **JSON Schema 输出约束** — Topic Detection 使用 `output_config.format.json_schema` 强制模型输出结构化 JSON，但模型仍然在 JSON 外包裹了 ` ```json ``` ` markdown 代码块
4. **`anthropic-dangerous-direct-browser-access: true`** — 绕过 CORS 限制的头部，因为 Claude Code 可能在浏览器可访问的上下文中运行
5. **首条消息注入了大量上下文** — Skills 列表、CLAUDE.md、startup hook 输出都塞在第一条 user 消息中，而非 system prompt，可能是为了保持 system prompt 的缓存稳定性

## 12. 完整的调用时序图

```
用户输入 "summary this project in short"
│
├─ [1] Topic Detection ──→ API ──→ {"isNewTopic": true, "title": "Project Summary"}
├─ [2] Token Count ──→ API ──→ {"input_tokens": 314}  (仅 MCP 工具预检)
├─ [3] 主对话 ──→ API ──→ 项目摘要文本（271 output tokens）
│
用户输入 "who are you?"
│
├─ [4] Topic Detection ──→ API ──→ {"isNewTopic": true, "title": "Self Introduction"}
├─ [5] 主对话 ──→ API ──→ 自我介绍文本（64 output tokens）
├─ [6] Suggestion ──→ API ──→ "how does the agentic loop work"（ghost text）
│
用户输入 "what tools and skills do you have?"
│
├─ [7] Topic Detection ──→ API ──→ {"isNewTopic": true, "title": "Tools and Skills"}
└─ [8] 主对话 ──→ API ──→ 工具和技能列表（419 output tokens）
```

## 13. 小结

Claude Code 不只是简单地"一问一答"调用 API。每个用户输入会触发 **2-3 次 API 调用**：

1. **Topic Detection** — 轻量级调用（~400 tokens），用结构化 JSON Schema 判断是否为新话题，用于 UI 标题
2. **主对话** — 重量级调用（~23K+ tokens），携带完整 system prompt、23 个工具定义和全部对话历史
3. **Suggestion**（可选）— 生成 ghost text，预测用户下一句话

此外还有 **Token Count 预检** 用于上下文预算规划。

这种架构意味着：
- **实际 API 调用量** 是用户感知的 2-3 倍
- **Prompt Caching 至关重要** — 每次主对话都发送 ~23K tokens 的 system prompt + tools，缓存命中可显著降低成本和延迟
- **首条消息承担了上下文注入** — Skills、CLAUDE.md、hook 输出等动态内容放在 user 消息而非 system prompt 中，保证 system prompt 的缓存稳定性

通过 mitmproxy 拦截，可以完全透明地观察这一过程，是理解 Agent 架构的绝佳方式。
