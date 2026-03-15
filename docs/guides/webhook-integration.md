---
title: Webhook Integration
description: Replace a hardcoded LLM call in your app with an ArkEvent webhook. Supports synchronous responses — POST and get the result back directly.
nav_order: 1
parent: Guides
---

# Webhook Integration

This guide shows how to replace a hardcoded LLM call in your app with an `ArkEvent` webhook. Your app POSTs to a URL and gets a structured response back directly — no polling, no code changes to your LLM logic, just YAML.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) with Kubernetes enabled
- `kubectl` (bundled with Docker Desktop)
- `helm`
- An Anthropic API key

**Enable Kubernetes in Docker Desktop:**
Settings → Kubernetes → Enable Kubernetes → Apply & Restart

Verify:
```bash
kubectl get nodes
# NAME             STATUS   ROLES           AGE
# docker-desktop   Ready    control-plane   1m
```

---

## 1. Install the operator

```bash
helm repo add arkonis https://charts.arkonis.dev
helm repo update

helm install ark-operator arkonis/ark-operator \
  --namespace ark-system \
  --create-namespace \
  --set taskQueueURL=redis.ark-system.svc.cluster.local:6379 \
  --set triggerWebhook.url=http://ark-operator.ark-system.svc.cluster.local:8092 \
  --set apiKeys.anthropicApiKey=sk-ant-...
```

---

## 2. Deploy Redis

```bash
kubectl apply -f https://raw.githubusercontent.com/arkonis-dev/ark-operator/main/config/prereqs/redis.yaml
```

Wait for it to be ready:
```bash
kubectl rollout status deployment/redis -n ark-system
```

---

## 3. Store your prompt in a ConfigMap

For long prompts, keep them out of the YAML spec and in a ConfigMap instead. The operator reads it at reconcile time and restarts agent pods automatically when it changes.

Create a file `prompt.txt` with your full system prompt:

```
You are a data extraction assistant. Your job is to read the provided
text carefully and extract structured information from it.

Return only a valid JSON object — no explanation, no markdown, no code
fences. The JSON must match the schema the caller expects exactly.

If a field cannot be determined from the text, set it to null.
Never guess or invent values.
```

Create the ConfigMap:

```bash
kubectl create configmap extractor-prompt \
  --from-file=system.txt=./prompt.txt \
  --namespace ark-system
```

---

## 4. Define your agent and flow

Create a file `my-flow.yaml`. The agent references the prompt from the ConfigMap via `systemPromptRef`.

```yaml
apiVersion: arkonis.dev/v1alpha1
kind: ArkAgent
metadata:
  name: extractor-agent
  namespace: ark-system
spec:
  model: claude-sonnet-4-20250514
  systemPromptRef:
    configMapKeyRef:
      name: extractor-prompt
      key: system.txt
  limits:
    maxTokensPerCall: 4000
    timeoutSeconds: 30
---
apiVersion: arkonis.dev/v1alpha1
kind: ArkFlow
metadata:
  name: extract-flow
  namespace: ark-system
spec:
  steps:
    - name: extract
      arkAgent: extractor-agent
      inputs:
        prompt: "{{ .pipeline.input.text }}"
  output: "{{ .steps.extract.output }}"
---
apiVersion: arkonis.dev/v1alpha1
kind: ArkEvent
metadata:
  name: extract-trigger
  namespace: ark-system
spec:
  type: webhook
  flows:
    - name: extract-flow
```

Apply it:
```bash
kubectl apply -f my-flow.yaml
```

Check the webhook URL was assigned:
```bash
kubectl get arkevent extract-trigger -n ark-system
```

---

## 5. Expose the webhook to localhost

```bash
kubectl port-forward svc/ark-operator 8092:8092 -n ark-system
```

Leave this running in a terminal. Your app can now reach the webhook at `http://localhost:8092`.

---

## 6. Call it from your app

Replace your hardcoded LLM call with a POST to the webhook. Add `?mode=sync` so the request waits and returns the result directly — no polling needed.

First get your webhook token:
```bash
kubectl get secret extract-trigger-webhook-token -n ark-system -o jsonpath='{.data.token}' | base64 -d
```

Then call it from your app:
```javascript
const token = process.env.ARK_WEBHOOK_TOKEN

const response = await fetch(
  'http://localhost:8092/triggers/ark-system/extract-trigger/fire?mode=sync&timeout=30s',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({ text: "Your input text here..." })
  }
)

const result = await response.json()
// result.status  → "succeeded" or "failed"
// result.output  → the agent's response string
// result.durationMs → how long it took
console.log(result.output)
```

The flow runs synchronously — the HTTP connection stays open until Claude responds, then the result is returned in the response body. Default timeout is 60s; override with `?timeout=30s`.

---

## 7. Iterate on the prompt

Edit `prompt.txt` and reapply the ConfigMap — no YAML changes needed:

```bash
kubectl create configmap extractor-prompt \
  --from-file=system.txt=./prompt.txt \
  --namespace ark-system \
  --dry-run=client -o yaml | kubectl apply -f -
```

The operator detects the ConfigMap change, restarts the agent pods, and the next webhook call uses the updated prompt. No app restart required.

---

## Teardown

```bash
helm uninstall ark-operator -n ark-system
kubectl delete namespace ark-system
```
