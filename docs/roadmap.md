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
`ArkMemory` for persistent cross-task context (in-process, Redis, vector-store). Typed artifact passing between pipeline steps with JSON schema validation. Dead-letter queue with retry and backoff.

### v0.4 — Orchestration, Events & Cost Controls
Conditional steps (`if:`), loop steps, flow-level timeouts, supervisor/worker sub-tasks, inline HTTP tool definitions. `ArkEvent` triggers: cron, webhook, and pipeline-output with fan-out. Per-run and daily token budgets with automatic replica scaling.

### v0.5 — Local Development CLI & Helm
`ark run` and `ark validate` — execute and validate `ArkFlow` definitions locally without a cluster. Multi-provider support: Anthropic, OpenAI, and mock. Auto-detection of provider from model name (`claude-*` → Anthropic, `gpt-*/o*` → OpenAI). Mixed-model flows work today — each step references its own `ArkAgent` with any model. `--output json` for CI pipelines. Pre-built binaries in every GitHub release.

Helm chart for single-command cluster install: `helm install ark-operator arkonis/ark-operator`. Supports configurable namespaces, resource limits, image pull secrets, and API key injection from existing secrets.

---

## In Progress

### v0.6 — Human-in-the-Loop
`ArkCheckpoint` — pause a pipeline at any step and wait for a human decision. Approve or reject via `kubectl`, webhook, or a future UI. Configurable timeout with automatic escalation to a fallback step.

---

## Planned

### v0.7 — Observability ⚠️
The biggest gap for production adoption. Without this, platform teams cannot safely operate the system at scale.

- Prometheus metrics: throughput, queue depth, token usage, latency histograms
- Distributed tracing via OpenTelemetry — spans for queue wait, LLM calls, and tool invocations
- Per-agent and per-namespace token spend dashboard
- Structured audit log — every agent action emitted as a Kubernetes Event
- `kubectl ark trace <task-id>` for per-task drill-down

### v0.8 — Multi-Model & Advanced Cost Controls
First-class Google Gemini support alongside Anthropic and OpenAI. `ArkService` model routing by cost, latency, or capability. Optional [LiteLLM](https://github.com/BerriAI/litellm) proxy for cross-provider failover and unified rate limiting.

Cost controls beyond per-run budgets: per-namespace quotas, pre-limit alerts, cost attribution by team label, and monthly spend rollups in `.status`.

### v0.9 — Developer Experience & Storage
OperatorHub listing for platform team discoverability. KEDA autoscaling on queue depth.

File artifact support — flow steps can produce and consume files, not just strings. Backends: local disk (`ark run`), S3, and GCS. A step can declare `outputType: file` and downstream steps receive a signed URL or local path. Enables agent-generated reports, images, and datasets to flow through pipelines without manual piping.

### v1.0 — Production Ready
Multi-tenancy hardening with per-namespace RBAC. CNCF sandbox application.
