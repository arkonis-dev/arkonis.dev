---
title: ark CLI
description: Run ArkFlow definitions locally without a Kubernetes cluster. Full provider auto-detection, step streaming, and DAG validation.
nav_order: 3
---

# ark CLI

The `ark` command-line tool lets you execute `ArkFlow` definitions locally — no Kubernetes cluster, no Redis, no operator required. It shares the same YAML format as the operator, so flows developed locally deploy to Kubernetes unchanged.

---

## Install

### go install (recommended)

Requires Go 1.22 or later.

```bash
go install github.com/arkonis-dev/ark-operator/cmd/ark@latest
```

Verify:

```bash
ark --version
# ark v0.5.3 (go1.22)
```

### Binary download

Pre-built binaries for Linux, macOS, and Windows are attached to each GitHub release:

```
https://github.com/arkonis-dev/ark-operator/releases/latest
```

Download the binary for your platform, make it executable, and place it on your `$PATH`:

```bash
# macOS (Apple Silicon)
curl -L https://github.com/arkonis-dev/ark-operator/releases/latest/download/ark-darwin-arm64 \
  -o ark && chmod +x ark && sudo mv ark /usr/local/bin/ark

# macOS (Intel)
curl -L https://github.com/arkonis-dev/ark-operator/releases/latest/download/ark-darwin-amd64 \
  -o ark && chmod +x ark && sudo mv ark /usr/local/bin/ark

# Linux (amd64)
curl -L https://github.com/arkonis-dev/ark-operator/releases/latest/download/ark-linux-amd64 \
  -o ark && chmod +x ark && sudo mv ark /usr/local/bin/ark

# Linux (arm64)
curl -L https://github.com/arkonis-dev/ark-operator/releases/latest/download/ark-linux-arm64 \
  -o ark && chmod +x ark && sudo mv ark /usr/local/bin/ark
```

