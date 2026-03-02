# Values reference

Full reference for all `values.yaml` keys. See the [annotated values.yaml](https://github.com/chrisk700/sympozium-events/blob/main/charts/sympozium-events/values.yaml) for inline comments and a complete route example.

## Argo Events

| Key | Type | Default | Description |
|---|---|---|---|
| `argoEvents.install` | bool | `true` | Install Argo Events as a subchart. Advanced escape hatch — not recommended to disable. |

## Sympozium

| Key | Type | Default | Description |
|---|---|---|---|
| `namespace` | string | `sympozium-system` | Namespace where `AgentRun` CRs are created. |

## EventBus

| Key | Type | Default | Description |
|---|---|---|---|
| `eventbus.name` | string | `default` | Name of the EventBus CR. |
| `eventbus.version` | string | `latest` | JetStream image tag. Pin to a specific version in production. |

## AgentRun

| Key | Type | Default | Description |
|---|---|---|---|
| `agentRun.agentId` | string | `event-triggered` | Agent ID within the target SympoziumInstance. **Must be overridden.** |
| `agentRun.cleanup` | string | `delete` | `delete` removes the AgentRun CR after completion. `keep` retains it for debugging. |
| `agentRun.model.provider` | string | — | LLM provider (e.g. `anthropic`, `openai`). Required. |
| `agentRun.model.baseURL` | string | — | Provider base URL. Optional for hosted providers. |
| `agentRun.model.model` | string | — | Model name (e.g. `claude-sonnet-4-6`). Required. |
| `agentRun.model.authSecretRef` | string | — | Name of a Secret containing the API key. Required. |

## Tailscale

| Key | Type | Default | Description |
|---|---|---|---|
| `tailscale.enabled` | bool | `false` | Enable Tailscale exposure. Requires Tailscale Operator out of band. |
| `tailscale.funnel` | bool | `false` | Create one Ingress per webhook route for public internet exposure via Funnel. |
| `tailscale.hostname` | string | `""` | Shared Ingress hostname in funnel mode. Defaults to the Helm release name. |
| `tailscale.annotations` | map | `{}` | Annotations injected into the EventSource Service. Used when `funnel: false`. |
| `tailscale.annotationMode` | string | `annotation` | `annotation` (preferred) or `job` (post-install Job fallback). |
| `tailscale.kubectl.image` | string | `bitnami/kubectl:1.29` | Image used by the post-install Job. `annotationMode: job` only. |

## ServiceAccount

| Key | Type | Default | Description |
|---|---|---|---|
| `serviceAccount.create` | bool | `true` | Create a ServiceAccount. Set `false` to bind an existing SA. |
| `serviceAccount.name` | string | `""` | Name of an existing SA. Used when `create: false`. |

## NetworkPolicy

| Key | Type | Default | Description |
|---|---|---|---|
| `networkPolicy.enabled` | bool | `true` | Allow egress from EventSource/Sensor pods to EventBus (:4222 client, :6222 cluster routing) and `sympozium-system`. |

## Conditions reset

| Key | Type | Default | Description |
|---|---|---|---|
| `conditionsResetCron` | string | `""` | Global cron schedule for resetting partial AND-condition state. Empty disables reset. Overridable per route via `routes[].conditionsResetCron`. |

## Default task templates

| Key | Type | Description |
|---|---|---|
| `defaults.taskTemplates.webhook` | string | Default prompt for generic webhook sources. |
| `defaults.taskTemplates.pagerduty` | string | Named preset for PagerDuty webhook sources. |
| `defaults.taskTemplates.github` | string | Named preset for GitHub webhook sources. |
| `defaults.taskTemplates.sqs` | string | Default prompt for SQS sources. |
| `defaults.taskTemplates.sns` | string | Default prompt for SNS sources. |
| `defaults.taskTemplates.calendar` | string | Default prompt for calendar sources. |
| `defaults.taskTemplates.resource` | string | Default prompt for Kubernetes resource sources. |
| `defaults.taskTemplates.nats` | string | Default prompt for NATS sources. |

Template resolution order per trigger: `targets[].taskTemplate` → `defaults.taskTemplates[source.name]` → `defaults.taskTemplates[source.type]`.

## Routes

| Key | Type | Description |
|---|---|---|
| `routes[].name` | string | Route name. Used as the EventSource/Sensor CR name and (for webhooks) the endpoint path. |
| `routes[].sources[]` | list | One or more signal sources. See [Signal sources](../sources/webhook.md). |
| `routes[].targets[]` | list | One or more AgentRun targets. |
| `routes[].targets[].instanceRef` | string | Name of the target `SympoziumInstance` CR in `sympozium-system`. |
| `routes[].targets[].taskTemplate` | string | Go-template task prompt. Overrides `defaults.taskTemplates`. |
| `routes[].targets[].filters` | object | Argo Events filter conditions passed through verbatim to the Sensor trigger. |
| `routes[].condition` | string | AND condition expression (e.g. `"source-a && source-b"`). See [AND conditions](../advanced/and-conditions.md). |
| `routes[].conditionsResetCron` | string | Per-route cron override for AND-condition reset. |
| `routes[].tailscaleHostname` | string | Per-route Funnel Ingress hostname. Defaults to `routes[].name`. |
