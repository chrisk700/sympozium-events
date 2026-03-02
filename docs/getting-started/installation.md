# Installation

## Install

```bash
helm install my-release \
  oci://ghcr.io/chrisk700/sympozium-events/sympozium-events \
  -f my-values.yaml
```

## Minimal values

```yaml
agentRun:
  agentId: my-agent          # agent ID inside the target SympoziumInstance
  model:
    provider: anthropic
    model: claude-sonnet-4-6
    authSecretRef: anthropic-key   # Secret containing ANTHROPIC_API_KEY

routes:
  - name: incident-response
    sources:
      - type: webhook
        name: pagerduty
        auth:
          secretName: pagerduty-secret   # see webhook auth below
    targets:
      - instanceRef: my-sympozium-instance
        taskTemplate: |
          Incident: {{ .Input.body.messages.0.incident.title }}
```

## Finding instanceRef and agentId

These two values are the most common first-run misconfiguration. They come from your Sympozium operator deployment:

- **`instanceRef`** — the name of a `SympoziumInstance` CR in `sympozium-system`
- **`agentId`** — the ID of an agent configured inside that instance

```bash
# List available SympoziumInstances
kubectl get sympoziuminstances -n sympozium-system

# Inspect an instance to find agentId values
kubectl get sympoziuminstance <name> -n sympozium-system -o yaml
```

## Webhook auth secret

Webhook sources require HMAC auth by default. Create a Secret before installing:

```bash
kubectl create secret generic pagerduty-secret \
  --from-literal=pagerduty=<your-hmac-token> \
  -n sympozium-events
```

The Secret key must match the source `name` in your values. See [Webhook](../sources/webhook.md) for details.

## Upgrade

```bash
helm upgrade my-release \
  oci://ghcr.io/chrisk700/sympozium-events/sympozium-events \
  --version <new-version> \
  -f my-values.yaml
```

## Uninstall

```bash
helm uninstall my-release
```

This removes the EventSource, Sensor, EventBus, RBAC, and NetworkPolicy resources. In-flight `AgentRun` CRs in `sympozium-system` are unaffected.