**Windows** — download `ark-windows-amd64.exe` from the [releases page](https://github.com/arkonis-dev/ark-operator/releases/latest), rename it to `ark.exe`, and add it to your `PATH`.

---

## Commands

### `ark run`

Execute a flow definition locally.

```
ark run <flow.yaml> [flags]
```

**Flags**

| Flag | Default | Description |
|---|---|---|
| `--provider` | `auto` | Model provider: `auto`, `anthropic`, `openai`, `mock` |
| `--watch` | `false` | Stream step status and output as each step completes |
| `--max-tokens` | (from spec) | Override the flow-level token budget |
| `--timeout` | (from spec) | Override `spec.timeoutSeconds` |
| `--input` | — | Inject a top-level input value, e.g. `--input topic="LLMs"` (repeatable) |
| `--output` | — | Write the flow's final output to a file instead of stdout |
| `--log-level` | `info` | Log verbosity: `debug`, `info`, `warn`, `error` |

**Examples**

```bash
# Run with mock provider (no API key needed)
ark run quickstart.yaml --provider mock --watch

# Run against Anthropic with a specific input
ark run research-flow.yaml \
  --provider anthropic \
  --input topic="transformer architecture" \
  --watch

# Write output to a file, no streaming
ark run briefing-flow.yaml \
  --provider auto \
  --output ./output.json

# Override token budget for a cost-sensitive run
ark run large-flow.yaml --provider auto --max-tokens 4000
```

---

### `ark init`

Scaffold a new project directory with a ready-to-run flow.

```
ark init <project-name>
```

Creates a new directory containing:

| File | Description |
|---|---|
| `quickstart.yaml` | `ArkAgent` + `ArkFlow` definition, runnable immediately with `--provider mock` |
| `.env.example` | Template for API keys — copy to `.env` and fill in |
| `docker-compose.yml` | Local Redis for the task queue (needed when deploying to a cluster) |

**Example**

```bash
ark init my-agent
cd my-agent
ark run quickstart.yaml --provider mock --watch
```

---

### `ark validate`

Check a flow definition for errors without making any model calls.

```
ark validate <flow.yaml> [flags]
```

Validation checks:

- DAG structure: detects cycles using DFS
- Agent references: verifies all `arkAgent` names resolve (when `--check-agents` is set and a kubeconfig is available)
- `dependsOn` chains: confirms referenced step names exist
- `if:` expressions: parses conditional syntax
- `loop.condition` expressions: validates loop step configuration
- JSON schema: validates all `spec` fields against the CRD schema

**Flags**

| Flag | Default | Description |
|---|---|---|
| `--check-agents` | `false` | Resolve `arkAgent` references against a live cluster |
| `--kubeconfig` | `~/.kube/config` | Path to kubeconfig (only used with `--check-agents`) |

**Example output**

```
$ ark validate quickstart.yaml
  DAG          ok  (2 steps, 0 cycles)
  Agents       ok  (research-agent, summarizer)
  Schema       ok
Validation passed.

$ ark validate broken-flow.yaml
  DAG          error  cycle detected: summarize → research → summarize
Validation failed.
```

---

## Provider auto-detection

When `--provider auto` is set (the default), the CLI infers the provider from the model name in the flow spec:

| Model name prefix | Provider | Required env var |
|---|---|---|
| `claude-*` | Anthropic | `ANTHROPIC_API_KEY` |
| `gpt-*`, `o1-*`, `o3-*` | OpenAI | `OPENAI_API_KEY` |
| `mock` | Built-in mock | None |

If a model name is ambiguous or unrecognized, `ark` exits with an error and asks you to pass `--provider` explicitly.

```bash
# These are equivalent
ark run flow.yaml --provider auto
ark run flow.yaml
```

---

## The mock provider

Pass `--provider mock` to run flows without any API key. The mock provider returns short, deterministic placeholder responses for every step. It is intended for:

- Testing DAG wiring and `dependsOn` chains
- CI pipelines that validate flow structure
- Local demos that need no credentials

Mock responses take the form:

```
[mock response for step "research": topic=LLMs]
```

---

## `--watch` output format

When `--watch` is passed, each step prints two lines as it completes:

```
  <step-name>        [running]
  <step-name>        [done]    <tokens> tokens  <seconds>s
    └─ <first 80 chars of step output>
```

If a step fails, `[done]` is replaced by `[failed]` and the error message is shown instead of the output snippet. The final line is the flow summary:

```
Flow Succeeded in 3.1s — total: 2154 tokens
```

or on failure:

```
Flow Failed in 1.2s — step "research" error: context deadline exceeded
```

---

## Example: quickstart.yaml

A minimal two-step flow you can run immediately with `--provider mock`:

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkFlow
metadata:
  name: quickstart
spec:
  steps:
    - name: research
      arkAgent: research-agent
      inputs:
        topic: "{{ .pipeline.input.topic }}"
    - name: summarize
      arkAgent: summarizer-agent
      dependsOn: [research]
      inputs:
        content: "{{ .steps.research.output }}"
  output: "{{ .steps.summarize.output }}"
```

Run it:

```bash
ark run quickstart.yaml --provider mock --watch --input topic="transformers"
```

Expected output:

```
  research         [running]
  research         [done]    1842 tokens  2.3s
    └─ Large language models are transformer-based neural networks...
  summarize        [running]
  summarize        [done]    312 tokens  0.8s
    └─ • LLMs use transformer architecture...

Flow Succeeded in 3.1s — total: 2154 tokens
```

---

## Environment variables

| Variable | Description |
|---|---|
| `ANTHROPIC_API_KEY` | API key for the Anthropic provider |
| `OPENAI_API_KEY` | API key for the OpenAI provider |
| `ARK_LOG_LEVEL` | Default log level (overridden by `--log-level`) |
| `ARK_PROVIDER` | Default provider (overridden by `--provider`) |

---

## Differences from the Kubernetes operator

The `ark` CLI is a local execution engine, not a full operator. The following operator features are not available locally:

| Feature | Operator | ark CLI |
|---|---|---|
| Replica management / pod scheduling | Yes | No |
| Redis task queue | Yes | No (in-process queue) |
| Semantic liveness probes | Yes | No |
| ArkService routing | Yes | No |
| GitOps / kubectl apply | Yes | No |
| ArkEvent (cron / webhook triggers) | Yes | No |
| ArkMemory (Redis / vector-store) | Planned | No |

For production workloads, deploy the operator to a Kubernetes cluster. Use the CLI for development, testing, and one-off runs.

---

## Next steps

- [Getting Started](../getting-started) — deploy the operator to a local kind cluster
- [ArkFlow reference](../concepts/arkflow) — full flow spec documentation
- [Roadmap](../roadmap) — what's coming next
