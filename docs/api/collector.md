# Collector API

The Collector service (port 8321) proxies requests to your observability backends. All endpoints except `/health` require API key authentication via `Authorization: Bearer <key>`.

## Health

**`GET /health`**

Returns service health with backend connectivity status.

```bash
curl http://localhost:8321/health
```

```json
{
  "status": "ok",
  "version": "0.1.0",
  "services": {
    "prometheus": "connected",
    "elasticsearch": "connected",
    "kubernetes": "configured",
    "cloudwatch": "not_configured"
  }
}
```

## Prometheus

**`POST /api/v1/prometheus/query`**

Execute a PromQL range query.

```bash
curl -X POST http://localhost:8321/api/v1/prometheus/query \
  -H "Authorization: Bearer traced-dev-key" \
  -H "Content-Type: application/json" \
  -d '{"query": "up", "start": "1h", "end": "now", "step": "1m"}'
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `query` | string | *required* | PromQL expression |
| `start` | string | `1h` | Start time (relative like `1h`, `30m` or epoch) |
| `end` | string | `now` | End time |
| `step` | string | `1m` | Query resolution step |

## Elasticsearch

**`POST /api/v1/logs/search`**

Search application logs.

```bash
curl -X POST http://localhost:8321/api/v1/logs/search \
  -H "Authorization: Bearer traced-dev-key" \
  -H "Content-Type: application/json" \
  -d '{"query": "level:ERROR AND service:payment", "time_range": "1h", "size": 50}'
```

## Kubernetes

**`POST /api/v1/kubernetes/pods`** — List pods in a namespace
**`POST /api/v1/kubernetes/pod/describe`** — Describe a specific pod
**`POST /api/v1/kubernetes/events`** — Get events for a namespace
**`POST /api/v1/kubernetes/deployment`** — Describe a deployment
**`POST /api/v1/kubernetes/nodes`** — Get node status
**`POST /api/v1/kubernetes/logs`** — Get container logs

## Change Detection

**`POST /api/v1/changes/deployments`** — List recent deployments
**`POST /api/v1/changes/configmaps`** — List ConfigMap changes

## Remediation Actions

**`POST /api/v1/actions/restart-pod`** — Delete a pod (triggers controller restart)
**`POST /api/v1/actions/rollback`** — Rollback a deployment
**`POST /api/v1/actions/scale`** — Scale a deployment's replica count

!!! warning "Remediation endpoints"
    These endpoints perform destructive actions. They are only called by the Cloud service after human approval in Slack. The Collector authenticates every request via API key.
