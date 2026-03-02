# First route

This walkthrough sets up a PagerDuty webhook that triggers an `AgentRun` on every incident.

## 1. Find your instanceRef and agentId

```bash
kubectl get sympoziuminstances -n sympozium-system
# NAME                    AGE
# incident-responder      3d

kubectl get sympoziuminstance incident-responder -n sympozium-system -o yaml | grep agentId
# agentId: on-call-agent
```

## 2. Create the webhook auth secret

```bash
kubectl create secret generic pagerduty-secret \
  --from-literal=pagerduty=<your-pagerduty-hmac-token> \
  -n sympozium-events
```

## 3. Write your values file

```yaml title="my-values.yaml"
agentRun:
  agentId: on-call-agent
  model:
    provider: anthropic
    model: claude-sonnet-4-6
    authSecretRef: anthropic-key

routes:
  - name: incident-response
    sources:
      - type: webhook
        name: pagerduty
        auth:
          secretName: pagerduty-secret
    targets:
      - instanceRef: incident-responder
        filters:
          data:
            - path: "body.messages.0.event"
              type: string
              value: "incident.trigger"
        taskTemplate: |
          Incident triggered: {{ .Input.body.messages.0.incident.title }}
          Service: {{ .Input.body.messages.0.incident.service.name }}
          Urgency: {{ .Input.body.messages.0.incident.urgency }}
          URL: {{ .Input.body.messages.0.incident.html_url }}
```

## 4. Install

```bash
helm install sympozium-events \
  oci://ghcr.io/chrisk700/sympozium-events/sympozium-events \
  -n sympozium-events --create-namespace \
  -f my-values.yaml
```

The NOTES.txt output printed after install lists your routes and the next verification steps.

## 5. Verify

```bash
# EventBus takes ~30s to start — check this first if other resources stay NotReady
kubectl get eventbus -n sympozium-events

# EventSource and Sensor should both show READY = true
kubectl get eventsource,sensor -n sympozium-events

# Watch for AgentRun creation when an event fires
kubectl get agentruns -n sympozium-system -w -l sympozium.ai/trigger=argo-events
```

## 6. Configure PagerDuty

In the PagerDuty webhook settings, point the webhook URL at your EventSource endpoint:

- **Tailnet-only**: `http://<hostname>.<tailnet>.ts.net:12000/pagerduty`
- **Funnel**: `https://incident-response.<tailnet>.ts.net/pagerduty`
- **Cluster-internal / port-forward**: `http://localhost:12000/pagerduty`

Set the HMAC secret to the same token used in step 2.
