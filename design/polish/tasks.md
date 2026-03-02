# Tasks: sympozium-events polish

Status key: `[ ]` pending | `[>]` in-progress | `[x]` complete

All tasks follow from the PRD-07 implementation (T01–T17 complete).
Reference the overview in `docs/polish/polish.md`.
Task details and sub-items: `tasks/<ID>.md`

---

## Phase 0 — Retrospective

### [x] P00 — Retrospective: close cross-document gaps from T01–T17
**Depends on**: nothing

---

## Phase 1 — Close Open Questions

### [x] P01 — PRD: close resolved open questions
**Depends on**: P00

### [x] P02 — Chart.yaml: version floor annotation
**Depends on**: nothing

### [x] P03 — Sympozium label strategy
**Depends on**: nothing

### [x] P04 — Namespace default decision
**Depends on**: nothing

### [x] P14 — AgentRun routing investigation: instanceRef UX
**Depends on**: nothing

---

## Phase 2 — Test + CI

### [x] P05 — helm lint + helm unittest
**Depends on**: P03, P04

### [x] P06 — Taskfile
**Depends on**: P05

### [x] P07 — GitHub Actions: CI + chart-releaser
**Depends on**: P05, P06

---

## Phase 3 — Live Validation

### [x] P08 — Tailscale Funnel end-to-end
**Depends on**: P03, live cluster with Tailscale Operator

### [x] P09 — Session deduplication: design + docs
**Depends on**: nothing

### [x] P10 — spec.cleanup field
**Depends on**: live Sympozium cluster

---

## Phase 4 — Docs + Future

### [x] P11 — NOTES.txt
**Depends on**: P03, P04

### [x] P12 — README: values reference + architecture diagrams
**Depends on**: P01, P03, P09, P11, P14

### [x] P13 — Upstream bundling evaluation
**Depends on**: P07

---

## Phase 5 — Security

### [x] P15 — Endpoint auth: secure by default
**Depends on**: nothing
