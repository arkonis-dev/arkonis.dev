---
title: MCP Servers
description: Connect MCP (Model Context Protocol) tool servers to your AI agents running on Kubernetes with agentops-operator.
parent: Concepts
nav_order: 2
---

# MCP Servers

## What is MCP?

[Model Context Protocol](https://modelcontextprotocol.io/) (MCP) is a standard interface for providing tools and context to LLMs. An MCP server exposes a set of callable tools — functions the model can invoke during a conversation. Examples: web search, database queries, file access, API calls.

agentops-operator connects agent pods to MCP servers at startup, making those tools available to the Anthropic API tool-use loop.

## How agentops-operator connects

MCP servers are specified in `spec.mcpServers`. At pod startup, the agent runtime reads this list from the `AGENT_MCP_SERVERS` environment variable (injected as a JSON array by the operator) and establishes SSE (Server-Sent Events) connections to each server.

The connection is established before the agent begins polling the task queue. Tool definitions from all connected MCP servers are collected and passed to the Anthropic API on every call.

## Spec field

```yaml
spec:
  mcpServers:
    - name: web-search
      url: https://search.mcp.internal/sse
    - name: memory
      url: https://memory.mcp.internal/sse
```

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Logical name for this server. Used as a prefix for tool names. |
| `url` | string | yes | SSE endpoint URL for the MCP server. |

## Tool name prefixing

To avoid collisions when multiple MCP servers expose tools with the same name, agentops-operator prefixes tool names with the server name using double underscore:

```
web-search__search
web-search__fetch_page
memory__store
memory__recall
```

The Anthropic API receives the prefixed names; the runtime strips the prefix before forwarding calls to the MCP server.

## Failure behavior

MCP server connection failures are non-fatal. If a server is unreachable at startup, the agent logs the error and continues with a reduced toolset — only the tools from servers that connected successfully are available. This prevents a single unavailable tool server from taking down the entire agent pool.

If a tool call fails at runtime (e.g., the MCP server becomes unavailable mid-task), the error is returned to the model as a tool result. The model decides how to proceed.
