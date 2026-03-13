---
title: Roadmap
description: What agentops-operator can do today and what is coming: agent memory, human approvals, multi-model support, event triggers, autoscaling, and observability.
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

### Human-in-the-loop approvals
Insert an approval gate anywhere in a pipeline. Execution pauses, a notification is sent, and the workflow resumes only after a human approves via `kubectl`, a webhook, or a UI. Unapproved tasks can escalate or time out automatically.

### Event-driven triggers
Start agent workflows from external events: an inbound webhook, a cron schedule, or the output of another pipeline. One event can fan out to multiple pipelines in parallel.

### Supervisor agents that spawn workers dynamically
A supervisor agent breaks a task at runtime and delegates sub-tasks to specialist agents. The supervisor waits for results and assembles the final output, with no pre-defined static pipeline required.

### Conditional and looping pipelines
Steps can branch based on what a previous agent returned, or repeat until a quality threshold is met. Useful for research loops, validation workflows, and iterative refinement tasks.

### Multi-model support
Run agents on OpenAI, Google Gemini, or other providers alongside Anthropic. Route tasks to the cheapest or fastest model that meets your quality bar. Mix models within a single pipeline.

### Cost controls and token budgets
Track token usage per agent, per namespace, and per team. Set daily token limits and the operator automatically pauses an agent that exceeds its budget and resumes it the next day. No application code changes required.

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
