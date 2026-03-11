---
title: AgentDeployment
description: AgentDeployment API reference — manage pools of AI agent instances with configurable models, system prompts, and MCP tool servers on Kubernetes.
parent: CRD Reference
nav_order: 1
---

# AgentDeployment

**API:** `agentops.agentops.io/v1alpha1`
**Kind:** `AgentDeployment`
**Short name:** `agdep`

The core resource. Analogous to a Kubernetes `Deployment` — manages a pool of agent instances running the same model, prompt, and tool configuration. The operator creates and maintains a backing `Deployment` and `Service` for each `AgentDeployment`.

## Example

```yaml
apiVersion: agentops.agentops.io/v1alpha1
kind: AgentDeployment
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
    - name: memory
      url: https://memory.mcp.internal/sse
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
| `replicas` | int32 | no | Number of agent pod replicas. Defaults to `1`. |
| `model` | string | yes | Model identifier passed to the Anthropic API (e.g. `claude-sonnet-4-20250514`). |
| `systemPrompt` | string | yes | System prompt injected into every API call made by agent pods. |
| `mcpServers` | []MCPServerSpec | no | List of MCP servers to connect at pod startup. See [MCP Servers](../concepts/mcp-servers). |
| `limits` | AgentLimits | no | Per-agent resource and token limits. |
| `livenessProbe` | AgentProbe | no | Semantic health check configuration. |
| `configRef` | string | no | Name of an `AgentConfig` in the same namespace. Merged into effective system prompt and model settings. |

### `mcpServers[]`

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Logical name. Used as tool name prefix (`name__tool`). |
| `url` | string | yes | SSE endpoint URL of the MCP server. |

### `limits`

| Field | Type | Default | Description |
|---|---|---|---|
| `maxTokensPerCall` | int | `4096` | Token budget (input + output) per Anthropic API call. |
| `maxConcurrentTasks` | int | `1` | Max tasks a single pod processes simultaneously. |
| `timeoutSeconds` | int | `60` | Per-task timeout. Task is abandoned and an error is returned after this duration. |

### `livenessProbe`

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | string | `ping` | `ping` — HTTP reachability only. `semantic` — enables `/readyz` API validation. |
| `intervalSeconds` | int | `60` | Interval between semantic validation checks. |
| `validatorPrompt` | string | *(internal)* | Custom validation prompt. Planned for v1beta1; currently unused. |

## Status fields

| Field | Type | Description |
|---|---|---|
| `replicas` | int32 | Total number of agent pods managed by this deployment. |
| `readyReplicas` | int32 | Number of pods passing both liveness and readiness checks. |
| `conditions` | []Condition | Standard Kubernetes conditions (Available, Progressing, Degraded). |

```bash
kubectl describe agdep research-agent
# ...
# Status:
#   Ready Replicas:  2
#   Replicas:        2
#   Conditions:
#     Type:    Available
#     Status:  True
```
