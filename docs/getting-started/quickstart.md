# Quick Start

Get Traced investigating incidents in your cluster in 3 steps.

## Step 1: Install the Collector

```bash
kubectl create namespace traced
kubectl apply -f https://traced.ai/install/rbac.yaml

helm install traced-collector traced/collector \
  --namespace traced \
  --set collectorApiKey="<key-from-traced-team>" \
  --set prometheus.url="http://prometheus-server.monitoring.svc:9090" \
  --set elasticsearch.hosts="http://elasticsearch-master.logging.svc:9200"
```

## Step 2: Verify

```bash
kubectl exec -n traced deploy/traced-collector -- curl -s localhost:8321/health
```

You should see `"prometheus": "connected"` and `"elasticsearch": "connected"`.

## Step 3: Trigger your first investigation

In Slack, tag the bot:

```
@Traced payment-service is showing high latency — p99 above 3 seconds
```

Or via webhook:

```bash
curl -X POST https://cloud.traced.ai/webhooks/generic \
  -H "Content-Type: application/json" \
  -d '{"message": "High latency on payment-service", "service": "payment-service", "severity": "high"}'
```

Traced will dispatch three specialist agents to investigate in parallel, synthesize a root cause, and post findings to Slack within 30 seconds.

## What's Next

- [Full Setup Guide](setup-guide.md) — Detailed walkthrough with RBAC, connectivity testing, and troubleshooting
- [How Investigations Work](first-investigation.md) — What happens under the hood
- [Configure Alert Forwarding](setup-guide.md#10-configure-alert-forwarding) — Auto-trigger from PagerDuty or Alertmanager
