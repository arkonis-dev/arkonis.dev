---
title: Roadmap
description: ark-operator milestones — released, in progress, and planned.
nav_order: 10
---

# Roadmap

---

## Released

### v0.1 — Core Agent Management
Deploy AI agents as Kubernetes resources. Define a model, a system prompt, and a tool set. The operator manages replica pools, routes tasks via `ArkService`, and monitors agent health with semantic liveness probes.

### v0.2 — Pipelines & MCP
Declarative multi-step workflows with `ArkFlow`. Agents chain outputs to inputs via template expressions. Any MCP-compatible tool server connects without code changes.

### v0.3 — Memory & Structured Outputs
`ArkMemory` for persistent cross-task context (in-process, Redis, vector-store). Typed artifact passing between pipeline steps. Dead-letter queue with retry and backoff.

### v0.4 — Orchestration, Events & Cost Controls
Conditional steps (`if:`), loop steps, flow-level timeouts, supervisor/worker sub-tasks, inline HTTP tool definitions. `ArkEvent` triggers: cron, webhook, and pipeline-output with fan-out. Per-run and daily token budgets with automatic replica scaling.

### v0.5 — Local Development CLI
`ark run` and `ark validate` — execute and validate `ArkFlow` definitions locally without a cluster. Multi-provider support: Anthropic, OpenAI, and mock. Auto-detection of provider from model name. Pre-built binaries in every GitHub release.

---

## In Progress

### v0.6 — Human-in-the-Loop
`ArkCheckpoint` — pause a pipeline at any step and wait for a human decision. Approve or reject via `kubectl`, webhook, or a future UI. Configurable timeout with automatic escalation to a fallback step.

---

## Planned

### v0.7 — Multi-Model & Secrets
First-class support for OpenAI (`gpt-4o`, `o3`) and Google Gemini alongside Anthropic. `ArkService` model routing by cost, latency, or capability. `ArkSecret` for centralized API key management with rotation without pod restarts. Optional [LiteLLM](https://github.com/BerriAI/litellm) proxy for cross-provider failover and unified rate limiting.

### v0.8 — Observability
Distributed tracing via OpenTelemetry — spans for queue wait, LLM calls, and individual tool invocations. Prometheus metrics for throughput, token usage, error rates, and latency. `kubectl ark trace <task-id>` for per-task drill-down.

### v1.0 — Production Ready
KEDA autoscaling on queue depth. Helm chart for single-command install. Multi-tenancy hardening with per-namespace cost quotas. CNCF sandbox application.
