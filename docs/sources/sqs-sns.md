# AWS SQS / SNS

## SQS

Poll an SQS queue for messages. Each message fires a trigger and creates an `AgentRun`.

```yaml
sources:
  - type: sqs
    name: my-queue
    region: us-east-1
    queue: my-queue-name
    waitTimeSeconds: 20          # optional, default 20
    credentialsSecret: aws-creds # optional
```

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Source name. |
| `region` | yes | — | AWS region. |
| `queue` | yes | — | Queue name (not URL). |
| `waitTimeSeconds` | no | `20` | Long-poll duration. |
| `credentialsSecret` | no | — | Secret name with keys `access-key-id` and `secret-access-key`. |

### Credentials secret

```bash
kubectl create secret generic aws-creds \
  --from-literal=access-key-id=<key-id> \
  --from-literal=secret-access-key=<secret> \
  -n sympozium-events
```

If `credentialsSecret` is omitted, the EventSource pod uses the pod's IAM identity (IRSA or instance profile).

### taskTemplate

```
SQS message received: {{ .Input.body.Message }}
```

The full SQS message envelope is available at `.Input.body`. Override per target:

```yaml
targets:
  - instanceRef: my-instance
    taskTemplate: |
      SQS event: {{ .Input.body.Subject }}
      Body: {{ .Input.body.Message }}
```

---

## SNS

Receive SNS notifications via HTTP subscription. Argo Events starts an HTTP server and SNS delivers to it.

```yaml
sources:
  - type: sns
    name: aws-alerts
    topicArn: "arn:aws:sns:us-east-1:123456789012:aws-alerts"
    port: "12000"              # optional, default "12000"
    endpoint: /aws-alerts      # optional, default /<name>
    credentialsSecret: aws-creds  # optional
```

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Source name. |
| `topicArn` | yes | — | Full ARN of the SNS topic. |
| `port` | no | `"12000"` | Port the EventSource HTTP server listens on. |
| `endpoint` | no | `/<name>` | HTTP path SNS delivers to. |
| `credentialsSecret` | no | — | Same format as SQS credentials. |

### Endpoint exposure

SNS requires the EventSource endpoint to be reachable from AWS. Use [Tailscale Funnel](../tailscale/funnel.md) or a standard Ingress to expose it.

### taskTemplate

```
SNS notification received: {{ .Input.body.Message }}
```
