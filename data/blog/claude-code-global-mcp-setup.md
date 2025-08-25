---
title: 'Claude Code 添加全局 MCP 配置方法'
date: '2025-01-22'
tags: ['claude', 'mcp', 'ai-tools', '配置']
draft: false
summary: '简单说明如何在 Claude Code 中使用 -s user 参数添加全局 MCP 服务配置。'
---

## 背景

Claude Code 支持通过 MCP (Model Context Protocol) 连接各种外部服务来扩展功能。要让这些 MCP 服务在全局范围内可用，需要使用 `-s user` 参数进行配置。

## 添加全局 MCP 的方法

### 基本语法

```bash
claude mcp add -s user --transport sse <服务名> <服务URL>
```

### 具体示例

#### 1. 添加 CopilotKit MCP

```bash
claude mcp add -s user --transport sse CopilotKit https://mcp.copilotkit.ai/sse
```

#### 2. 添加 Context7 MCP

```bash
claude mcp add -s user --transport sse context7 https://mcp.context7.com/sse
```

## 参数说明

- `-s user`: 指定为用户级别的全局配置
- `--transport sse`: 指定传输协议为 Server-Sent Events
- `<服务名>`: 自定义的服务名称
- `<服务URL>`: MCP 服务的端点地址

## 总结

通过添加 `-s user` 参数，可以将 MCP 服务配置为全局可用，无需在每个项目中重复配置。配置完成后，这些 MCP 服务将在所有 Claude Code 会话中自动可用。
