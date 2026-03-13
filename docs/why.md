---
title: Why arkonis-operator
description: Why engineering teams running AI agents in production need arkonis-operator: cost controls, semantic health checks, GitOps, and Kubernetes-native agent management.
nav_order: 3
---

# Why arkonis-operator

If you are a solo developer experimenting with LLMs, you do not need this. Claude Code, the Anthropic API, or a simple Python script will serve you better.

arkonis-operator is for **engineering teams running agents in production at scale**, where the problems are operational, not algorithmic.

---

## The problems it solves

### "We have agents across multiple teams. How do we manage them?"

The same way you manage any other production workload: Kubernetes. Your platform team already knows the tooling, the access controls, and the deployment patterns. arkonis-operator plugs agents into that existing system instead of introducing a parallel management layer.

### "An agent is burning through our API budget"

Hard limits enforced at the infrastructure level, not in application code that a developer can accidentally remove or misconfigure:

```yaml
spec:
  limits:
    maxTokensPerCall: 8000
    maxConcurrentTasks: 5
    timeoutSeconds: 120
```

These are applied to every pod the operator creates, regardless of what the agent code does.

### "How do we know if our agent is actually working?"

Standard Kubernetes liveness probes check whether a process is running. They cannot tell you whether an LLM is producing useful output.

arkonis-operator introduces **semantic health checks**: a secondary LLM call that validates the agent's actual output quality. Pods that fail the semantic probe are removed from routing until they recover, exactly like a failing HTTP health check.

```yaml
spec:
  livenessProbe:
    type: semantic
    intervalSeconds: 60
    validatorPrompt: "Reply with exactly one word: HEALTHY"
```

### "We need to roll back our agent after a bad deploy"

A system prompt change is a pull request. A rollback is a sync revert in ArgoCD or Flux. The same GitOps workflow your team already uses for every other service works identically for agents.

### "Compliance requires an audit trail of every agent configuration change"

Every agent definition goes through `kubectl`, RBAC, and git. Kubernetes already provides the audit trail: who changed what, when, and through which approval process.

### "We need to chain agents together without writing orchestration code"

ArkonisPipeline defines multi-agent DAGs declaratively. Outputs from one agent are templated directly into the next step's input:

```yaml
spec:
  steps:
    - name: research
      arkonisDeployment: research-agent
      inputs:
        prompt: "Research this topic: {{ .pipeline.input.topic }}"
    - name: summarize
      arkonisDeployment: summarizer-agent
      dependsOn: [research]
      inputs:
        prompt: "Summarize: {{ .steps.research.output }}"
```

No orchestration code. No custom queue logic. Just YAML.

---

## Who this is for

| Role | Why it helps |
|---|---|
| Platform engineer | Standard Kubernetes primitives for AI agents. No new management layer to operate. |
| Team lead | Cost controls and limits enforced at infrastructure level, not per-developer discipline |
| SRE / on-call | Semantic health checks and standard alerting work without custom tooling |
| Security / compliance | RBAC, audit logs, and namespace isolation apply to agents automatically |
