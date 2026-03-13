---
title: Getting Started
description: Install agentops-operator on Kubernetes in minutes. Deploy AI agents with a single kubectl apply command.
nav_order: 2
---

# Getting Started

## Local development

The fastest way to try agentops-operator is with the `make dev` command. It creates a [kind](https://kind.sigs.k8s.io/) cluster and sets everything up in one step with no manual Redis config required.

**Prerequisites:** Docker, kind, kubectl, Go 1.25+

```bash
git clone https://github.com/agentops-io/agentops-operator.git
cd agentops-operator
make dev ANTHROPIC_API_KEY=sk-ant-...
```

This will:
1. Create a kind cluster
2. Build and load both Docker images into the cluster
3. Install the CRDs
4. Deploy Redis and the operator inside the cluster
5. Create the API key secret

When it finishes, deploy your first agent:

```bash
kubectl apply -f config/samples/agentops_v1alpha1_agentdeployment.yaml
kubectl get agdep -w
# NAME             MODEL                      REPLICAS   READY   AGE
# research-agent   claude-sonnet-4-20250514   2          2       30s
```

Tear down when done:

```bash
make dev-down
```

---

## Production install

### Prerequisites

- Kubernetes 1.31+
- `kubectl` configured against your cluster
- An API key for your chosen LLM provider (Anthropic by default)

### 1. Install the operator

```bash
kubectl apply -f https://github.com/agentops-io/agentops-operator/releases/latest/download/install.yaml
```

This installs the CRDs, RBAC, and the operator deployment in the `agentops-system` namespace.

### 2. Deploy Redis

agentops-operator uses Redis Streams as the task queue between the operator and agent pods.

```bash
kubectl apply -f https://raw.githubusercontent.com/agentops-io/agentops-operator/main/config/prereqs/redis.yaml
```

### 3. Create the API key secret

Create one secret per namespace where agents will run. The operator injects the entire secret as environment variables into every agent pod. Add whichever keys your chosen provider requires.

**Anthropic (default):**
```bash
kubectl create secret generic agentops-operator-api-keys \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-... \
  --from-literal=TASK_QUEUE_URL=redis.agent-infra.svc.cluster.local:6379
```

`TASK_QUEUE_URL` is always required. To use a different provider, also add `AGENT_PROVIDER=<name>` to the secret.

### 4. Deploy your first agent

```bash
kubectl apply -f https://raw.githubusercontent.com/agentops-io/agentops-operator/main/config/samples/agentops_v1alpha1_agentdeployment.yaml
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
