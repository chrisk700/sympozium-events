# Session deduplication

By default, Argo Events fires one trigger per HTTP POST regardless of payload identity. Two identical webhook payloads from the same incident create two `AgentRun` CRs, both sharing the same `sessionKey`.

## Mitigation: `Nats-Msg-Id` header

The JetStream event bus deduplicates messages with the same `Nats-Msg-Id` header within a 300-second window. Configure your webhook sender to include a stable identifier:

```
Nats-Msg-Id: incident-<id>
```

Example with curl:

```bash
curl -X POST https://<host>/pagerduty \
  -H 'Nats-Msg-Id: incident-12345' \
  -H 'X-Hub-Signature-256: sha256=<hmac>' \
  -H 'Content-Type: application/json' \
  -d '<payload>'
```

This prevents a second `AgentRun` from being created for the same event within the 300-second window. It does not handle re-alerts that arrive after the window expires.

## Limitations of this approach

- Dedup window is fixed at 300 seconds by JetStream — not configurable via this chart.
- Does not handle the same incident alerting again after the window expires.
- Sender must support custom HTTP headers.

## Long-term fix

The correct solution is Sympozium-side: `AgentRun` should support a `sessionMode: append` or upsert field that routes subsequent event payloads to an existing active run. This is tracked as a PRD-08 feature request and is not a chart concern.
