[中文版](README.zh.md)

# Claude Code Notes

Learning notes for Claude Code, documenting tips and techniques along the way.

## Table of Contents

### [mitmproxy Guide](docs/en/mitmproxy-guide.md)

Use mitmproxy to intercept HTTP traffic between a local agent loop and the Anthropic LLM API. This makes it easy to inspect the exact messages sent and received — including tool-use calls and responses — for debugging agent behavior.

### [How Claude Code Talks to the Model](docs/en/How%20Claude%20Code%20Talk%20to%20Model.md)

Based on actual mitmproxy packet capture (3 questions, 8 API calls), analyzing how Claude Code communicates with the Anthropic API: request types (topic detection, main conversation, suggestion, token count), system prompt structure, tool definitions, message history, SSE streaming, and prompt caching.

### [Claude Code Source Code Architecture — Deep Analysis](docs/en/Claude%20Code%20Source%20Code%20Analyze.md)

Comprehensive module-by-module architectural trace based on decompiled Claude Code v2.1.88.
