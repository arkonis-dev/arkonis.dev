---
title: FAQ
description: Frequently asked questions about ark-operator — how it works, what it costs, and how it compares to other tools.
nav_order: 11
---

# FAQ

---

## What is ARKONIS?

ARKONIS (Agentic Reconciler for Kubernetes, Operator-Native Inference System) is a Kubernetes operator that lets you deploy, scale, and manage AI agents as first-class Kubernetes resources. Instead of managing containers, you manage agents — each defined by a model, a system prompt, and a set of tools.

---

## Do I need a Kubernetes cluster to try it?

No. The `ark` CLI lets you run `ArkFlow` pipelines locally on your laptop with no cluster, no Redis, and no operator required. The same YAML files work unchanged when you deploy to Kubernetes.

```bash
go install github.com/arkonis-dev/ark-operator/cmd/ark@latest
ark run quickstart.yaml --provider mock --watch
```

---

## Which LLM providers are supported?

Anthropic (`claude-*` models) and OpenAI (`gpt-*`, `o1-*`, `o3-*` models) are supported today. Google Gemini is planned for v0.8. The provider is auto-detected from the model name, or you can set it explicitly with `--provider`.

---

## How is this different from LangChain / LlamaIndex / CrewAI?

Those are Python frameworks for building agent logic inside your application code. ARKONIS is infrastructure — it runs outside your code and manages agents the same way Kubernetes manages containers. You don't import it; you `kubectl apply` it.

---

## How is this different from just running agents in a Deployment?

A plain Deployment knows nothing about agents. It can't detect when an agent is producing bad output, enforce token budgets, chain agents into DAG pipelines, or route tasks by load. ARKONIS introduces purpose-built primitives (`ArkAgent`, `ArkFlow`, `ArkService`) that encode those concepts at the infrastructure level.

---

## Does it work with any Kubernetes distribution?

Yes. It uses standard controller-runtime and has no cloud-provider dependencies. It works on EKS, GKE, AKS, kind, k3s, and any other conformant distribution.

---

## What task queue does it use?

Redis Streams with consumer groups. Redis is the only external dependency. A future release will add a Kubernetes-native queue option for teams that prefer not to run Redis.

---

## Is there a Helm chart?

Yes. Add the Helm repo and install with one command:

```bash
helm repo add arkonis https://charts.arkonis.dev
helm install ark-operator arkonis/ark-operator \
  --namespace ark-system --create-namespace \
  --set taskQueueURL=redis.agent-infra.svc.cluster.local:6379 \
  --set apiKeys.anthropicApiKey=sk-ant-...
```

See the [Getting Started](../getting-started) guide for the full install options.

---

## Is it production-ready?

The core operator (v0.1–v0.4) is stable and battle-tested. Observability (Prometheus metrics, OpenTelemetry tracing) is planned for v0.7 and is the main gap before we recommend it for production at scale. See the [roadmap](./roadmap) for the full picture.

---

## How do I contribute?

Open an issue or pull request on [GitHub](https://github.com/arkonis-dev/ark-operator). The project is Apache 2.0 licensed.
