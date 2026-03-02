# CLI selectors

Label selectors for querying resources managed by `sympozium-events`.

## EventSources and Sensors

```bash
# All EventSources and Sensors created by sympozium-events
kubectl get eventsource,sensor -A -l sympozium.ai/component=events-bridge

# Resources for a specific route
kubectl get eventsource,sensor -n sympozium-events -l sympozium-events/route=incident-response
```

## AgentRuns

```bash
# All AgentRuns created by event triggers
kubectl get agentruns -n sympozium-system -l sympozium.ai/trigger=argo-events

# AgentRuns for a specific route
kubectl get agentruns -n sympozium-system -l sympozium-events/route=incident-response

# Watch for new AgentRuns
kubectl get agentruns -n sympozium-system -w -l sympozium.ai/trigger=argo-events
```

## EventBus

```bash
kubectl get eventbus -n sympozium-events
```

## Tailscale Ingresses

```bash
# Ingresses created by sympozium-events in funnel mode
kubectl get ingress -n sympozium-events -l sympozium.ai/component=events-bridge
```

## Full status check

```bash
# Quick health snapshot
kubectl get eventbus,eventsource,sensor -n sympozium-events
kubectl get agentruns -n sympozium-system -l sympozium.ai/trigger=argo-events --sort-by=.metadata.creationTimestamp
```
