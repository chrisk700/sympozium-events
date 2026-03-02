# NATS

Subscribe to a NATS subject and fire a trigger on each message.

## Configuration

```yaml
sources:
  - type: nats
    name: my-subject
    url: nats://nats:4222
    subject: sympozium.events
```

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Source name. |
| `url` | yes | — | NATS server URL. |
| `subject` | yes | — | Subject to subscribe to. |

## taskTemplate

The message body is available at `.Input.body`:

```
NATS message received: {{ .Input.body | toJson }}
```

Override per target:

```yaml
targets:
  - instanceRef: my-instance
    taskTemplate: |
      NATS event on sympozium.events:
      {{ .Input.body | toJson }}
```

## Example

```yaml
routes:
  - name: nats-trigger
    sources:
      - type: nats
        name: events
        url: nats://nats.nats-system.svc.cluster.local:4222
        subject: sympozium.events
    targets:
      - instanceRef: my-sympozium-instance
        taskTemplate: |
          Event: {{ .Input.body | toJson }}
```

!!! note
    The NATS server referenced here is your application NATS, separate from the JetStream EventBus that Argo Events manages internally.
