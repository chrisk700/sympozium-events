# CRD management

## Bundled version

| Component | Version |
|---|---|
| Argo Events Helm chart | `2.4.x` (locked to `2.4.20` in `Chart.lock`) |
| Argo Events controller | `v1.9.x` (`v1.9.10` at current lock) |

The dependency range is `2.4.x` — patch updates are picked up automatically on `helm dependency update`. The exact locked version is always visible in `charts/sympozium-events/Chart.lock`.

## How CRDs are installed

`sympozium-events` bundles Argo Events as a subchart. Argo Events requires three CRDs:

- `eventbus.argoproj.io`
- `eventsources.argoproj.io`
- `sensors.argoproj.io`

The chart installs these CRDs from its `crds/` directory. Helm processes `crds/` before planning or applying any templates, which guarantees the CRDs are registered with the API server before the `EventBus` CR and other Argo Events resources are created.

## Why not the subchart's own CRD templates?

The Argo Events Helm chart ships its CRDs in `templates/crds/`. Helm applies `templates/` resources in a single pass — there is no ordering guarantee between the CRD templates and the `EventBus` CR that depends on them. Helm also validates the full manifest against the API server before applying anything, so if the CRDs are not yet registered, the install fails immediately.

Placing CRDs in `crds/` is Helm's purpose-built solution to this ordering problem.

## Build-time extraction

The `crds/` directory is not committed to source — it is generated at build time. The extraction step runs after `helm dependency update`:

```bash
task crds:extract   # or: task lint (which includes it)
```

This unpacks the three CRD YAML files from the bundled `argo-events-*.tgz` and strips the Helm template guards (`{{- if ... }}`/`{{- end }}`) that wrap them in the subchart, since files in `crds/` are not processed as Go templates.

The extracted CRDs are always in sync with the argo-events version pinned in `Chart.lock`. Bumping the argo-events dependency and re-running `task lint` updates them automatically.

## Upgrading CRDs

CRDs in `crds/` are applied on both `helm install` and `helm upgrade`. Applying an identical CRD version is a no-op. If the chart is upgraded to a newer Argo Events dependency, the CRDs will be upgraded automatically.

To skip CRD installation entirely, use `helm install --skip-crds`. Only use this if you are certain the correct CRD versions are already present in the cluster.
