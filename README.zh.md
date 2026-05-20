[English](README.md)

# Claude Code 学习笔记

Claude Code 的学习笔记，记录使用过程中的技巧和心得。

## 目录

### [mitmproxy 抓包指南](docs/zh/mitmproxy%20抓包指南.md)

使用 mitmproxy 拦截本地 Agent 循环与 Anthropic LLM API 之间的 HTTP 流量，方便检查发送和接收的完整消息——包括 tool-use 调用和响应——用于调试 Agent 行为。

### [Claude Code 如何与大模型通信](docs/zh/Claude%20Code%20如何与大模型通信.md)

基于 mitmproxy 实际抓包数据（3 个问题、8 次 API 调用），分析 Claude Code 与 Anthropic API 的通信过程：请求类型（话题检测、主对话、建议补全、Token 计数）、System Prompt 结构、工具定义、消息历史、SSE 流式响应及 Prompt Caching。
