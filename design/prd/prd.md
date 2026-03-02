# PRD-07: sympozium-events

**Status**: Implementation complete (T01–T17); polish in progress
**Target**: Standalone Helm chart (`sympozium-events`) + companion repository
**Depends on**: Sympozium installed and operational
**Enables**: Event-driven AgentRun triggering from any signal source — webhooks, SQS, SNS, NATS, K8s events, schedules, and more

---

## Summary

Connect any external signal to Sympozium AgentRun executions via Argo Events. Rather than building a bespoke shim service, `sympozium-events` is an opinionated Helm wrapper chart that treats Argo Events as the signal ingestion layer and exposes a simple `routes/sources/targets` abstraction. Users configure routes; the chart generates EventSource and Sensor CRDs.

No changes to Sympozium are required. `AgentRun` is a first-class CRD — the same object the schedule controller creates. The Sympozium controller does not distinguish externally-created runs from scheduled ones; it reconciles both identically.

---

## Architecture

```
Signals (webhooks, SQS, SNS, schedules, K8s events...)
  → Argo Events EventSource  (namespace: argo-events)
    → EventBus (Argo Events JetStream — dedicated StatefulSet in argo-events namespace)
      → Sensor (filter, route, template payload)
        → K8s trigger: create AgentRun CR
          → Sympozium controller  (namespace: sympozium-system)
            → ephemeral agent pod
```

Argo Events is the universal entry point. Adding a new signal source is a new entry in `routes` — nothing else in the stack changes.

---

## Why Argo Events

Argo Events replaces what would otherwise be a bespoke shim service. It provides native support for webhooks, SQS, SNS, NATS, schedules, K8s events, and more. Payload extraction and field templating are built in via `parameters`. The `k8s` trigger type can create any CR — including `AgentRun` — without additional code. It is declarative, CRD-driven, and reconcilable.

Critically, Argo Events ships with NATS JetStream as its own EventBus. Sympozium already runs JetStream internally. Validation (T02) confirmed that the Argo Events `jetstream` EventBus type always manages its own StatefulSet — external NATS reuse is only available for the legacy non-JetStream `nats` type. Argo Events runs a dedicated JetStream bus in `argo-events` (~3-pod StatefulSet, ~300Mi idle). No degradation to Sympozium's NATS was observed; the two buses are fully isolated.

Alternatives evaluated (Tekton Triggers, Knative Eventing, KEDA, Keptn) are fully documented in the Alternatives section below. Argo Events is the only option that satisfies all core requirements natively.

---

## Routing Model

The `routes` key is the primary user-facing abstraction. It replaces the need to hand-author Argo Events EventSource or Sensor YAML directly.

A route has a `name`, a list of `sources`, and a list of `targets`. The chart generates one Sensor per route, iterating the targets list to produce one trigger block per entry — conceptually similar to an Argo CD ApplicationSet list generator.

**Sources** declare what signal types feed the route. Multiple sources on a single route means any one of them can fire the targets (OR condition). An AND condition is expressed via `condition`, mapping directly to Argo Events' native dependency conditions.

**Targets** declare which Sympozium instances receive AgentRuns and what the task prompt looks like. Each target carries an optional `filters` block using native Argo Events filter syntax passed through directly — no abstraction or DSL — so users familiar with Argo Events can port existing filter config as-is.

**taskTemplate** ships with sensible defaults per known source type and is overridden in values.

### Example: route values

```yaml
routes:
  - name: incident-response
    sources:
      - type: webhook
        name: pagerduty
      - type: sns
        name: aws-alerts
    targets:
      - instanceRef: incident-responder
        filters:
          data:
            - path: "body.incident.urgency"
              type: string
              value: "critical"
              comparator: "="
          exprLogicalOperator: "and"
        taskTemplate: |
          Alert: {{ .body.incident.title }}
          Service: {{ .body.incident.service.name }}
          Severity: {{ .body.incident.urgency }}
          URL: {{ .body.incident.html_url }}
      - instanceRef: sre-watchdog
        taskTemplate: |
          Observe: {{ .body.incident.title }}
          Severity: {{ .body.incident.urgency }}
```

### Multi-source AND condition

