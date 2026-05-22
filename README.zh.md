[English](README.md)

# Claude Code 学习笔记

Claude Code 的学习笔记，记录使用过程中的技巧和心得。

## 目录

### [mitmproxy 抓包指南](docs/zh/mitmproxy%20抓包指南.md)

使用 mitmproxy 拦截本地 Agent 循环与 Anthropic LLM API 之间的 HTTP 流量，方便检查发送和接收的完整消息——包括 tool-use 调用和响应——用于调试 Agent 行为。

### [Claude Code 如何与大模型通信](docs/zh/Claude%20Code%20如何与大模型通信.md)

基于 mitmproxy 实际抓包数据（3 个问题、8 次 API 调用），分析 Claude Code 与 Anthropic API 的通信过程：请求类型（话题检测、主对话、建议补全、Token 计数）、System Prompt 结构、工具定义、消息历史、SSE 流式响应及 Prompt Caching。

### [Claude Code 源码架构深度分析](docs/zh/Claude%20Code%20源码分析.md)

基于 Claude Code v2.1.88 反编译源码的全模块架构追踪。

### [Claude Code 完整链路分析：从输入到响应的全模块追踪](docs/zh/Claude%20Code%20完整链路分析：从输入到响应的全模块追踪.md)

端到端架构走读：从入口层（REPL/CLI/SDK）到输入处理、QueryEngine、核心 query 循环、工具编排、MCP、Skills、Hooks、上下文压缩、插件系统、子代理与状态管理——附完整时序图和模块依赖图。

### [Claude Code Query Loop 实现详解](docs/zh/Claude%20Code%20Query%20Loop实现详解.md)

深入剖析 `query()` AsyncGenerator：六阶段迭代循环（Compact → API 调用 → 错误恢复 → 工具执行 → Attachment 注入 → 循环决策）、StreamingToolExecutor 并发模型、Tool Permission 权限系统及架构分析。

### [Claude Code System Prompt 实现详解](docs/zh/Claude%20Code%20System%20Prompt%20实现详解.md)

System Prompt 的组装机制：静态/动态 Section 分割、`systemPromptSection()` 缓存、BOUNDARY Marker 与全局缓存范围、Prompt Cache 命中机制（字节前缀匹配、Beta Header Latching、Message 级缓存）及架构设计分析。

### [Claude Code 子 Agent 深度分析](docs/zh/Claude%20Code%20子%20Agent%20深度分析.md)

Sub Agent 系统全链路追踪：AgentTool 作为普通工具的调度机制、Fork vs Normal 两种路径、上下文隔离、内置 Agent 类型、Fork 的 Prompt Cache 优化、结果返回机制及 Worktree 隔离。
