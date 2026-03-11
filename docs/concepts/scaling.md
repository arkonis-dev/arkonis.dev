---
title: Scaling
description: Scale AI agent deployments on Kubernetes manually or automatically based on task queue depth with agentops-operator.
parent: Concepts
nav_order: 3
---

# Scaling

## Manual scaling

In v1alpha1, replica count is set directly in `spec.replicas`:

```yaml
spec:
  replicas: 5
```

The operator reconciles the backing pod count to match. Scale up or down by editing the `AgentDeployment` and applying it:

```bash
kubectl patch agdep research-agent --type=merge -p '{"spec":{"replicas":10}}'
```

## Queue-depth autoscaling (planned)

CPU and memory are poor proxies for agent load. What matters is task queue depth — how many tasks are waiting to be processed.

Queue-depth-based autoscaling via [KEDA](https://keda.sh/) is planned for v1beta1. The `AgentScaler` resource will allow defining scale-up/scale-down triggers based on Redis Streams queue length, with configurable thresholds and cooldown periods.

## Resource limits

The `spec.limits` block controls per-agent resource consumption, not Kubernetes CPU/memory limits (those are set on the backing pods separately):

```yaml
spec:
  limits:
    maxTokensPerCall: 8000
    maxConcurrentTasks: 5
    timeoutSeconds: 120
```

| Field | Type | Default | Description |
|---|---|---|---|
| `maxTokensPerCall` | int | `4096` | Maximum tokens (input + output) per Anthropic API call. |
| `maxConcurrentTasks` | int | `1` | Maximum tasks a single agent pod will process simultaneously. |
| `timeoutSeconds` | int | `60` | Per-task timeout. The agent pod abandons the task and returns an error after this duration. |

These values are injected as environment variables (`AGENT_MAX_TOKENS`, `AGENT_TIMEOUT_SECONDS`) into agent pods and enforced by the agent runtime.
