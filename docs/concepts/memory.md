---
title: Agent Memory
description: How ark-operator gives AI agents persistent memory across tasks using Redis or vector store backends.
parent: Concepts
nav_order: 4
---

# Agent Memory

By default, each agent task starts with no knowledge of previous tasks. The agent receives its system prompt, the current task, and any tools — nothing else. This is fine for stateless workloads but breaks down for agents that need to learn from past interactions, avoid repeating work, or maintain context across a long-running job.

`ArkMemory` gives agents durable memory that persists across tasks and pod restarts.

## How it works

The operator injects memory configuration into agent pods as environment variables. The agent runtime reads these at startup and connects to the configured backend before polling the task queue.

During a task, the runtime:
1. Queries the memory backend for relevant past context and prepends it to the prompt
2. Writes the task result back to the memory backend after completion

The LLM never calls the memory backend directly. The runtime handles all reads and writes.

## Backends

### in-context

No external dependency. Memory is carried forward within a single task's context window only. Once the task ends, the memory is gone.

Use this when your agents are stateless by design or when you are prototyping.

### redis

Short-term memory backed by Redis. Entries are stored as key-value pairs and expire via a configurable TTL. Reads are fast; writes are synchronous before the task result is acknowledged.

Use this for session-scoped context, recent interaction history, or deduplication state.

```yaml
spec:
  backend: redis
  redis:
    secretRef:
      name: redis-credentials   # must contain REDIS_URL
    ttlSeconds: 3600
    maxEntries: 500
```

### vector-store

Long-term semantic memory backed by a vector database (Qdrant, Pinecone, or Weaviate). Past task results are embedded and stored as vectors. At the start of each task, the runtime performs a similarity search and retrieves the most relevant past context.

Use this when agents need to recall information from days or weeks ago, or when the memory corpus is too large to scan linearly.

```yaml
spec:
  backend: vector-store
  vectorStore:
    provider: qdrant
    endpoint: http://qdrant.agent-infra.svc.cluster.local:6333
    collection: agent-memories
```

## Attaching memory to an agent

Create the `ArkMemory` resource, then reference it from the `ArkAgent`:

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkMemory
metadata:
  name: research-memory
  namespace: default
spec:
  backend: redis
  redis:
    secretRef:
      name: redis-credentials
    ttlSeconds: 7200
---
apiVersion: arkonis.dev/v1alpha1
kind: ArkAgent
metadata:
  name: research-agent
  namespace: default
spec:
  model: claude-sonnet-4-20250514
  systemPrompt: "You are a research agent."
  memoryRef:
    name: research-memory
```

The same `ArkMemory` can be referenced by multiple `ArkAgent` resources in the same namespace.

## Failure behavior

If the memory backend is unreachable at pod startup, the agent logs the error and continues without memory. Tasks are not blocked. This matches the same non-fatal philosophy as MCP server connection failures.

If a memory write fails after a task completes, the failure is logged but the task result is still acknowledged. Memory writes are best-effort and do not affect task delivery guarantees.

## Environment variables injected

The operator injects these into every agent pod that has a `memoryRef`:

| Variable | Description |
|---|---|
| `AGENT_MEMORY_BACKEND` | `in-context`, `redis`, or `vector-store` |
| `AGENT_MEMORY_REDIS_URL` | From the referenced Secret (redis backend) |
| `AGENT_MEMORY_REDIS_TTL` | TTL in seconds |
| `AGENT_MEMORY_REDIS_MAX_ENTRIES` | Entry cap |
| `AGENT_MEMORY_VECTOR_STORE_PROVIDER` | `qdrant`, `pinecone`, or `weaviate` |
| `AGENT_MEMORY_VECTOR_STORE_ENDPOINT` | Base URL |
| `AGENT_MEMORY_VECTOR_STORE_COLLECTION` | Collection name |
| `AGENT_MEMORY_VECTOR_STORE_API_KEY` | From the referenced Secret (if set) |
| `AGENT_MEMORY_VECTOR_STORE_TTL` | TTL in seconds |

## See also

- [ArkMemory reference](/docs/crds/agent-memory) — full field reference
- [ArkAgent reference](/docs/crds/agent-deployment) — `spec.memoryRef` field
