# Quick Start

Get Traced running locally in 5 minutes with Docker Compose.

## Prerequisites

- Docker and Docker Compose installed
- An Anthropic API key ([console.anthropic.com](https://console.anthropic.com))

## 1. Clone and Configure

```bash
git clone https://github.com/traced-ai/traced.ai.git
cd traced.ai
cp .env.example .env
```

Edit `.env` and set at minimum:

```bash
ANTHROPIC_API_KEY=sk-ant-api03-...
```

!!! tip "Optional: Slack integration"
    To enable the Slack bot, also set `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN`. See the [Setup Guide](setup-guide.md#2-create-a-slack-app) for how to create a Slack app.

## 2. Start the Stack

```bash
docker-compose up --build
```

This starts four services:

| Service | Port | Purpose |
|---------|------|---------|
| Collector | 8321 | Queries Prometheus, Elasticsearch, K8s |
| Cloud | 8322 | AI agents, webhooks, incident API, dashboard |
| Prometheus | 9090 | Dev metrics backend |
| Elasticsearch | 9200 | Dev log backend |

## 3. Verify

```bash
# Collector health — should show "connected" for Prometheus and ES
curl http://localhost:8321/health | python3 -m json.tool

# Cloud health
curl http://localhost:8322/health | python3 -m json.tool
```

## 4. Trigger a Test Investigation

```bash
curl -X POST http://localhost:8322/webhooks/generic \
  -H "Content-Type: application/json" \
  -d '{
    "message": "OOMKilled in payment-service — pods restarting in namespace payments",
    "source": "test",
    "service": "payment-service",
    "severity": "critical"
  }'
```

## 5. Check Results

```bash
# List all incidents
curl http://localhost:8322/api/v1/incidents | python3 -m json.tool

# Or open the dashboard
open http://localhost:8322/dashboard
```

## What Happens Behind the Scenes

1. The webhook normalizes the alert into an `IncidentTrigger`
2. The **oom_kill** runbook matches based on "OOMKilled" in the message
3. Three agents investigate in parallel (observability, kubernetes, change detection)
4. Each agent uses Claude to decide which tools to call (PromQL queries, log searches, pod inspection)
5. Claude synthesizes all findings into a root cause with confidence score
6. Remediation actions are suggested with safety tier classifications
7. Everything is persisted and visible on the dashboard

## Next Steps

- [Full Setup Guide](setup-guide.md) — Connect to your real cluster
- [Architecture Overview](../architecture/overview.md) — Understand how agents work
- [Custom Runbooks](../configuration/runbooks.md) — Add playbooks for your incidents
