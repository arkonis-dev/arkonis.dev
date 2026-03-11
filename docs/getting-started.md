---
title: Getting Started
nav_order: 2
---

# Getting Started

## Prerequisites

- Kubernetes 1.26+
- `kubectl` configured against your cluster
- An Anthropic API key

For local development, create a cluster first:

```bash
kind create cluster --name agent-operator-dev
```

## Install

### 1. Install the operator

```bash
kubectl apply -f https://github.com/agentops-io/agent-operator/releases/latest/download/install.yaml
```

This installs the CRDs, RBAC, and the operator deployment in the `agent-operator-system` namespace.

### 2. Deploy Redis

agent-operator uses Redis Streams as the task queue between the operator and agent pods.

```bash
kubectl apply -f https://raw.githubusercontent.com/agentops-io/agent-operator/main/config/prereqs/redis.yaml
```

### 3. Create the API key secret

Create one secret per namespace where agents will run:

```bash
kubectl create secret generic agent-operator-api-keys \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-... \
  --from-literal=TASK_QUEUE_URL=redis.agent-infra.svc.cluster.local:6379
```

### 4. Deploy your first agent

```bash
kubectl apply -f https://raw.githubusercontent.com/agentops-io/agent-operator/main/config/samples/agentops_v1alpha1_agentdeployment.yaml
```

## Verify

```bash
kubectl get agdep
# NAME              MODEL                      REPLICAS   READY   AGE
# research-agent    claude-sonnet-4-20250514   2          2       30s

kubectl get pods -l agentops.io/deployment=research-agent
# NAME                              READY   STATUS    RESTARTS   AGE
# research-agent-agent-7d9f-xk2p8   1/1     Running   0          30s
# research-agent-agent-7d9f-mn4q1   1/1     Running   0          30s
```

## Next steps

- [AgentDeployment reference](/docs/crds/agent-deployment) — full spec for the core resource
- [AgentService reference](/docs/crds/agent-service) — routing tasks to agents
- [AgentConfig reference](/docs/crds/agent-config) — reusable configuration
- [AgentPipeline reference](/docs/crds/agent-pipeline) — multi-agent DAGs
- [Semantic Health Checks](/docs/concepts/semantic-health-checks) — how readiness probes work
- [MCP Servers](/docs/concepts/mcp-servers) — connecting tools to agents
- [Scaling](/docs/concepts/scaling) — replicas and limits
