# Architecture Overview

Traced uses a split architecture with two services: the **Collector** runs inside the customer's cluster, and the **Cloud** service runs the AI orchestration.

## System Architecture

```mermaid
graph TB
    subgraph "Customer Cluster"
        PROM[Prometheus]
        ES[Elasticsearch]
        K8S[Kubernetes API]
        CW[CloudWatch]
        COLL[Collector Service<br/>Port 8321]
        COLL --> PROM
        COLL --> ES
        COLL --> K8S
        COLL --> CW
    end

    subgraph "Traced Cloud"
        WH[Webhook Endpoints]
        ORCH[Orchestrator]
        OBS[Observability Agent]
        KA[Kubernetes Agent]
        CA[Change Agent]
        LLM[Claude API]
        STORE[Incident Store<br/>SQLite]
        SLACK[Slack Bot]
        DASH[Dashboard]
        RB[Runbook Library]

        WH --> ORCH
        ORCH --> OBS
        ORCH --> KA
        ORCH --> CA
        OBS --> LLM
        KA --> LLM
        CA --> LLM
        ORCH --> LLM
        ORCH --> STORE
        ORCH --> SLACK
        ORCH --> RB
    end

    COLL <-->|HTTP + API Key Auth| ORCH
    OBS -->|Tool calls| COLL
    KA -->|Tool calls| COLL
    CA -->|Tool calls| COLL

    PD[PagerDuty] --> WH
    AM[Alertmanager] --> WH
    CUSTOM[Custom Webhooks] --> WH
    SLACK <--> SLACKAPI[Slack API<br/>Socket Mode]
```

## Data Flow

### Investigation Lifecycle

```mermaid
sequenceDiagram
    participant Alert as Alert Source
    participant Cloud as Cloud Service
    participant RB as Runbook Library
    participant Agents as Specialist Agents
    participant Claude as Claude API
    participant Collector as Collector
    participant Infra as Prometheus/ES/K8s
    participant Slack as Slack
    participant Store as Incident Store

    Alert->>Cloud: POST /webhooks/*
    Cloud->>Cloud: Normalize + Dedup
    Cloud->>RB: Match runbook
    Cloud->>Store: Save trigger
    Cloud->>Agents: Dispatch (parallel)

    par Observability Agent
        Agents->>Claude: System prompt + runbook guidance
        Claude->>Agents: Tool call: query_prometheus
        Agents->>Collector: POST /api/v1/prometheus/query
        Collector->>Infra: PromQL query
        Infra->>Collector: Metrics data
        Collector->>Agents: Scrubbed response
        Agents->>Claude: Tool result
        Claude->>Agents: report_finding
    and Kubernetes Agent
        Agents->>Claude: System prompt + runbook guidance
        Claude->>Agents: Tool call: get_pods
        Agents->>Collector: POST /api/v1/kubernetes/pods
        Collector->>Infra: K8s API call
        Infra->>Collector: Pod data
        Collector->>Agents: Scrubbed response
    and Change Agent
        Agents->>Collector: POST /api/v1/changes/deployments
    end

    Agents->>Cloud: Agent results + findings
    Cloud->>Claude: Synthesize root cause
    Claude->>Cloud: Root cause + remediation actions
    Cloud->>Store: Save results
    Cloud->>Slack: Post findings + approval buttons
    Slack->>Cloud: User approves action
    Cloud->>Collector: Execute remediation
    Cloud->>Cloud: Post-fix verification
    Cloud->>Slack: Resolution summary
    Cloud->>Store: Save verification
```

## Component Details

### Collector Service

The Collector runs in the customer's cluster and acts as a secure proxy. It:

- Queries Prometheus, Elasticsearch, Kubernetes API, and CloudWatch
- Scrubs sensitive data (emails, IPs, credentials, tokens) before sending responses
- Authenticates requests via API key (constant-time comparison)
- Exposes remediation action endpoints (restart pod, rollback, scale)

The Collector never initiates outbound connections to the Cloud service — it only responds to requests.

### Cloud Service

The Cloud service is the brain. It:

- Receives alerts via webhooks (PagerDuty, Alertmanager, generic)
- Deduplicates alerts using fingerprint + cooldown window
- Matches runbooks based on alert content
- Orchestrates specialist agents in parallel
- Manages the Claude tool-calling loop for each agent
- Synthesizes root cause from all agent findings
- Manages the Slack approval flow for remediation
- Runs post-fix verification
- Persists everything to SQLite
- Serves the incident dashboard

### Specialist Agents

Each agent is a Claude tool-calling loop with a specialized system prompt and tool set:

| Agent | System Prompt Focus | Tools |
|-------|-------------------|-------|
| Observability | Metrics, logs, traces | `query_prometheus`, `search_logs`, `query_cloudwatch` |
| Kubernetes | Cluster state, pod health | `get_pods`, `describe_pod`, `get_events`, `get_logs`, etc. |
| Change Detection | Recent deployments, config | `get_recent_deployments`, `get_configmap_changes` |

Agents run in parallel with a configurable timeout (default 30s). If a runbook matches, its investigation steps are injected into the agent's prompt.

### Safety Model

See [Safety Model](safety-model.md) for the full tier system.
