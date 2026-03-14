---
title: ArkMemory
description: ArkMemory API reference — configure persistent memory backends (Redis or vector store) for AI agents managed by ark-operator.
parent: CRD Reference
nav_order: 5
---

# ArkMemory

**API:** `arkonis.dev/v1alpha1`
**Kind:** `ArkMemory`
No short name

Defines a persistent memory backend for agent instances. Reference it from an `ArkAgent` via `spec.memoryRef` to give agents durable memory across tasks.

Analogous to a `PersistentVolumeClaim` — it declares where and how memory is stored, and `ArkAgent` claims it by name.

## Example: Redis backend

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
    ttlSeconds: 3600
    maxEntries: 500
```

## Example: Vector store backend

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkMemory
metadata:
  name: research-memory-vector
  namespace: default
spec:
  backend: vector-store
  vectorStore:
    provider: qdrant
    endpoint: http://qdrant.agent-infra.svc.cluster.local:6333
    collection: agent-memories
    secretRef:
      name: qdrant-credentials
    ttlSeconds: 86400
```

## Referencing from ArkAgent

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkAgent
metadata:
  name: research-agent
spec:
  model: claude-sonnet-4-20250514
  systemPrompt: "You are a research agent."
  memoryRef:
    name: research-memory
```

## Spec fields

### Top-level

| Field | Type | Required | Description |
|---|---|---|---|
| `backend` | string | yes | Memory storage strategy. One of `in-context`, `redis`, `vector-store`. |
| `redis` | RedisMemoryConfig | conditional | Required when `backend` is `redis`. |
| `vectorStore` | VectorStoreMemoryConfig | conditional | Required when `backend` is `vector-store`. |

### `redis`

| Field | Type | Default | Description |
|---|---|---|---|
| `secretRef.name` | string | required | Name of a Secret in the same namespace. The Secret must contain a `REDIS_URL` key. |
| `ttlSeconds` | int | `3600` | How long memory entries are retained. `0` means no expiry. |
| `maxEntries` | int | `0` (unlimited) | Maximum number of stored entries per agent instance. Oldest entries are evicted when the limit is reached. |

### `vectorStore`

| Field | Type | Default | Description |
|---|---|---|---|
| `provider` | string | required | Vector database provider. One of `qdrant`, `pinecone`, `weaviate`. |
| `endpoint` | string | required | Base URL of the vector database. |
| `collection` | string | `agent-memories` | Collection or index name to store memories in. |
| `secretRef.name` | string | no | Name of a Secret containing a `VECTOR_STORE_API_KEY` key. Required for hosted providers. |
| `ttlSeconds` | int | `0` | How long memory entries are retained. `0` means no expiry. |

## Status fields

| Field | Type | Description |
|---|---|---|
| `conditions` | []Condition | Standard Kubernetes conditions. `Ready=True` means the spec is valid and the memory backend config has been accepted. |
| `observedGeneration` | int64 | The `.metadata.generation` this status reflects. |

```bash
kubectl get arkmemory
# NAME               BACKEND        AGE
# research-memory    redis          2m

kubectl describe arkmemory research-memory
# ...
# Status:
#   Conditions:
#     Type:    Ready
#     Status:  True
#     Reason:  Accepted
```

## Backend comparison

| Backend | Best for | Notes |
|---|---|---|
| `in-context` | Stateless agents, simple tasks | No external dependency. Memory lives only within a single task's context window. |
| `redis` | Short-term memory, session state | Fast reads and writes. Entries expire via TTL. |
| `vector-store` | Long-term semantic memory, RAG | Similarity search across past interactions. Requires a running vector database. |

## See also

- [Agent Memory concept guide](/docs/concepts/memory)
- [ArkAgent reference](/docs/crds/agent-deployment) — `spec.memoryRef` field
