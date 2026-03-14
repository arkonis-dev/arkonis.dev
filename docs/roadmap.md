---
title: Roadmap
description: What ark-operator can do today and what is coming: agent memory, human approvals, multi-model support, event triggers, autoscaling, and observability.
nav_order: 10
---

# Roadmap

This page describes capabilities in terms of what you can do, not internal version numbers or API names.

---

## Available today

### Deploy and scale agents
Define an AI agent with a model, a system prompt, and a set of tools. The operator runs it as a pool of instances and scales replicas up or down the same way you scale any other service.

### Route tasks intelligently
A service layer distributes incoming tasks across available agent instances using round-robin, least-busy, or random strategies. Add more replicas and routing adjusts automatically.

### Chain agents into pipelines
Define multi-step workflows where one agent's output becomes another's input. No orchestration code, just a declarative definition of which agent runs when and what data it receives.

### Connect agents to tools via MCP
Agents connect to any [MCP-compatible](https://modelcontextprotocol.io) tool server: web search, code execution, databases, APIs. Tools are declared in the agent spec and discovered automatically at startup.

### Semantic health checks
Standard health checks tell you if a process is running. Semantic health checks tell you if the agent is actually producing useful output. Agents that fail the quality check are removed from task routing until they recover.

### Manage agents like any other service
System prompt changes go through pull requests. Rollbacks are a sync revert in ArgoCD or Flux. RBAC, audit logs, namespaces, and `kubectl` all work without modification.

### Conditional and looping pipelines
Steps can branch based on what a previous agent returned, or repeat until a quality threshold is met. Declare an `if:` expression to skip a step when a condition is false, or a `loop:` block to repeat until a quality condition is satisfied or a maximum iteration count is reached.

### Supervisor agents that spawn workers dynamically
Any running agent can enqueue sub-tasks at runtime via the built-in `submit_subtask` tool. The agent submits a prompt, gets back a task ID, and the new task is processed asynchronously by the agent pool — no pre-defined static pipeline required.

### Inline tool definitions
For simple integrations that do not warrant a full MCP server, declare tools directly in `spec.tools` as HTTP endpoints. The agent calls your URL when the LLM invokes the tool and returns the response to the model. No extra infrastructure required.

### Pipeline-level timeouts
Set `spec.timeoutSeconds` on any `ArkFlow` to fail the entire pipeline if it hasn't completed within the configured wall-clock limit.

### Token usage visibility and budget enforcement
Every pipeline step reports how many input and output tokens it consumed. The pipeline status rolls this up into a total. Set `spec.maxTokens` on any `ArkFlow` and the pipeline is automatically failed with reason `BudgetExceeded` if the limit is reached — before the next step runs. Token counts appear as a column in `kubectl get arkflow`.

Set `spec.limits.maxDailyTokens` on any `ArkAgent` to cap rolling 24-hour spend. When the limit is hit, the operator scales the agent pool to zero replicas and sets a `BudgetExceeded` condition. The deployment resumes automatically once the 24-hour window rotates and accumulated usage drops below the limit. Current 24-hour token usage appears as a column in `kubectl get arkagent`.

### Event-driven triggers
Start agent workflows from external events. Define an `ArkEvent` with a source type of `cron`, `webhook`, or `pipeline-output`. One trigger can fan out to multiple pipelines in parallel. Webhook triggers are authenticated via a per-trigger token stored in a Kubernetes Secret.

---

## In progress

### Agent memory
Agents that remember context across many tasks over time, not just within a single conversation. Choose the memory backend that fits your stack: in-process context, Redis, or a vector store like Qdrant, Pinecone, or Weaviate.

### Structured outputs between pipeline steps
Pipeline steps can declare the shape of their output. The agent is instructed to respond in that format, the output is validated, and downstream steps can reference individual fields directly instead of passing raw text.

### Reliable task processing with retry
Failed tasks are automatically retried with backoff before being moved to a dead-letter queue. Operators can inspect and replay dead-lettered tasks without any custom tooling.

---

## Coming soon

### Human-in-the-loop approvals *(in progress)*
Insert an approval gate anywhere in a pipeline. Execution pauses, a notification is sent, and the workflow resumes only after a human approves via `kubectl`, a webhook, or a UI. Unapproved tasks can escalate or time out automatically.

### Multi-model support
Run agents on OpenAI, Google Gemini, or other providers alongside Anthropic. Route tasks to the cheapest or fastest model that meets your quality bar. Mix models within a single pipeline.

### LiteLLM integration
Optional integration with [LiteLLM](https://github.com/BerriAI/litellm) for teams that want cross-provider failover, unified rate limiting, or namespace-level token quotas. The operator injects the proxy endpoint automatically. You get the benefits of a smart LLM proxy without ark-operator needing to sit in the call path.

### Cost controls and token budgets *(partial — per-run budget enforcement available today)*
Per-run token budgets are available now via `spec.maxTokens`. Daily limits per agent, per namespace, and per team are planned for a future release.

---

## Planned

### Autoscaling on queue depth
Agent replicas scale based on how many tasks are waiting, not CPU or memory. An empty queue scales to zero; a deep queue adds replicas immediately. Implemented via [KEDA](https://keda.sh) so it integrates with existing autoscaling infrastructure.

### Distributed tracing
Every task produces a trace: time spent in queue, LLM call latency, individual tool call durations, and total cost. Exportable to any OpenTelemetry-compatible backend.

### Usage dashboards
Per-agent and per-namespace views of task throughput, token consumption, error rates, and latency. No custom instrumentation required.

### Helm chart
Single-command install with sensible defaults. Configurable for production deployments without editing raw manifests.
