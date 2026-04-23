# Webhook API

Traced accepts alerts via three webhook endpoints on the Cloud service (port 8322).

## PagerDuty

**`POST /webhooks/pagerduty`**

Accepts PagerDuty v2 and v3 webhook payloads. Only processes `incident.trigger` and `incident.acknowledge` events.

```bash
curl -X POST http://localhost:8322/webhooks/pagerduty \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{
      "event": "incident.trigger",
      "incident": {
        "id": "PD001",
        "title": "High error rate on payment-service",
        "urgency": "high",
        "service": {"name": "payment-service"}
      }
    }]
  }'
```

**Response (202):**
```json
{
  "status": "accepted",
  "incidents": ["abc12345"],
  "deduplicated": []
}
```

## Alertmanager

**`POST /webhooks/alertmanager`**

Accepts Alertmanager group notification payloads. Only processes alerts with `status: firing`.

```bash
curl -X POST http://localhost:8322/webhooks/alertmanager \
  -H "Content-Type: application/json" \
  -d '{
    "alerts": [{
      "status": "firing",
      "labels": {
        "alertname": "HighErrorRate",
        "severity": "critical",
        "namespace": "payments",
        "service": "payment-service"
      },
      "annotations": {
        "summary": "Error rate above 5% for payment-service"
      },
      "fingerprint": "abc123"
    }],
    "groupLabels": {"alertname": "HighErrorRate"}
  }'
```

## Generic

**`POST /webhooks/generic`**

Accepts a simple JSON payload from any custom system.

```bash
curl -X POST http://localhost:8322/webhooks/generic \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Description of the incident",
    "source": "your-monitoring-system",
    "service": "affected-service",
    "severity": "high",
    "namespace": "production"
  }'
```

| Field | Required | Description |
|-------|----------|-------------|
| `message` | Yes | Incident description |
| `source` | No | Source system name (default: "generic") |
| `service` | No | Affected service name |
| `severity` | No | Severity level (default: "high") |

## Response Codes

| Code | Meaning |
|------|---------|
| `202 Accepted` | Alert accepted, investigation dispatched |
| `200 OK` | Payload received but no actionable events (e.g., PagerDuty resolve, Alertmanager resolved) |
| `200 Deduplicated` | Same alert already being investigated (within cooldown window) |

## Deduplication

Traced deduplicates alerts using a fingerprint based on source + service + alert name. If the same alert fires again within 15 minutes (configurable), it's acknowledged but not investigated again. The response includes a `deduplicated` array showing which incident IDs were suppressed.
