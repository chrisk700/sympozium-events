# sympozium-events

Argo Events bridge for Sympozium — routes external signals (webhooks, SQS, SNS, schedules, Kubernetes events, NATS) to Sympozium `AgentRun` CRs without any bespoke shim service.

```mermaid
flowchart LR
    subgraph external["External signals"]
        W[Webhook<br>PagerDuty / GitHub / custom]
        Q[AWS SQS / SNS]
        C[Calendar<br>schedule]
        K[K8s events]
        N[NATS]
    end

    subgraph release["sympozium-events namespace<br>(release namespace)"]
        AEC[Argo Events<br>controller]
        ES[EventSource]
        EB[(EventBus<br>NATS JetStream)]
        S[Sensor]
        AEC -. manages .-> ES & EB & S
    end

    subgraph sym["sympozium-system namespace"]
        AR[AgentRun CR]
        SC[Sympozium<br>controller]
        P[ephemeral<br>agent pod]
    end

    W & Q & C & K & N --> ES
    ES -- publish --> EB
    EB -- subscribe --> S
    S -- "k8s trigger:<br>create AgentRun" --> AR
    AR --> SC --> P
```

Each entry in `routes` produces one EventSource and one Sensor. Multiple sources on a route fire with OR logic by default; an AND condition uses Argo Events native dependency conditions.

## Install

```bash
helm install my-release \
  oci://ghcr.io/chrisk700/sympozium-events/sympozium-events \
  -f my-values.yaml
```

No `helm repo add` needed — OCI charts are referenced directly.

→ [Prerequisites](getting-started/prerequisites.md) · [Installation](getting-started/installation.md) · [First route](getting-started/first-route.md)
