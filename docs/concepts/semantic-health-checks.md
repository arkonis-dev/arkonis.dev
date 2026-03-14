---
title: Semantic Health Checks
description: How ark-operator validates AI agent health using LLM-based readiness probes that check output quality, not just liveness.
parent: Concepts
nav_order: 1
---

# Semantic Health Checks

## The problem

Standard Kubernetes liveness and readiness probes check whether a process is alive and whether a port is accepting connections. For stateless web services, this is sufficient. For LLM agents, it is not.

An agent pod can be running, healthy from Kubernetes' perspective, and consistently producing wrong, hallucinated, or off-task output. A broken API key, a misconfigured system prompt, or model degradation can cause this. None of these are detectable by an HTTP probe that checks `/healthz`.

## The solution

Each agent pod exposes two health endpoints on port `8080`:

| Endpoint | Type | Behavior |
|---|---|---|
| `GET /healthz` | Liveness | Always returns `200 OK` if the process is running. Used by Kubernetes to decide whether to restart the container. |
| `GET /readyz` | Semantic readiness | Calls the configured LLM provider with a validation prompt and checks the response. Returns `200 OK` if the output passes validation; returns `503 Service Unavailable` if it fails. |

When `/readyz` returns `503`, Kubernetes marks the pod `NotReady`. The `ArkService` stops routing tasks to that pod. The operator logs the failure and the pod continues trying — if the underlying issue resolves (e.g., transient API error), the pod recovers automatically.

## How the readiness probe works

1. The agent runtime receives an HTTP `GET /readyz` from the kubelet.
2. The runtime sends a validation prompt to the configured LLM provider using the agent's model.
3. The response is checked against expected output characteristics (correct format, non-empty, no error indicators).
4. If the check passes, the endpoint returns `200`. If it fails or the API call errors, it returns `503`.

The probe runs on the kubelet's schedule, configured via standard Kubernetes `readinessProbe` fields that the operator injects automatically.

## Spec field

```yaml
spec:
  livenessProbe:
    type: semantic        # "semantic" | "ping"
    intervalSeconds: 60   # how often to run the semantic check
```

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | string | `ping` | `ping` — HTTP reachability only. `semantic` — enables `/readyz` API validation. |
| `intervalSeconds` | int | `60` | Interval between semantic checks. |
| `validatorPrompt` | string | *(internal default)* | Custom prompt sent to the LLM during `/readyz` validation. Falls back to a built-in default if not set. |
