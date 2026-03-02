# Polish: sympozium-events

**Status**: Planned
**Follows**: PRD-07 implementation (T01–T17 complete)
**Purpose**: Post-implementation hardening, gap closure, and release readiness

---

## Summary

The chart core is complete and smoke-tested (T01–T17). This polish track closes the open questions surfaced during implementation, hardens the test and CI infrastructure, and prepares the chart for a v0.1 public release. It also folds in the items from `TODO.md` that were deferred as out-of-scope during implementation.

No new chart capabilities are introduced in this track. The routing model, source types, and RBAC are frozen at the T17 baseline.

---

## What Was Learned During Implementation

The following PRD open questions were resolved during T02–T17 and inform the polish tasks directly.

**EventBus reuse (PRD OQ-1)** — Closed. Argo Events `jetstream` EventBus always manages its own StatefulSet; external reuse is only available for the legacy non-JetStream `nats` type. Argo Events runs a second JetStream bus (~300Mi idle). No degradation to Sympozium's NATS was observed. The PRD's `reuseSympoziumEventbus` design is dropped; `eventbus.yaml` always emits the CR.

**conditionWindow (PRD OQ-2)** — Closed with caveats. Argo Events has no rolling-window TTL tied to event arrival time. The chart maps the concept to `conditionsReset.byTime.cron` — a wall-clock cron schedule that resets accumulated partial AND state. This is coarser than a per-event TTL: `conditionsResetCron: "* * * * *"` approximates a 60s window but does not guarantee it. Known upstream issue: `conditionsReset` does not fire if the Sensor pod is down at the cron time (argo-events#1543).

**sessionKey uniqueness (PRD OQ-4)** — Partially closed. T15 confirmed that Argo Events fires once per HTTP POST regardless of payload identity. If the same webhook payload is sent twice (e.g. a PagerDuty re-alert for the same incident), two AgentRun CRs are created — both will share the same `sessionKey` derived from the incident ID. Sympozium does not currently enforce sessionKey uniqueness or support an upsert/append mode. This is a known limitation requiring a Sympozium-level feature to resolve (see P09).

**Tailscale annotation injection (PRD OQ — T04)** — Closed. Option 1 (EventSource `spec.service.metadata.annotations` passthrough) works on Argo Events v1.9.10. Annotations pass through verbatim to the controller-managed Service. Options 2 (post-install Job) and 3 (Kyverno) are not needed for v1. The Job template ships as a documented fallback but is a no-op by default.

**Argo Events version floor** — Confirmed: `dataTemplate` parameter syntax and `conditionsReset.byTime` both require Argo Events ≥ 1.7. The chart locks `2.4.x` (Helm chart); chart 2.4.0–2.4.1 ship controller v1.8.x, chart 2.4.2+ ship v1.9.x. Documented in Chart.yaml annotations and `kubeVersion` field (P02 complete).

**Multi-source port routing (T16)** — Two webhook sources on the same port (e.g. 12000) with distinct path endpoints (`/source-a`, `/source-b`) work correctly. Argo Events runs a single HTTP server per EventSource and routes by path; there is no port conflict.

---

## Scope

Four phases. Phases 1–2 are required for v0.1 release. Phases 3–4 are post-release polish.

```
Phase 1 — Close open questions     P01 – P04   PRD cleanup + chart label strategy
Phase 2 — Test + CI                P05 – P07   lint, unit tests, CI pipeline, chart repo
Phase 3 — Live validation          P08 – P10   Tailscale Funnel, dedup design, cleanup field
Phase 4 — Docs + future            P11 – P13   NOTES, README, upstream bundling eval
```

---

## Phase 1 — Close Open Questions

Resolves the PRD open questions that were answered during T02–T17 and closes the `TODO.md` items that require documentation or chart-level changes.

**P01 — PRD: close resolved open questions**
Update `docs/prd/prd.md` with findings from T02–T17. Mark the five open questions as closed with their outcomes. Remove the `reuseSympoziumEventbus` key from the Key Values example block. Add a Findings section.

**P02 — Chart.yaml: version floor annotation**
Add `artifacthub.io/containsValues` and `annotations` block to `Chart.yaml` documenting the Argo Events ≥ 1.7 requirement and minimum Kubernetes version (1.25+ for stable NetworkPolicy). The chart dependency already pins `2.4.x` but this is not surfaced to users installing in BYO mode.

**P03 — Sympozium label strategy**
Add `sympozium.ai/component: events-bridge` to the common labels helper (`_helpers.tpl`) so it appears on all Sensor, EventSource, and RBAC resources. This gives the Sympozium CLI a stable selector to list active triggers across namespaces: `kubectl get sensors,eventsources -A -l sympozium.ai/component=events-bridge`. The AgentRun already carries `sympozium.ai/trigger: argo-events`; this closes the gap on the infrastructure side.

**P04 — Namespace default decision**
Decide whether `argoEvents.resourceNamespace` should default to `argo-events` (current, Argo-native) or `sympozium-events` (opinionated isolation). Document the decision in values.yaml comments and the Decisions section of the PRD. Remove stale `reuseSympoziumEventbus` comments from values.yaml.

---

## Phase 2 — Test + CI

**P05 — helm lint + helm unittest**
`ci/two-routes.yaml` exists but is only used with `helm template` by hand. Add `ct.yaml` (chart-testing config), `helm lint` invocation, and `helm unittest` snapshot tests for the generated Sensor and EventSource output against `ci/two-routes.yaml`. Cover: single-source route, multi-source AND route, BYO mode, Tailscale annotation mode.

**P06 — Taskfile**
Add `Taskfile.yaml` with tasks: `lint`, `template`, `test`, `package`, `release`. Replaces the ad-hoc `helm template` commands used during T06–T17. Used locally and called by CI.

**P07 — GitHub Actions: CI + chart-releaser**
Two workflows: (1) `ci.yaml` — runs on PR: `helm lint`, `helm unittest`, `helm template` with `ci/two-routes.yaml`; (2) `release.yaml` — runs on tag push: packages chart, publishes to GitHub Pages via `chart-releaser`. Targets the repo URL `https://chrisk700.github.io/sympozium-events`.

---

## Phase 3 — Live Validation

**P08 — Tailscale Funnel end-to-end** ✓
Tailnet-only validated: annotation passthrough confirmed on Argo Events v1.9.10, operator reconciles Argo Events-managed Services without issue, webhook POST → AgentRun `Succeeded`. Key finding: `tailscale.com/funnel: "true"` on a Service annotation is silently ignored — Funnel requires a Kubernetes Ingress with `ingressClassName: tailscale`. Chart updated: `tailscale-ingress.yaml` added; `tailscale.funnel: true` now creates one Ingress per webhook route. Funnel (public internet) validation pending ACL `nodeAttrs` funnel grant for `tag:k8s`.

**P09 — Session deduplication: design + docs**
T15 finding: identical payloads POSTed twice create two AgentRuns with the same `sessionKey`. The `sessionKey` derived from a stable event ID (incident ID, PR node ID) collides on re-alerts. Document the three strategies available today: (a) Argo Events NATS dedup window (300s, requires `Nats-Msg-Id` header — sender-side); (b) Argo Events `CircuitBreaker` on the Sensor — suppresses re-trigger for a configurable period; (c) Sensor data filter on a stable ID field. Propose which to expose in values and implement if warranted. The Sympozium upsert/append AgentRun mode is tracked separately as PRD-08 input.

**P10 — spec.cleanup field**
The PRD mentions `spec.cleanup: delete` vs `keep` on AgentRun as a debugging aid. Verify whether this is a real AgentRun CRD field. If yes, add `agentRun.cleanup` to values.yaml (default `delete`) and emit it in the Sensor k8s trigger resource block.

---

## Phase 4 — Docs + Future

**P11 — NOTES.txt**
Replace the empty Helm scaffold NOTES.txt with post-install guidance: how to verify EventSource and Sensor reconciliation (`kubectl get eventsource,sensor -n ...`), how to watch for AgentRun creation, a quickstart `routes` snippet, and a link to the values reference.

**P12 — README.md**
The README is currently one line. Expand with: architecture summary, install quickstart, values reference table (top-level keys only), link to `docs/prd/prd.md`, and the confirmed signal sources table from the PRD.

**P13 — Upstream bundling evaluation**
Evaluate whether `sympozium-events` should be offered as a feature-toggled dependency of the upstream Sympozium Helm chart (accessible via `sympozium.events.enabled: true`). This reduces the user's install surface from two `helm install` commands to one. Prerequisite: v0.1 is stable and the chart API (routes, sources, targets) is settled. Output: decision document, not implementation.

---

## Known Limitations (not addressed in this track)

**conditionsReset reliability** — `conditionsReset.byTime.cron` does not fire if the Sensor pod is down at the scheduled time (argo-events#1543). Partial AND state can persist across pod restarts. No mitigation in this track; document as a known limitation.

**Content-based dedup** — Argo Events deduplicates by `Nats-Msg-Id` header (sender-set), not by payload content. Two identical payloads without that header create two AgentRuns. P09 addresses what can be done chart-side; a full fix requires Sympozium upsert support.

**No concurrency policy** — Multiple simultaneous events can produce multiple concurrent AgentRuns targeting the same `instanceRef`. Sympozium has no cross-run concurrency enforcement for externally-created runs. Not addressed in this track.

**No memory injection** — `spec.task` is templated from event payload only. The schedule controller's `includeMemory` logic is not available to event-triggered runs. Inject memory context manually into `taskTemplate` or leave retrieval to the agent via tools.

---

## Decisions

- conditionWindow (PRD concept) is implemented as `conditionsResetCron` — a cron schedule, not a per-event TTL. The names are intentionally kept different to avoid implying equivalence.
- EventBus reuse is not supported. The chart always creates its own Argo Events JetStream StatefulSet.
- Tailscale Option 1 (annotation passthrough) is the shipped approach. Option 2 (Job) ships as a documented fallback. Option 3 (Kyverno) is not implemented.
- sessionKey collision on re-alerts is a known limitation. No chart-side mitigation until Sympozium supports AgentRun upsert.
- Namespace default (P04): `argoEvents.resourceNamespace` stays `argo-events`. Matches the Argo Events controller namespace (`install: true`) and BYO convention. A dedicated `sympozium-events` CR namespace was considered but rejected — it would require a chart-managed `Namespace` resource and extra controller RBAC without compelling isolation benefit. `sympozium-events` resources are identifiable by the `sympozium.ai/component=events-bridge` label.
- Upstream bundling (P13): `sympozium-events` stays standalone. Bundling deferred to v1.0 — routes API is not yet stable, Argo Events footprint (~300Mi) is opt-in only, cross-namespace ownership complicates parent chart, and independent releases allow faster iteration while P08/P10 are open. Future bundling values namespace: `events.*`. Trigger conditions: routes v1.0 stable, P08/P10 closed, PRD-08 scoped.
- AgentRun routing / instanceRef (P14): `instanceRef` is the name of a `SympoziumInstance` CR in `sympozium-system` — a stable, user-assigned name discoverable via `kubectl get sympoziuminstances -n sympozium-system`. No schema change is needed; the gap is documentation. `agentRun.agentId` is chart-global (not per-target): a known limitation for multi-agent-type deployments, acceptable for v0.1. `agentId: event-triggered` is a convention from the chart's own tests, not a Sympozium built-in — users must set it to match an agent in their SympoziumInstance. NOTES.txt (P11) and README (P12) must document `instanceRef` discovery and `agentId` semantics. Per-target `agentId` support is a known v0.2 candidate.
- Label strategy (P03): `sympozium.ai/component: events-bridge` is added to the common labels helper and propagates to all infrastructure resources (Sensor, EventSource, EventBus, RBAC, NetworkPolicy, tailscale Job). AgentRuns carry `sympozium.ai/trigger: argo-events` and `sympozium-events/route: <name>`. Canonical selectors:
  ```bash
  # All EventSources and Sensors managed by sympozium-events
  kubectl get eventsources,sensors -A -l sympozium.ai/component=events-bridge

  # All AgentRuns created by event triggers
  kubectl get agentruns -n sympozium-system -l sympozium.ai/trigger=argo-events
  ```
