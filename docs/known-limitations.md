# Known limitations

## conditionsReset reliability

The cron-based reset (`conditionsResetCron`) does not fire if the Sensor pod is down at the scheduled time ([argo-events#1543](https://github.com/argoproj/argo-events/issues/1543)). Partial AND-condition state may persist beyond the intended window if the pod restarts around the scheduled time.

## Session deduplication

Duplicate webhook payloads create duplicate `AgentRun` CRs. Mitigate with sender-side `Nats-Msg-Id` headers within a 300-second JetStream dedup window. See [Session deduplication](advanced/session-dedup.md).

## No concurrency policy

Argo Sensors have no native circuit-breaker or concurrency guard. Multiple in-flight triggers for the same route are not serialized. The only available rate control is trigger-level `rateLimit` (frequency throttle, not identity dedup).

## agentId is chart-global

All routes in a single chart install share one `agentRun.agentId`. Deployments targeting different agent types require separate chart installs.

## Tailscale Funnel not validated end-to-end

`tailscale.funnel: true` deploys the correct Ingress resource but public internet exposure via Funnel has not been tested. Tailnet-only mode is fully validated. Funnel requires an additional ACL `nodeAttrs` entry granting the `funnel` attribute to `tag:k8s` in the Tailscale admin console.

## eventbus.version: latest

The default `eventbus.version: latest` is an unpinned image tag. Pin to a specific NATS version in production to avoid unexpected changes on pod restart.
