[中文版](README.zh.md)

# Claude Code Notes

Learning notes for Claude Code, documenting tips and techniques along the way.

## Table of Contents

### [mitmproxy Guide](docs/en/mitmproxy%20Guide.md)

Use mitmproxy to intercept HTTP traffic between a local agent loop and the Anthropic LLM API. This makes it easy to inspect the exact messages sent and received — including tool-use calls and responses — for debugging agent behavior.

### [How Claude Code Talks to the Model](docs/en/How%20Claude%20Code%20Talks%20to%20the%20Model.md)

Based on actual mitmproxy packet capture (3 questions, 8 API calls), analyzing how Claude Code communicates with the Anthropic API: request types (topic detection, main conversation, suggestion, token count), system prompt structure, tool definitions, message history, SSE streaming, and prompt caching.

### [Claude Code Source Code Architecture — Deep Analysis](docs/en/Claude%20Code%20Source%20Code%20Architecture%20Deep%20Analysis.md)

Comprehensive module-by-module architectural trace based on decompiled Claude Code v2.1.88.

### [Claude Code Full Chain Analysis — Complete Module Trace from Input to Response](docs/en/Claude%20Code%20Full%20Chain%20Analysis%20-%20Complete%20Module%20Trace%20from%20Input%20to%20Response.md)

End-to-end architectural walkthrough of Claude Code: from entry points (REPL/CLI/SDK) through input processing, QueryEngine, the core query loop, tool orchestration, MCP, skills, hooks, compaction, plugins, sub-agents, and state management — with full sequence and dependency diagrams.

### [Claude Code Query Loop Implementation Deep Dive](docs/en/Claude%20Code%20Query%20Loop%20Implementation%20Deep%20Dive.md)

Deep dive into the `query()` AsyncGenerator: the six-stage iteration cycle (compact → API call → error recovery → tool execution → attachment injection → loop decision), StreamingToolExecutor concurrency model, tool permission system, and architectural trade-offs.

### [Claude Code System Prompt Implementation Deep Dive](docs/en/Claude%20Code%20System%20Prompt%20Implementation%20Deep%20Dive.md)

How Claude Code assembles its system prompt: static/dynamic section split, `systemPromptSection()` memoization, BOUNDARY marker for global cache scope, prompt cache hit mechanics (byte-prefix matching, beta header latching, message-level caching), and architectural analysis.

### [Claude Code Sub-Agent Deep Dive](docs/en/Claude%20Code%20SubAgent.md)

Deep dive into the Sub-Agent system: AgentTool as a regular tool, Fork vs Normal paths, context isolation, built-in agent types, Prompt Cache optimization for forked agents, result return mechanics, and worktree isolation.
