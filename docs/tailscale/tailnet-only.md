# Tailnet-only

Exposes webhook endpoints on your tailnet. Reachable only by tailnet members — no public internet exposure.

**Protocol**: plain HTTP on the webhook port (default 12000). The WireGuard layer provides transport encryption; there is no TLS at the HTTP layer.

**URL form**: `http://<hostname>.<tailnet>.ts.net:<port>/<endpoint>`

## Prerequisites

The [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator) must be installed in your cluster before enabling this mode.

## Configuration

```yaml
tailscale:
  enabled: true
  funnel: false
  annotations:
    tailscale.com/expose: "true"
    tailscale.com/hostname: my-sympozium-events   # optional
```

The annotations are injected into `spec.service.metadata.annotations` on the EventSource Service. The Tailscale Operator watches for these annotations and creates a proxy pod.

| Annotation | Required | Description |
|---|---|---|
| `tailscale.com/expose` | yes | Must be `"true"` to enable tailnet exposure. |
| `tailscale.com/hostname` | no | Short hostname on the tailnet. Defaults to an operator-assigned name. |

## Verify

```bash
# Check the EventSource Service has the Tailscale annotations
kubectl get svc -n sympozium-events -o yaml | grep tailscale

# The operator creates a proxy pod — it should be Running
kubectl get pods -n tailscale
```

Once the proxy is running, your webhook endpoint is reachable from any tailnet member:

```
http://my-sympozium-events.<tailnet>.ts.net:12000/pagerduty
```

## annotationMode: job (fallback)

Some older Argo Events controller versions do not support `spec.service.metadata.annotations` passthrough. In that case, use the post-install Job fallback:

```yaml
tailscale:
  enabled: true
  annotationMode: job
  kubectl:
    image: bitnami/kubectl:1.29
```

The Job patches the Service directly after install. For production environments, pin the `kubectl` image to a digest.
