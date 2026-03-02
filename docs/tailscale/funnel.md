# Funnel (public internet)

Exposes webhook endpoints on the public internet via Tailscale Funnel. Also reachable from within the tailnet — no separate Service annotation needed.

**Protocol**: HTTPS on port 443, Let's Encrypt certificate, TLS 1.3.

**URL form**: `https://<route-name>.<tailnet>.ts.net/<endpoint>`

## Prerequisites

- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator) installed in your cluster
- ACL `nodeAttrs` entry granting the `funnel` attribute (see below)

## ACL prerequisite

Add this to your tailnet policy in the Tailscale admin console **before** enabling Funnel. Without it, the proxy pod starts but Funnel is silently inactive:

```json
"nodeAttrs": [
  {
    "target": ["tag:k8s"],
    "attr": ["funnel"]
  }
]
```

## Configuration

```yaml
tailscale:
  enabled: true
  funnel: true
  hostname: ""   # optional — defaults to the Helm release name
```

The chart creates one Kubernetes Ingress per webhook route with `ingressClassName: tailscale` and `tailscale.com/funnel: "true"`. The Tailscale Operator provisions a Funnel proxy.

```yaml
routes:
  - name: incident-response   # becomes <incident-response>.<tailnet>.ts.net
    sources:
      - type: webhook
        name: pagerduty
        auth:
          secretName: pagerduty-secret
    targets:
      - instanceRef: my-instance
```

## Hostname customisation

By default the Ingress hostname is the route `name`. Override per route with `tailscaleHostname`:

```yaml
routes:
  - name: incident-response
    tailscaleHostname: pagerduty-hook   # → pagerduty-hook.<tailnet>.ts.net
```

## Verify

```bash
# Ingress should have an address once Funnel is provisioned
kubectl get ingress -n sympozium-events

# The endpoint is then reachable from the public internet:
# https://incident-response.<tailnet>.ts.net/pagerduty
```

!!! warning "Validation status"
    Funnel mode deploys the correct Ingress resource. Public internet exposure via Funnel has not been validated end-to-end. Tailnet-only mode is fully validated.

!!! note "Service annotation vs Ingress"
    `tailscale.com/funnel: "true"` applied directly to a Service (not an Ingress) is silently ignored by the Tailscale Operator. Funnel requires an Ingress resource, which this chart creates automatically when `funnel: true`.
