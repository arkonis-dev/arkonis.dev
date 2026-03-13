---
title: Overview
description: agentops-operator is a Kubernetes operator that manages AI agents as first-class cluster resources, covering scaling, health checks, and lifecycle management for LLM-based agents.
nav_order: 1
---

# agentops-operator

agentops-operator is a Kubernetes operator that introduces AI agents as first-class cluster resources. Instead of managing containers, you manage LLM-based agents: each defined by a model, a system prompt, and a set of MCP servers that provide tools. The operator handles scheduling, scaling, health checking, and lifecycle management using the same reconciliation model as any other Kubernetes controller.

Everything integrates with the Kubernetes ecosystem you already use. GitOps workflows, RBAC, namespaces, and `kubectl` all work without modification. Promoting a new system prompt is a pull request; rolling it back is a revert in ArgoCD or Flux, the same workflow you already use for everything else.

## CRDs

| Resource | Analogy | Description |
|---|---|---|
| `AgentDeployment` | `Deployment` | Manages a pool of agent instances running the same model, prompt, and MCP tool servers. Handles scaling and semantic health checks. |
| `AgentService` | `Service` | Routes incoming tasks to available agent instances using configurable load balancing strategies (round-robin, least-busy, random). |
| `AgentConfig` | `ConfigMap` | Reusable prompt fragments and model settings. Referenced from `AgentDeployment` via `spec.configRef` to keep configuration DRY. |
| `AgentPipeline` | *(novel)* | Defines a DAG of agents where outputs feed into inputs. The primitive for declarative multi-agent workflows. |

Short names: `agdep`, `agsvc`, `agcfg`, `agpipe`

## Next steps

- [Why agentops-operator](/docs/why) — who this is for and what problems it solves
- [Getting Started](/docs/getting-started) — install and deploy your first agent
- [CRD Reference](/docs/crds/) — full field reference for all resource types
- [Concepts](/docs/concepts/) — semantic health checks, MCP servers, scaling
- [Roadmap](/docs/roadmap) — what is coming next
