# Tasks: 07-sympozium-events

Status key: `[ ]` pending | `[>]` in-progress | `[x]` complete

All code lives in a new standalone `sympozium-events` repository (private, GitHub).
Reference the design in `prd/planned/07-sympozium-events/prd.md`.
Task details and sub-items: `tasks/<ID>.md`

---

## Phase 0 — Repository

### [x] T01 — Init repository and chart scaffold
**Depends on**: nothing

---

## Phase 1 — Validation

### [x] T02 — EventBus reuse validation
**Depends on**: T01, live Sympozium cluster

### [x] T03 — AgentRun k8s trigger validation
**Depends on**: T01, live Sympozium cluster

### [x] T04 — Tailscale exposure validation
**Depends on**: T01, Tailscale Operator installed in cluster

---

## Phase 2 — Chart Core

### [x] T05 — RBAC template
**Depends on**: T03

### [x] T06 — EventSource template
**Depends on**: T02

### [x] T07 — Sensor template
**Depends on**: T03, T06

### [x] T08 — EventBus template
**Depends on**: T02

### [x] T09 — NetworkPolicy template
**Depends on**: T06, T07

---

## Phase 3 — Chart Features

### [x] T10 — Default taskTemplates
**Depends on**: T07

### [x] T11 — Tailscale annotation injection
**Depends on**: T04, T06

### [x] T12 — Post-install Job (Tailscale fallback)
**Depends on**: T04, T11

### [x] T13 — BYO Argo Events mode
**Depends on**: T05, T06, T07, T08

### [x] T14 — Per-sender HMAC auth
**Depends on**: T06, T07

---

## Phase 4 — Integration Tests

### [x] T15 — Smoke test: webhook → AgentRun end-to-end
**Depends on**: T05, T06, T07, T08, T09, T10

### [x] T16 — Smoke test: multi-source AND condition
**Depends on**: T15

### [x] T17 — Smoke test: BYO Argo Events
**Depends on**: T13, T15
