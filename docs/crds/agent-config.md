---
title: ArkSettings
description: ArkSettings API reference — reusable prompt fragments and model settings shared across multiple ArkAgents in ark-operator.
parent: CRD Reference
nav_order: 3
---

# ArkSettings

**API:** `arkonis.dev/v1alpha1`
**Kind:** `ArkSettings`
No short name

Analogous to a Kubernetes `ConfigMap`. Stores reusable prompt fragments and model settings that can be referenced from multiple `ArkAgent` resources. Keeps configuration DRY — define a persona or output format once and share it across many agents.

## Example

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkSettings
metadata:
  name: analyst-base
  namespace: default
spec:
  temperature: "0.3"
  outputFormat: structured-json
  memoryBackend: in-context
  promptFragments:
    persona: "You are an expert analyst."
    outputRules: "Always cite sources. Flag uncertainty explicitly."
```

## Spec fields

| Field | Type | Required | Description |
|---|---|---|---|
| `temperature` | string | no | Sampling temperature passed to the Anthropic API. String-encoded float (e.g. `"0.7"`). |
| `outputFormat` | string | no | Requested output format. `structured-json` instructs the agent to return JSON. `markdown` and `plain` are also accepted. |
| `memoryBackend` | string | no | Memory strategy for the agent. `in-context` — conversation history kept in the context window. `vector-store` and `redis` planned for v1beta1. |
| `promptFragments` | map[string]string | no | Named prompt fragments merged into the effective system prompt. |

### `promptFragments`

A key-value map of prompt fragment names to text values. When an `ArkAgent` references this config via `spec.configRef`, the operator appends each fragment to the agent's system prompt in definition order.

Reserved keys with defined merge behavior:

| Key | Merge position |
|---|---|
| `persona` | Prepended to the system prompt. |
| `outputRules` | Appended after the system prompt. |

Other keys are appended in alphabetical order.

## Referencing ArkSettings from ArkAgent

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkAgent
metadata:
  name: analyst-agent
spec:
  model: claude-sonnet-4-20250514
  systemPrompt: |
    Analyze the provided dataset and identify key trends.
  configRef: analyst-base   # references the ArkSettings above
```

The operator resolves the `configRef` during reconciliation and builds the effective system prompt by merging `promptFragments`. The `temperature` and `outputFormat` values are injected as environment variables into agent pods alongside the standard operator-managed variables.

`ArkSettings` and the referencing `ArkAgent` must be in the same namespace.
