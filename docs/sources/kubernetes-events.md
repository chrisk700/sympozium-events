# Kubernetes events

Watches Kubernetes resource events (ADD, UPDATE, DELETE) and fires a trigger on matches. Useful for reacting to pod failures, deployment rollouts, or custom resource state changes.

## Configuration

```yaml
sources:
  - type: resource
    name: pod-watcher
    resource: pods
    version: v1
    group: ""                        # optional, default "" (core group)
    namespace: sympozium-system      # optional, default .Values.namespace
    eventTypes:                      # optional, default ADD/UPDATE/DELETE
      - ADD
      - UPDATE
```

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Source name. |
| `resource` | yes | — | Kubernetes resource plural name (e.g. `pods`, `deployments`, `agentruns`). |
| `version` | no | `v1` | API version (e.g. `v1`, `v1beta1`). |
| `group` | no | `""` | API group. Empty string for core group. Use `apps` for Deployments, `sympozium.ai` for AgentRun. |
| `namespace` | no | `.Values.namespace` | Namespace to watch. |
| `eventTypes` | no | `[ADD, UPDATE, DELETE]` | Event types to capture. |

## taskTemplate

The full resource manifest is available at `.Input.body`:

```
Kubernetes resource event: {{ .Input.body.metadata.name }} ({{ .Input.body.kind }})
```

Override per target:

```yaml
targets:
  - instanceRef: platform-engineer
    taskTemplate: |
      Pod {{ .Input.body.metadata.name }} changed in {{ .Input.body.metadata.namespace }}.
      Status: {{ .Input.body.status.phase }}
```

## Example — watch for failed pods

```yaml
routes:
  - name: pod-failures
    sources:
      - type: resource
        name: pod-watcher
        resource: pods
        version: v1
        namespace: production
        eventTypes:
          - UPDATE
    targets:
      - instanceRef: sre-responder
        filters:
          data:
            - path: "body.status.phase"
              type: string
              value: "Failed"
        taskTemplate: |
          Pod failed: {{ .Input.body.metadata.name }}
          Namespace: {{ .Input.body.metadata.namespace }}
          Reason: {{ .Input.body.status.reason }}
```

## Watching custom resources

To watch `AgentRun` CRs:

```yaml
sources:
  - type: resource
    name: agentrun-watcher
    resource: agentruns
    group: sympozium.ai
    version: v1alpha1
    namespace: sympozium-system
```
