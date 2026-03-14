---
title: ArkService
description: ArkService API reference — route tasks to AI agents using round-robin, least-busy, or random load balancing strategies in ark-operator.
parent: CRD Reference
nav_order: 2
---

# ArkService

**API:** `arkonis.dev/v1alpha1`
**Kind:** `ArkService`
**Short name:** `aosvc`

Analogous to a Kubernetes `Service`. Routes incoming tasks to available agent instances, decoupling task producers from the agent pool. An `ArkService` selects a target `ArkAgent` and applies a routing strategy to distribute tasks across ready pods.

## Example

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkService
metadata:
  name: research-service
  namespace: default
spec:
  selector:
    arkonisDeployment: research-agent
  routing:
    strategy: least-busy
  ports:
    - protocol: HTTP
      port: 8081
```

## Spec fields

### Top-level

| Field | Type | Required | Description |
|---|---|---|---|
| `selector` | ServiceSelector | yes | Identifies the target `ArkAgent`. |
| `routing` | RoutingSpec | no | Task routing configuration. |
| `ports` | []PortSpec | no | Ports exposed by the service. |

### `selector`

| Field | Type | Required | Description |
|---|---|---|---|
| `arkonisDeployment` | string | yes | Name of the `ArkAgent` in the same namespace to route tasks to. |

### `routing`

| Field | Type | Default | Description |
|---|---|---|---|
| `strategy` | string | `round-robin` | Routing strategy. See strategies below. |

#### Routing strategies

| Strategy | Behavior |
|---|---|
| `round-robin` | Distributes tasks sequentially across all ready pods. Simple and predictable. |
| `least-busy` | Routes each task to the pod with the fewest active tasks. Best for tasks with variable duration. |
| `random` | Selects a ready pod at random. Useful for even distribution without tracking state. |

Only pods with a passing `/readyz` check are eligible to receive tasks. Pods that fail the semantic readiness probe are excluded from routing until they recover.

### `ports[]`

| Field | Type | Required | Description |
|---|---|---|---|
| `protocol` | string | yes | `HTTP` — external task submission. `A2A` — agent-to-agent communication. |
| `port` | int | yes | Port number. |

## Status fields

| Field | Type | Description |
|---|---|---|
| `readyReplicas` | int32 | Number of ready agent pods currently receiving tasks. |
| `conditions` | []Condition | Standard Kubernetes conditions. |
