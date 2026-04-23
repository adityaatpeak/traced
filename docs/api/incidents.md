# Incidents API

The Cloud service exposes REST endpoints for querying investigation history.

## List Incidents

**`GET /api/v1/incidents`**

Returns a paginated list of incidents, newest first.

### Query Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | int | 50 | Results per page (1-200) |
| `offset` | int | 0 | Skip N results |
| `status` | string | — | Filter by status: `triggered`, `investigating`, `completed` |

### Example

```bash
curl "http://localhost:8322/api/v1/incidents?status=completed&limit=10" | python3 -m json.tool
```

### Response

```json
{
  "incidents": [
    {
      "id": "1fc14287",
      "source": "alertmanager",
      "raw_message": "[Alertmanager/critical] payment-service: Error rate above 5%",
      "status": "completed",
      "root_cause_summary": "OOM kill in payment-service due to memory leak after v2.3.1 deploy",
      "root_cause_confidence": 0.85,
      "created_at": "2026-04-23T08:38:21.611441+00:00",
      "updated_at": "2026-04-23T08:38:42.123000+00:00",
      "metadata": {
        "alertname": "HighErrorRate",
        "service": "payment-service",
        "severity": "critical"
      }
    }
  ],
  "total": 42,
  "limit": 10,
  "offset": 0
}
```

---

## Get Incident Detail

**`GET /api/v1/incidents/{incident_id}`**

Returns full incident details including agent results, actions, and verifications.

### Example

```bash
curl http://localhost:8322/api/v1/incidents/1fc14287 | python3 -m json.tool
```

### Response

```json
{
  "id": "1fc14287",
  "source": "alertmanager",
  "raw_message": "[Alertmanager/critical] payment-service: Error rate above 5%",
  "status": "completed",
  "root_cause_summary": "OOM kill in payment-service due to memory leak after v2.3.1 deploy",
  "root_cause_confidence": 0.85,
  "created_at": "2026-04-23T08:38:21.611441+00:00",
  "metadata": { "alertname": "HighErrorRate", "service": "payment-service" },

  "agent_results": [
    {
      "agent_name": "observability",
      "status": "completed",
      "execution_time": 8.2,
      "findings": [
        {
          "category": "metric_anomaly",
          "summary": "Memory usage climbing from 60% to 98% over last 2 hours",
          "severity": "high",
          "evidence": { "query": "container_memory_working_set_bytes", "values": "..." }
        }
      ]
    },
    {
      "agent_name": "kubernetes",
      "status": "completed",
      "execution_time": 5.1,
      "findings": [
        {
          "category": "pod_issue",
          "summary": "Pod payment-service-7b8f4c-x2k1m terminated with OOMKilled",
          "severity": "critical"
        }
      ]
    }
  ],

  "actions": [
    {
      "action_id": "act-a1b2c3",
      "description": "Rollback payment-service to v2.3.0",
      "command": "kubectl rollout undo deployment/payment-service -n payments",
      "safety_tier": "medium_risk",
      "status": "approved",
      "approved_by": "U0SRE_LEAD1",
      "executed_at": "2026-04-23T08:39:12.000000+00:00"
    }
  ],

  "verifications": [
    {
      "verified": 1,
      "summary": "All checks passed — fix appears to have resolved the issue.",
      "checks": [
        { "name": "pod_health", "status": "ok", "detail": "All pods healthy" },
        { "name": "recent_warnings", "status": "ok", "detail": "No concerning warnings" }
      ]
    }
  ]
}
```

### 404 Response

```json
{ "error": "Incident abc12345 not found" }
```
