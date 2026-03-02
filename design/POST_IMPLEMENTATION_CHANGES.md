# Post-Implementation Changes

**Covers**: changes made after the P00–P15 polish track completed, during the documentation, live-cluster e2e, and pre-release hardening session.

This document bridges `design/polish/polish.md` (the planned polish track) and the current state of the chart at v0.1.7.

---

## 1. Release pipeline: chart-releaser → OCI (GHCR)

**Polish plan (P07)**: two-workflow CI with `chart-releaser` publishing to a `gh-pages` branch and `index.yaml`.

**What shipped**: `chart-releaser` was replaced with direct OCI push to GHCR (`oci://ghcr.io/chrisk700/sympozium-events/sympozium-events`). No `gh-pages` branch, no `index.yaml`, no `CR_TOKEN` secret. `GITHUB_TOKEN` is sufficient.

**Rationale**: OCI eliminates the perma `gh-pages` branch; `main` is now the canonical source of truth. `helm install` without `helm repo add` matches the appliance model. `chart-releaser` was designed for the traditional Helm repo index pattern, which is superseded by OCI for new charts.

**Impact on P13 (upstream bundling)**: the migration plan in P13.md references `helm repo add sympozium ...`; this is no longer the install path. If bundling is revisited, note that parent chart users pull the subchart tgz directly — OCI is compatible with this.

---

## 2. Documentation site

**Polish plan**: P12 expanded the README; no dedicated docs site was planned.

**What shipped**: a full Zensical/MkDocs docs site deployed to GitHub Pages via Actions (`docs.yaml` workflow). 16 pages covering getting started, all source types, Tailscale, advanced topics (CRD management, session dedup, AND conditions), and a values reference. README thinned to a gateway with a link to the docs site.

**Note on tooling**: MkDocs Material was initially used but emits a deprecation warning in every build (mkdocs-material 9.7.3 → Material team pushing users to Zensical). Switched to Zensical — drop-in replacement, same `mkdocs.yml`, same Mermaid support.

---

## 3. CRD extraction: build-time step added

**Polish plan**: not anticipated. P07 assumed a straightforward `helm dependency update` + package pipeline.

**Discovery**: during the first live-cluster install, Helm failed with `no kind "EventBus" is registered`. Root cause: the Argo Events subchart places CRDs in `templates/`, not `crds/`. Helm validates the full template set (including our `eventbus.yaml`) against the API server before applying anything — so the EventBus CRD is not yet registered when validation runs.

**Fix**: at build time, extract the three Argo Events CRDs from the bundled subchart tgz into the chart's own `crds/` directory. Helm processes `crds/` before templates, guaranteeing CRD registration before CR validation.

**Implementation**:
- `task crds:extract` added to `Taskfile.yaml`
- `task lint` calls `crds:extract` (dep update → extract → lint)
- Both `ci.yaml` and `release.yaml` delegate to `task crds:extract`
- `charts/sympozium-events/crds/` is gitignored (always regenerated from `Chart.lock`)
- `argo-events.crds.install: false` in values prevents the subchart from installing CRDs a second time

**Cross-platform gotchas encountered**:
- macOS `sed` does not support `\s` in basic regex — use `[[:space:]]`
- `sed -i ''` (macOS) vs `sed -i` (GNU/Linux) — resolved with `sed -i.bak` + cleanup on both platforms
- `{{` inside Taskfile `cmds` is interpreted as Go template syntax — escaped with `{{"{{"}}`

---

## 4. EventSource `spec.service` — always required for webhook/SNS

**Discovery**: after the CRD fix, the EventSource pod started and connected to NATS successfully, but no Kubernetes Service was created for the webhook endpoint. Argo Events 1.9.x requires an explicit `spec.service` block in the EventSource spec to create the Service — it does not auto-create one from the presence of webhook sources.

**Fix**: `eventsource.yaml` now always emits `spec.service.ports` when there are webhook or SNS sources (both listen on HTTP). Calendar, SQS, resource, and NATS sources are unaffected. Tailscale annotations are injected into `spec.service.metadata.annotations` as before (conditional on `tailscale.enabled`).

---

## 5. NetworkPolicy: NATS cluster port 6222 missing

**Discovery**: after the first `helm install`, EventBus pods stayed at 2/3 ready indefinitely. NATS logs showed `Waiting for routing to be established...` and `JetStream has not established contact with a meta leader`. The NetworkPolicy only allowed egress on port 4222 (NATS client) — port 6222 (NATS cluster routing between StatefulSet peers) was blocked.

**Fix**: port 6222 added to the egress allow rule alongside 4222.

---

## 6. Single-namespace collapse + BYO mode removed

**Polish plan (P04)**: decided to keep `argoEvents.resourceNamespace: argo-events`. Rationale at the time: "matches the Argo Events controller namespace and BYO convention."

**What shipped**: this decision was reversed. The two-namespace split (controller in `sympozium-events`, resources in `argo-events`) was inherited from the shared-cluster Argo Events pattern. As an appliance chart with no sharing model, the split adds cross-namespace RBAC complexity with no isolation benefit.

**Changes**:
- `argoEvents.resourceNamespace` and `argoEvents.installNamespace` removed from values
- All templates use `{{ .Release.Namespace }}` — everything lands in the release namespace
- `docs/advanced/byo-argo-events.md` removed; `byo_test.yaml` removed
- `argoEvents.install` kept as an undocumented escape hatch (no docs, not recommended)
- Prerequisites page updated: if existing Argo Events is running in the cluster, uninstall before installing this chart

**Impact on P13 decision record**: the namespace model concern in P13 ("bridge spans three concerns: CRs in `argo-events`, RBAC in `sympozium-system`, cross-namespace RoleBinding") is now resolved — there are only two namespaces: `sympozium-events` (everything chart-owned) and `sympozium-system` (AgentRun creation target). The namespace complexity argument against bundling weakens slightly. The other arguments (API stability, footprint, release coupling) remain.

---

## 7. Bugs fixed during e2e (not in polish plan)

| Bug | Root cause | Fix |
|---|---|---|
| `no kind "EventBus" is registered` | Argo Events CRDs in subchart `templates/` not `crds/` | Build-time CRD extraction (§3) |
| No Service created for webhook EventSource | Argo Events 1.9.x requires explicit `spec.service` | Always emit `spec.service.ports` for HTTP sources (§4) |
| EventBus 2/3 pods, NATS cluster won't form | NetworkPolicy blocked port 6222 | Add port 6222 to egress rule (§5) |
| Sensor uses `default` SA, AgentRun creation forbidden | `spec.template.spec.serviceAccountName` wrong nesting — Sensor CRD uses `spec.template.serviceAccountName` | Remove extra `spec:` level in sensor.yaml |
| `task crds:extract` fails with template error | `{{` in Taskfile interpreted as Go template | Escape with `{{"{{"}}`  |
| `sed` not stripping indented Helm guards | macOS sed rejects `\s`; `sed -i ''` fails on Linux | Use `[[:space:]]` and `sed -i.bak` |

---

## Current state (v0.1.8)

- Single namespace model: all chart resources in the Helm release namespace
- OCI distribution via GHCR — no `helm repo add` required
- Build-time CRD extraction: `task lint` / CI workflows handle this automatically
- 29 unit tests, lint clean
- e2e validated on sympozium-02: install → EventBus up → webhook trigger → AgentRun created
- Docs site live at https://chrisk700.github.io/sympozium-events
