---
title: ArkAgent
description: ArkAgent API reference — manage pools of AI agent instances with configurable models, system prompts, MCP tool servers, and inline webhook tools on Kubernetes.
parent: CRD Reference
nav_order: 1
---

# ArkAgent

**API:** `arkonis.dev/v1alpha1`
**Kind:** `ArkAgent`
**Short name:** `arkagent`

The core resource. Analogous to a Kubernetes `Deployment` — manages a pool of agent instances running the same model, prompt, and tool configuration. The operator creates and maintains a backing `Deployment` for each `ArkAgent`.

## Example

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkAgent
metadata:
  name: research-agent
  namespace: default
spec:
  replicas: 2
  model: claude-sonnet-4-20250514
  systemPrompt: |
    You are a research agent. Gather and summarize information
    accurately and concisely. Always cite your sources.
  mcpServers:
    - name: web-search
      url: https://search.mcp.internal/sse
  tools:
    - name: fetch_news
      description: "Fetch the latest news headlines for a given topic."
      url: "http://news-api.internal/headlines"
      method: POST
      inputSchema: |
        {"type":"object","properties":{"topic":{"type":"string"}},"required":["topic"]}
  limits:
    maxTokensPerCall: 8000
    maxConcurrentTasks: 5
    timeoutSeconds: 120
  livenessProbe:
    type: semantic
    intervalSeconds: 60
```

## Spec fields

### Top-level

| Field | Type | Required | Description |
|---|---|---|---|
| `replicas` | int32 | no | Number of agent pod replicas. Defaults to `1`. Range: 0–50. |
| `model` | string | yes | Model identifier passed to the configured LLM provider (e.g. `claude-sonnet-4-20250514` for Anthropic). |
| `systemPrompt` | string | yes | System prompt injected into every API call made by agent pods. |
| `mcpServers` | []MCPServerSpec | no | List of MCP servers to connect at pod startup. See [MCP Servers](/docs/concepts/mcp-servers). |
| `tools` | []WebhookToolSpec | no | Inline HTTP webhook tools available to agents without a full MCP server. The operator serialises these into the `AGENT_WEBHOOK_TOOLS` environment variable. |
| `limits` | AgentLimits | no | Per-agent resource and token limits. |
| `livenessProbe` | AgentProbe | no | Semantic health check configuration. |
| `configRef` | LocalObjectReference | no | Name of an `ArkSettings` in the same namespace. Merged into effective system prompt and model settings. |
| `memoryRef` | LocalObjectReference | no | Name of an `ArkMemory` in the same namespace. Injects memory backend config into agent pods. See [Agent Memory](/docs/concepts/memory). |

### `mcpServers[]`

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Logical name. Used as tool name prefix (`name__tool`). |
| `url` | string | yes | SSE endpoint URL of the MCP server. |

### `tools[]` (inline webhook tools)

Define simple HTTP tools directly in the agent spec — no MCP server required. The agent runtime calls the configured URL when the LLM invokes the tool and returns the response body to the model.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Tool identifier exposed to the LLM. Must be unique within the deployment. |
| `description` | string | no | Explains the tool's purpose to the LLM. |
| `url` | string | yes | HTTP endpoint the agent calls when the tool is invoked. |
| `method` | string | no | HTTP method (`GET`, `POST`, `PUT`, `PATCH`). Defaults to `POST`. |
| `inputSchema` | string | no | JSON Schema (as a raw JSON string) describing the tool's input parameters. The LLM uses this to construct valid inputs. |

### `limits`

| Field | Type | Default | Description |
|---|---|---|---|
| `maxTokensPerCall` | int | `8000` | Token budget (input + output) per LLM API call. |
| `maxConcurrentTasks` | int | `5` | Max tasks a single pod processes simultaneously. |
| `timeoutSeconds` | int | `120` | Per-task timeout. Task is abandoned and an error is returned after this duration. |

### `livenessProbe`

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | string | `ping` | `ping` — HTTP reachability only. `semantic` — enables `/readyz` API validation. |
| `intervalSeconds` | int | `60` | Interval between semantic validation checks. |
| `validatorPrompt` | string | *(internal default)* | Custom prompt sent to the LLM during `/readyz` semantic validation. Falls back to a built-in default if not set. |

## Built-in agent tools

In addition to MCP and webhook tools, every agent pod has access to the following built-in tool:

| Tool | Description |
|---|---|
| `submit_subtask` | Enqueue a new agent task for asynchronous processing. Takes `{"prompt": "..."}` and returns the assigned task ID. Enables the supervisor/worker pattern where a running agent spawns sub-tasks at runtime. |

## Status fields

| Field | Type | Description |
|---|---|---|
| `replicas` | int32 | Total number of agent pods managed by this deployment. |
| `readyReplicas` | int32 | Number of pods passing both liveness and readiness checks. |
| `conditions` | []Condition | Standard Kubernetes conditions (Available, Progressing, Degraded). |

```bash
kubectl describe arkagent research-agent
# ...
# Status:
#   Ready Replicas:  2
#   Replicas:        2
#   Conditions:
#     Type:    Available
#     Status:  True
```
