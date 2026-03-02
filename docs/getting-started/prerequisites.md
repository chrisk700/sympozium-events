# Prerequisites

## Cluster

| Requirement | Version |
|---|---|
| Kubernetes | ≥ 1.25 |
| Helm | ≥ 3.12 |
| Sympozium operator | installed in `sympozium-system` |
| Argo Events controller | bundled (default) or BYO ≥ 1.7.0 |

The chart bundles Argo Events as a subchart and manages it as part of the release. If you already have Argo Events running in the same cluster, uninstall it before installing this chart — running two Argo Events controller-managers in the same namespace will conflict.

## Sympozium operator

You need at least one `SympoziumInstance` CR in `sympozium-system` with an agent configured. The chart creates `AgentRun` CRs targeting that instance — it does not install the operator itself.

```bash
# Verify the operator is running
kubectl get sympoziuminstances -n sympozium-system
```

## Tailscale (optional)

If you want webhook endpoints exposed on your tailnet or the public internet, the [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator) must be installed out of band before enabling `tailscale.enabled: true`. See [Tailscale](../tailscale/tailnet-only.md).

## Development tools

For local chart work only — not needed to install the chart:

| Tool | Install |
|---|---|
| `helm` ≥ 3.12 | [helm.sh](https://helm.sh/docs/intro/install/) |
| `helm-unittest` plugin | `helm plugin install https://github.com/helm-unittest/helm-unittest --verify=false` |
| `task` (go-task) ≥ 3 | [taskfile.dev](https://taskfile.dev/installation/) |