```yaml
routes:
  - name: correlated-incident
    sources:
      - type: webhook
        name: pagerduty
      - type: sns
        name: aws-alerts
    condition: pagerduty && aws-alerts
    conditionsResetCron: "*/30 * * * *"   # cron schedule to reset partial AND state (wall-clock, not per-event TTL)
    targets:
      - instanceRef: incident-responder
        taskTemplate: |
          Correlated alert: {{ .body.incident.title }}
```

The `conditionsResetCron` field maps the concept of an AND-condition window to Argo Events' `conditionsReset.byTime.cron` (validated T16). Argo Events has no native rolling-window TTL tied to event arrival time; `conditionsResetCron` resets accumulated partial AND state on a wall-clock cron schedule. This is a coarser approximation: `"*/30 * * * *"` resets every 30 minutes, not 30 seconds from each event's arrival. The global cron is overridable per route. Known limitation: reset does not fire if the Sensor pod is down at the scheduled cron time (argo-events#1543).

---

## Packaging

The integration is distributed as an opinionated wrapper chart that declares Argo Events as a dependency without forking or vendoring it.

```
charts/sympozium-events/
  Chart.yaml            # argo-events as a dependency, conditionally enabled
  values.yaml           # opinionated defaults including route definitions
  templates/
    eventbus.yaml       # EventBus CR — always emitted; Argo Events manages its own JetStream
    sensor.yaml         # generated from routes[].targets
    eventsource.yaml    # generated from routes[].sources
    rbac.yaml           # working RBAC: Argo Events SA + scoped AgentRun create role
    networkpolicy.yaml  # allow EventSource/Sensor → NATS across namespaces
```

EventSource and Sensor resources are generated from the `routes` values. Users never write Sensor YAML directly. RBAC is handled by the chart and is not a user-facing configuration surface in v1.

### Key values

```yaml
sympozium-events:
  argoEvents:
    install: true
    installNamespace: argo-events    # ignored if install: false
    resourceNamespace: argo-events   # where EventSource/Sensor CRs are created
  namespace: sympozium-system
  tailscale:
    enabled: false       # applies annotations only — operator install is out of band
    funnel: false
  # auth is per webhook source — see routes[].sources[].auth
  serviceAccount:
    create: true
    name: ""             # name of existing SA if create: false
  conditionsResetCron: ""   # cron schedule for AND condition state reset, e.g. "*/30 * * * *"; overridable per route
  routes: []
```

BYO Argo Events is supported via `argoEvents.install: false`. The chart skips the Argo Events dependency and deploys only EventSource, Sensor, and RBAC into the user's existing install. The `routes` abstraction, taskTemplates, and Tailscale annotations work identically in both modes.

### Install

```bash
helm repo add sympozium https://chrisk700.github.io/sympozium-events
helm install sympozium-events sympozium/sympozium-events
```

---

## Security

The chart does not change between deployment modes. Security is what you configure around it.

**Open** — EventSource service exposed via Ingress or LoadBalancer. Suitable for internal clusters or development.

**Secure (tailnet-only)** — Tailscale Operator installed out of band. Set `tailscale.enabled: true` and the chart injects Tailscale annotations into the EventSource `service.metadata.annotations` field. Senders POST to a static `*.ts.net` URL on port 12000. The connection initiates outbound from inside the cluster. Validated end-to-end (P08).

**Secure (public internet via Funnel)** — Set `tailscale.funnel: true`. The chart creates one Kubernetes Ingress per webhook route with `ingressClassName: tailscale` and `tailscale.com/funnel: "true"`. The Tailscale Operator provisions a Funnel proxy; senders POST to `https://<route-name>.<tailnet>.ts.net/<endpoint>`. Requires an ACL `nodeAttrs` entry granting the `funnel` attribute to `tag:k8s`. Chart support shipped (P08); public internet validation pending ACL setup. **Important**: `tailscale.com/funnel: "true"` on a Service annotation is silently ignored — Funnel only works via an Ingress resource.

1. **EventSource annotation passthrough** ✅ — inject via `spec.service.metadata.annotations` in the EventSource spec. Annotations pass through verbatim to the controller-managed Service. Used for tailnet-only mode.
2. **Ingress** ✅ — chart creates `ingressClassName: tailscale` Ingress per webhook route when `tailscale.funnel: true`. Used for public Funnel exposure.
3. **Post-install Helm Job** — ships as a documented fallback gated on `tailscale.annotationMode: job`; not needed for v1.
4. **Kyverno MutatingPolicy** — evaluated, not implemented. Intentionally heavy dependency (cluster-wide admission controller) for a chart-scoped feature.

**Per-sender auth** — HMAC via Argo Events `authSecret` field, configured per webhook source (`routes[].sources[].auth.secretName`). User provides the Secret — no chart-generated tokens. Missing auth on a webhook source is a template error. Opt out per source with `auth.allowInsecure: true` (NOTES.txt warns). Non-webhook sources (sqs, sns, calendar, resource, nats) carry their own auth in the source definition and are unaffected. (P15)

---

## Signal Sources

| Source | Status |
|---|---|
| Webhook (PagerDuty, GitHub, custom) | Day one |
| AWS SQS / SNS | Day one |
| Scheduled / heartbeat | Day one via `CalendarEventSource` |
| K8s resource events | Day one |
| NATS | Day one |
| Kafka, GCP Pub/Sub, Azure Event Hubs | Argo Events native — trivial to add |

---

## Under the Hood: Generated YAML

The `routes` abstraction generates standard Argo Events resources. The following examples show what the chart produces for common source types — useful for understanding the generated output or for constructing direct Argo Events YAML without the chart.

### RBAC (chart-managed)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argo-sensor-agentrun-creator
  namespace: sympozium-system
rules:
  - apiGroups: ["sympozium.ai"]
    resources: ["agentruns"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argo-sensor-agentrun-creator
  namespace: sympozium-system
subjects:
  - kind: ServiceAccount
    name: default          # chart substitutes the managed SA name
    namespace: argo-events
roleRef:
  kind: Role
  name: argo-sensor-agentrun-creator
  apiGroup: rbac.authorization.k8s.io
```

Note: the Sensor ServiceAccount lives in `argo-events` but needs `create` on `agentruns` in `sympozium-system`. This cross-namespace RoleBinding is easy to miss when hand-authoring — the chart handles it automatically.

### Minimal AgentRun (what a Sensor k8s trigger creates)

```yaml
apiVersion: sympozium.ai/v1alpha1
kind: AgentRun
metadata:
  generateName: event-triggered-   # unique name per event
  namespace: sympozium-system
  labels:
    sympozium.ai/trigger: argo-events
spec:
  instanceRef: <instance-name>
  agentId: event-triggered
  cleanup: delete                        # delete (default) or keep (retain CR for debugging)
  sessionKey: <derived-from-event-id>   # must be unique per run
  task: "<templated prompt>"
  model:
    provider: anthropic
    model: claude-sonnet-4-6
    authSecretRef: <secret-name>
```

`sessionKey` is not enforced unique by Sympozium — duplicate keys create two independent concurrent runs. Derive it from a stable event ID (incident ID, PR node ID, commit SHA) to avoid duplicate work. See Implementation Notes.

### GitHub PR merged — generated Sensor

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: github-pr-agent
  namespace: argo-events
spec:
  dependencies:
    - name: pr-merged
      eventSourceName: github
      eventName: pr-merged
      filters:
        data:
          - path: body.action
            type: string
            value:
              - closed
          - path: body.pull_request.merged
            type: bool
            value:
              - "true"
  triggers:
    - template:
        name: create-agentrun
        k8s:
          operation: create
          source:
            resource:
              apiVersion: sympozium.ai/v1alpha1
              kind: AgentRun
              metadata:
                generateName: github-pr-
                namespace: sympozium-system
                labels:
                  sympozium.ai/trigger: argo-events
                  sympozium.ai/event-source: github
              spec:
                instanceRef: platform-engineer
                agentId: event-triggered
                sessionKey: placeholder
                task: placeholder
                model:
                  provider: anthropic
                  model: claude-sonnet-4-6
                  authSecretRef: anthropic-key
          parameters:
            - src:
                dependencyName: pr-merged
                dataTemplate: "{{ .Input.body.pull_request.node_id }}"
              dest: spec.sessionKey
            - src:
                dependencyName: pr-merged
                dataTemplate: |
                  PR merged: {{ .Input.body.pull_request.title }}
                  Author: {{ .Input.body.pull_request.user.login }}
                  Branch: {{ .Input.body.pull_request.head.ref }}
              dest: spec.task
```

### PagerDuty incident — generated Sensor

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: pagerduty-incident-agent
  namespace: argo-events
spec:
  dependencies:
    - name: incident-triggered
      eventSourceName: pagerduty
      eventName: incident
      filters:
        data:
          - path: body.messages.0.event
            type: string
            value:
              - incident.trigger
  triggers:
    - template:
        name: create-agentrun
        k8s:
          operation: create
          source:
            resource:
              apiVersion: sympozium.ai/v1alpha1
              kind: AgentRun
              metadata:
                generateName: pagerduty-incident-
                namespace: sympozium-system
              spec:
                instanceRef: incident-responder
                agentId: event-triggered
                sessionKey: placeholder
                task: placeholder
                model:
                  provider: anthropic
                  model: claude-sonnet-4-6
                  authSecretRef: anthropic-key
          parameters:
            - src:
                dependencyName: incident-triggered
                dataTemplate: "{{ .Input.body.messages.0.incident.id }}"
              dest: spec.sessionKey
            - src:
                dependencyName: incident-triggered
                dataTemplate: |
                  Incident triggered: {{ .Input.body.messages.0.incident.title }}
                  Service: {{ .Input.body.messages.0.incident.service.name }}
                  Urgency: {{ .Input.body.messages.0.incident.urgency }}
                  URL: {{ .Input.body.messages.0.incident.html_url }}
              dest: spec.task
```

---

## Workstreams

These run in parallel and do not block each other.

**Validation** — prove out EventBus reuse against Sympozium's NATS, confirm AgentRun CR creation via Sensor k8s trigger with real payload templating, validate Tailscale exposure.

Tailscale validation: tailnet-only mode confirmed end-to-end (P08) — annotation passthrough to EventSource Service, operator proxy reconciliation, webhook POST → AgentRun `Succeeded`. Funnel (public internet) mode chart support shipped; live validation pending ACL `nodeAttrs` setup.

T02 validation confirmed that EventBus reuse fails — the Argo Events `jetstream` type always manages its own StatefulSet. Argo Events runs a dedicated JetStream bus in `argo-events` (~3 pods, ~300Mi idle). No NATS consolidation is planned for v1. The two buses are fully isolated; no degradation to Sympozium's NATS was observed.

**Packaging** — build the wrapper chart, wire the Argo Events dependency, implement the routes-to-Sensor generator in chart templates, define default taskTemplates per known source type, ensure working RBAC ships out of the box.

---

## Implementation Notes

**Cross-namespace RoleBinding** — the Sensor ServiceAccount lives in `argo-events` but needs to create AgentRun CRs in `sympozium-system`. The RBAC template must include a cross-namespace RoleBinding in `sympozium-system`. Easy to miss during template authoring.

**Tailscale exposure** — Two modes shipped (P08):
- *Tailnet-only* (`funnel: false`): annotation passthrough via EventSource `spec.service.metadata.annotations` (Option 1) validated on Argo Events v1.9.10 (T04). The Tailscale Operator reconciles the annotation on Argo Events-managed Services without issue. End-to-end confirmed: `sympozium-events-test.barred-porgy.ts.net:12000` → HTTP 200 → AgentRun `Succeeded`.
- *Funnel / public internet* (`funnel: true`): chart creates one Kubernetes Ingress (`ingressClassName: tailscale`, `tailscale.com/funnel: "true"`) per webhook route. `tailscale.com/funnel: "true"` on a Service annotation is silently ignored by the operator — Funnel requires an Ingress. Requires ACL `nodeAttrs` funnel grant for `tag:k8s`. Chart support complete; live validation pending.
- The post-install Job fallback (`tailscale.annotationMode: job`) ships for controllers that do not support `spec.service.metadata.annotations`; uses `kubectl wait --for=jsonpath='{.status.loadBalancer.ingress}'` to avoid a race condition.

**conditionsReset** — Argo Events has no native per-event rolling window. AND-condition time bounding is implemented via `conditionsReset.byTime.cron` (chart key: `conditionsResetCron`) — a wall-clock cron schedule reset, not a per-event TTL. Per-route override is supported. Known limitation: reset does not fire if the Sensor pod is down at the cron time (argo-events#1543). Validated T16.

**Argo Events version floor** — `dataTemplate` parameter syntax and `conditionsReset.byTime` both require Argo Events ≥ 1.7. Confirmed: v1.9.10 tested. Chart dependency pins `2.4.x` (chart 2.4.0–2.4.1 ship controller v1.8.x; chart 2.4.2+ ship v1.9.x). Documented in Chart.yaml annotations and `kubeVersion` field (P02 complete).

**sessionKey uniqueness** — Sympozium silently tolerates duplicate `sessionKey` values; two AgentRuns with the same key both reconcile independently. Uniqueness is the caller's responsibility — derive `sessionKey` from a stable event ID (incident ID, PR node ID, commit SHA). Content-based deduplication is not native to Argo Events; the NATS dedup window (300s) only deduplicates on the `Nats-Msg-Id` header, which must be set by the sender. See P09 for mitigation options.

**Multi-source port routing** — Two webhook sources on the same port (e.g. 12000) with distinct path endpoints work correctly. Argo Events runs a single HTTP server per EventSource and routes by path — no port conflict (validated T16).

**No TUI visibility** — event-triggered `AgentRun` objects appear in the Sympozium runs view like any other run, but there is no "configured triggers" pane. Trigger configuration lives in `routes` values, not in Sympozium itself.

**Cleanup policy** — `spec.cleanup` is a first-class field on the AgentRun CRD: `enum: [delete, keep]`, CRD default `delete`. The chart value `agentRun.cleanup` (default `delete`) is emitted directly in the Sensor AgentRun spec block. Set `keep` to retain the completed AgentRun CR for debugging Sensor parameter mapping. Confirmed against live cluster (P10).

**No concurrency policy** — Argo Sensors have no native concurrency guard or circuit-breaker field (P09 finding: `circuitBreaker` does not exist in the Sensor CRD). Sympozium has no cross-run concurrency enforcement for externally-created AgentRun objects. The only available rate control is trigger-level `rateLimit` (frequency throttle, not identity dedup).

**No memory injection** — the schedule controller's `includeMemory` logic is not available to externally-created runs. Inject memory context manually into `spec.task` if needed, or leave memory retrieval to the agent via its tools.

**helm-unittest subchart limitation** — The argo-events subchart is stored as a `.tgz` in `charts/`. Helm-unittest v1.0.3 cannot reference `.tgz`-bundled subchart template paths in the `templates:` list — and `notContainsDocument` is not a supported assertion type in this version. The BYO test (`byo_test.yaml`) verifies that chart-owned resources render correctly when `argoEvents.install: false`; suppression of subchart resources is enforced by Helm's native `condition:` field on the dependency and is not redundantly tested here.

---

## Alternatives Considered

### Tekton Triggers

Purpose-built for CI/CD: `EventListener`, `TriggerBinding`, `TriggerTemplate`, `Interceptor`. Webhook ingestion and K8s CR creation via `TriggerTemplate` maps cleanly to this use case.

**Falls short:** Non-webhook sources (SQS, SNS, schedules, NATS) are not natively supported. No built-in EventBus, no NATS reuse opportunity. Multi-source AND conditions not supported. Four CRD types per trigger vs Argo Events' two.

**Verdict:** Good for webhook-only CI/CD. Too narrow for multi-signal ingestion.

### Knative Eventing

CloudEvents-native event mesh: `Broker`, `Trigger`, `Channel`, `Source`. Fan-out and filtering are native strengths.

**Falls short:** Designed to deliver events to HTTP endpoints, not create K8s CRs directly. Triggering an `AgentRun` would require an intermediary service — reintroducing the shim we were eliminating. Heavy install footprint (cert-manager, Channel implementation, optionally Knative Serving). No native SQS/SNS sources in core.

**Verdict:** Architecturally elegant but cannot create K8s CRs natively. Poor fit without shim work.

### KEDA

Scales Kubernetes workloads based on external event sources (SQS depth, Kafka lag, Prometheus metrics, cron). Lightweight, 65+ scalers, well-understood.

**Falls short:** KEDA is a scaling tool, not an event routing tool. It does not create new CRs, extract payload fields, or template task prompts. `AgentRun` is an ephemeral job-per-event pattern — not a scalable-deployment pattern.

**Verdict:** Wrong tool for this job. Relevant only if Sympozium adopts a queue-depth-driven worker pool model.

### Keptn

Deployment lifecycle orchestration: pre/post deployment checks, quality gates, SLO-based evaluations. Uses NATS internally.

**Falls short:** Narrowly focused on deployment lifecycle events — not a general-purpose signal ingestion layer. Not suitable for arbitrary webhook ingestion (PagerDuty, GitHub). In active architectural transition (v1 EOL, v2 RC) — adoption risk.

**Verdict:** Not a fit for general signal ingestion.

### Comparison

| | Argo Events | Tekton Triggers | Knative Eventing | KEDA | Keptn |
|---|---|---|---|---|---|
| Multi-signal sources | ✅ 20+ native | ⚠️ Webhook only | ⚠️ Extensions needed | ✅ 65+ scalers | ❌ Deployment events only |
| K8s CR trigger | ✅ Native | ✅ Via TriggerTemplate | ❌ Needs shim | ❌ Wrong model | ❌ Wrong model |
| Payload templating | ✅ Native parameters | ✅ TriggerBinding | ❌ Not native | ❌ N/A | ❌ N/A |
| NATS EventBus reuse | ❌ jetstream type owns its StatefulSet | ❌ None | ⚠️ Via Channel adapter | ❌ None | ⚠️ Uses NATS internally |
| Multi-source AND/OR | ✅ Native | ❌ No | ❌ No | ❌ No | ❌ No |
| Install footprint | Medium | Medium | Heavy | Light | Medium |
| CNCF status | Incubating | Graduated | Graduated | Graduated | Incubating |

Argo Events is the only option that satisfies all core requirements natively.

---

## Decisions

- Argo Events is the signal ingestion layer. No bespoke shim service.
- `sympozium-events` lives as a standalone repository under the author's GitHub, with the Sympozium org as the natural future home.
- EventSource and Sensor resources are generated from `routes` values. Users configure routes, not Argo Events primitives.
- Filters use native Argo Events syntax passed through directly — no abstraction, no DSL, no escape hatch needed.
- taskTemplates ship as chart defaults per known source type and are overridden in values.
- RBAC ships working out of the box. Customisation is phase 2.
- Tailscale operator install is out of band. The chart applies annotations only.
- Argo Events installs in `argo-events` namespace. Sympozium remains in `sympozium-system`. EventSource, Sensor, and EventBus CRs are created in `argo-events` (`argoEvents.resourceNamespace` default — P04). A dedicated `sympozium-events` namespace was considered and rejected: it would require chart-managed Namespace creation and extra controller RBAC, with no compelling isolation benefit beyond the `sympozium.ai/component=events-bridge` label selector.
- BYO Argo Events is supported via `argoEvents.install: false`.
- `conditionsResetCron` is empty by default (no automatic AND state reset). Set to a cron expression (e.g. `*/30 * * * *`) globally, overridable per route. The name is intentionally distinct from `conditionWindow` to avoid implying equivalence to a per-event TTL.
- Sympozium's NATS is currently vanilla with no auth or TLS. Known follow-on concern (see PRD-04), does not block this work.
- Observability is future work.
- EventBus reuse is not supported. The Argo Events `jetstream` EventBus type always manages its own StatefulSet; external NATS reuse requires the legacy `nats` type (`exotic` mode) and is not used.
- `conditionWindow` (PRD concept) was renamed `conditionsResetCron` in the chart. The semantics differ: it is a wall-clock cron schedule that resets accumulated AND state, not a per-event rolling TTL.
- Sympozium's NATS service is named `nats` (not `eventbus-default-js` as assumed during design). This does not affect the implementation since EventBus reuse was ruled out.
- **Upstream bundling deferred to v1.0** (P13) — `sympozium-events` remains a standalone chart. Bundling as a sub-chart of the Sympozium parent was evaluated and deferred: the `routes` schema is at v0.1 with no stability guarantee; Argo Events adds ~300Mi idle footprint that is opt-in only; the bridge spans `argo-events` and `sympozium-system` namespaces with a cross-namespace RoleBinding that complicates parent chart ownership; and independent releases allow faster iteration while P08/P10 are still open. If bundled in future, the values namespace should be `events.*` (not `sympozium.events.*`). Trigger conditions for re-evaluation: routes schema at v1.0, P08/P10 closed, PRD-08 (`SympoziumEventTrigger`) scoped.

---

## Open Questions

### Closed

- **EventBus reuse** [Closed — T02] — Reuse fails. The Argo Events `jetstream` EventBus type always manages its own StatefulSet. External reuse is only available for the legacy (non-JetStream) `nats` type (`exotic` mode). Argo Events runs a dedicated JetStream bus in `argo-events` (~300Mi idle, fully isolated). Sympozium NATS service name is `nats` (not `eventbus-default-js` as assumed). See Implementation Notes.
- **conditionWindow scope** [Closed — T16] — Not a native Argo Events field. Implemented as `conditionsReset.byTime.cron` (chart key: `conditionsResetCron`). Wall-clock cron schedule reset, not per-event TTL. Per-Sensor scope (overridable per route). Known reliability gap: reset does not fire if the Sensor pod is down at the scheduled time (argo-events#1543).
- **Argo Events version floor** [Closed — T03/P02] — `dataTemplate` requires ≥ 1.7; `conditionsReset.byTime` requires ≥ 1.6. Confirmed: v1.9.10 tested. Chart dependency pins `2.4.x` (chart 2.4.0–2.4.1 ship controller v1.8.x; chart 2.4.2+ ship v1.9.x). Documented in Chart.yaml annotations (`sympozium.ai/argoEventsMinVersion`, `sympozium.ai/kubernetesMinVersion`) and `kubeVersion: ">=1.25.0"` field (P02 complete).
- **sessionKey uniqueness** [Closed — T03/T15] — Sympozium does not enforce uniqueness; duplicate `sessionKey` values create two independent concurrent runs. Argo Events deduplicates by `Nats-Msg-Id` header only — not by payload content. Two identical POSTs create two AgentRuns. Deduplication must be handled at the Sensor level or via sender-side `Nats-Msg-Id` headers. See P09 for mitigation options.
- **Cleanup policy** [Closed — P10] — `spec.cleanup` exists on the AgentRun CRD: `enum: [delete, keep]`, default `delete`. `delete` removes the pod after completion; `keep` retains the AgentRun CR for debugging Sensor parameter mapping. Chart adds `agentRun.cleanup: delete` to `values.yaml` and emits `cleanup: {{ $.Values.agentRun.cleanup }}` in the Sensor AgentRun spec block.
- **Tailscale exposure** [Partially closed — P08] — Tailnet-only mode validated end-to-end: annotation passthrough confirmed on Argo Events v1.9.10, operator reconciles Argo Events-managed Services, webhook POST → AgentRun `Succeeded`. Funnel (public internet) mode: chart ships correct Ingress-based approach (`tailscale.funnel: true`); live validation pending ACL `nodeAttrs` funnel grant for `tag:k8s`. Key finding: `tailscale.com/funnel: "true"` on a Service annotation is silently ignored — Funnel requires a Kubernetes Ingress with `ingressClassName: tailscale`.

### Open

- **Native CRD path** — a `SympoziumEventTrigger` CRD that wraps Argo Events behind native Sympozium primitives is tracked as a separate v2 effort (PRD-08). The wrapper chart validates the routing model; the native CRD is informed by what the chart reveals in practice.
- **AgentRun upsert / session continuation** — When the same incident alert fires again after the NATS dedup window (300s), a second `AgentRun` is created rather than continuing the existing investigation. PRD-08 feature ask: `AgentRun` should support a `sessionMode: append` (or `upsert`) field that routes subsequent event payloads to an existing active run rather than creating a new one. This requires Sympozium-side sessionKey uniqueness enforcement and is not a chart-side concern. (P09)
