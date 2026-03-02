# AND conditions

By default, multiple sources on a route fire with OR logic — any source firing creates an `AgentRun`. Use `condition` for AND logic: all named sources must fire before a trigger is created.

## Configuration

```yaml
routes:
  - name: correlated-alert
    sources:
      - type: webhook
        name: pagerduty
        auth:
          secretName: pagerduty-secret
      - type: sqs
        name: cloudwatch
        region: us-east-1
        queue: cloudwatch-alerts
        credentialsSecret: aws-creds
    condition: "pagerduty && cloudwatch"   # both must fire
    targets:
      - instanceRef: incident-responder
        taskTemplate: |
          Correlated alert: PagerDuty incident and CloudWatch alarm both fired.
```

`condition` uses Argo Events native dependency condition syntax. Source names in the expression must exactly match `sources[].name` values on the same route.

## Resetting partial AND state

When only some of the required sources have fired, the Sensor holds partial state. If the remaining sources never fire (e.g. a transient alert that doesn't correlate), partial state accumulates indefinitely.

Use `conditionsResetCron` to clear it on a schedule:

```yaml
routes:
  - name: correlated-alert
    condition: "pagerduty && cloudwatch"
    conditionsResetCron: "*/5 * * * *"   # reset partial state every 5 min
```

Or set a global default for all routes:

```yaml
conditionsResetCron: "0 * * * *"   # reset every hour
```

Per-route `conditionsResetCron` overrides the global value.

!!! warning "Reliability caveat"
    The cron-based reset does not fire if the Sensor pod is down at the scheduled time ([argo-events#1543](https://github.com/argoproj/argo-events/issues/1543)). Partial AND-condition state may persist beyond the intended window if the pod restarts around the reset time.
