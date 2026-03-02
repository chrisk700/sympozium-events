# Webhook

Receives HTTP POST requests from external systems. Compatible with any sender that supports HMAC webhook secrets (GitHub, PagerDuty, custom).

## Configuration

```yaml
routes:
  - name: my-route
    sources:
      - type: webhook
        name: pagerduty        # becomes the endpoint path: /pagerduty
        port: "12000"          # optional, default "12000"
        endpoint: /pagerduty   # optional, default /<name>
        method: POST           # optional, default POST
        auth:
          secretName: pagerduty-secret   # required unless allowInsecure: true
```

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Source name. Used as the endpoint path and as the Secret key. |
| `port` | no | `"12000"` | Port the EventSource listens on. |
| `endpoint` | no | `/<name>` | HTTP path. |
| `method` | no | `POST` | HTTP method. |
| `auth.secretName` | yes* | — | Name of a Secret with a key matching `name` holding the HMAC token. |
| `auth.allowInsecure` | no | `false` | Skip HMAC validation for this source. See [Insecure opt-out](#insecure-opt-out). |

*Required unless `auth.allowInsecure: true`. Missing `auth` on a webhook source is a template error — not a silent insecure default.

## Auth secret

Argo Events validates `X-Hub-Signature-256: sha256=<hmac>` on every incoming request. Create the Secret before installing the chart:

```bash
kubectl create secret generic pagerduty-secret \
  --from-literal=pagerduty=<hmac-token> \
  -n sympozium-events
```

The Secret key must match the source `name`. Multiple sources can share one Secret:

```bash
kubectl create secret generic webhook-secrets \
  --from-literal=pagerduty=<token-1> \
  --from-literal=github=<token-2> \
  -n sympozium-events
```

```yaml
sources:
  - type: webhook
    name: pagerduty
    auth:
      secretName: webhook-secrets
  - type: webhook
    name: github
    auth:
      secretName: webhook-secrets
```

## Token rotation

Update the Secret value only — no chart redeploy needed. Argo Events reloads secrets per request.

```bash
kubectl create secret generic pagerduty-secret \
  --from-literal=pagerduty=<new-token> \
  -n sympozium-events \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Insecure opt-out

For internal-only or trusted-network sources, HMAC can be explicitly disabled:

```yaml
sources:
  - type: webhook
    name: internal-trigger
    auth:
      allowInsecure: true
```

!!! warning
    Sources with `allowInsecure: true` accept any request without verification. NOTES.txt warns on install and upgrade when insecure sources are present.

## taskTemplate

The incoming payload is available at `.Input.body`. The default template for webhook sources:

```
Event received: {{ .Input.body | toJson }}
```

Named presets for common senders (`pagerduty`, `github`) are available in `defaults.taskTemplates`. Override per target:

```yaml
targets:
  - instanceRef: my-instance
    taskTemplate: |
      Incident: {{ .Input.body.messages.0.incident.title }}
      Urgency: {{ .Input.body.messages.0.incident.urgency }}
```

## Filters

Filters are applied at the Sensor trigger level. Example — only fire on PagerDuty `incident.trigger` events:

```yaml
targets:
  - instanceRef: my-instance
    filters:
      data:
        - path: "body.messages.0.event"
          type: string
          value: "incident.trigger"
```
