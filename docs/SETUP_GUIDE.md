# Traced — Customer Setup Guide

> Get Traced running in your Kubernetes cluster in under 30 minutes.

This guide walks you through setting up Traced to investigate production incidents in your environment. By the end, you'll have Traced connected to your observability stack, listening for alerts, and reporting findings to Slack.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Create a Slack App](#2-create-a-slack-app)
3. [Prepare Your Cluster](#3-prepare-your-cluster)
4. [Install Traced with Helm](#4-install-traced-with-helm)
5. [Connect Your Observability Stack](#5-connect-your-observability-stack)
6. [Configure Alert Webhooks](#6-configure-alert-webhooks)
7. [Configure RBAC for Approvals](#7-configure-rbac-for-approvals)
8. [Verify the Installation](#8-verify-the-installation)
9. [Send Your First Test Alert](#9-send-your-first-test-alert)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Prerequisites

Before starting, make sure you have:

| Requirement | Version | Notes |
|-------------|---------|-------|
| Kubernetes cluster | 1.24+ | EKS, GKE, AKS, or self-managed |
| Helm | 3.x | For installing the Traced chart |
| kubectl | Configured | With access to the target cluster |
| Prometheus | 2.x+ | Already running in your cluster |
| Elasticsearch | 7.x or 8.x | For log search (Kibana/ELK stack) |
| Anthropic API key | — | Sign up at [console.anthropic.com](https://console.anthropic.com) |
| Slack workspace | — | With permission to install apps |

**Optional:** AWS credentials if you use CloudWatch for monitoring.

---

## 2. Create a Slack App

Traced uses Slack Socket Mode, so no public URL or ingress is needed for the bot.

### Step 1: Create the app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App**
2. Choose **From scratch**
3. Name it `Traced` and select your workspace
4. Click **Create App**

### Step 2: Enable Socket Mode

1. In the left sidebar, go to **Socket Mode**
2. Toggle **Enable Socket Mode** to on
3. Create an app-level token with the scope `connections:write`
4. Name it `traced-socket` and click **Generate**
5. **Save this token** — it starts with `xapp-` (this is your `SLACK_APP_TOKEN`)

### Step 3: Configure Bot Permissions

1. Go to **OAuth & Permissions** in the sidebar
2. Under **Bot Token Scopes**, add these scopes:

| Scope | Why |
|-------|-----|
| `app_mentions:read` | Detect when users tag @traced |
| `chat:write` | Post investigation results and approval buttons |
| `im:history` | Handle direct messages |
| `im:read` | Read DMs sent to the bot |
| `users:read` | Look up who approved actions |

### Step 4: Enable Events

1. Go to **Event Subscriptions** in the sidebar
2. Toggle **Enable Events** to on
3. Under **Subscribe to bot events**, add:
   - `app_mention`
   - `message.im`
4. Click **Save Changes**

### Step 5: Enable Interactivity

1. Go to **Interactivity & Shortcuts** in the sidebar
2. Toggle **Interactivity** to on
3. You don't need a Request URL (Socket Mode handles it)
4. Click **Save Changes**

### Step 6: Install to Workspace

1. Go to **Install App** in the sidebar
2. Click **Install to Workspace** and authorize
3. **Save the Bot User OAuth Token** — it starts with `xoxb-` (this is your `SLACK_BOT_TOKEN`)

### Step 7: Invite the bot to a channel

In Slack, go to the channel where you want incident reports and type:
```
/invite @Traced
```

---

## 3. Prepare Your Cluster

Traced runs two services in your cluster: the **Collector** (queries your infra) and the **Cloud** service (runs AI agents and Slack bot). The Collector needs read access to your Kubernetes API, Prometheus, and Elasticsearch.

### Create the namespace

```bash
kubectl create namespace traced
```

### Create a ServiceAccount with read access

The Collector needs to read pods, events, deployments, configmaps, and nodes. Create a ClusterRole:

```yaml
# traced-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: traced-collector
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events", "configmaps", "nodes", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
---
# For remediation actions (optional — only if you want Traced to execute fixes)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: traced-remediation
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete"]             # Pod restart
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["patch"]              # Scale, rollback
    # NOTE: Traced will NEVER execute remediation without human approval in Slack
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: traced-collector
subjects:
  - kind: ServiceAccount
    name: traced
    namespace: traced
roleRef:
  kind: ClusterRole
  name: traced-collector
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: traced-remediation
subjects:
  - kind: ServiceAccount
    name: traced
    namespace: traced
roleRef:
  kind: ClusterRole
  name: traced-remediation
  apiGroup: rbac.authorization.k8s.io
```

Apply it:

```bash
kubectl apply -f traced-rbac.yaml
```

> **Security note:** If you only want Traced to investigate (not remediate), skip the `traced-remediation` ClusterRole. Traced will still suggest fixes — it just won't be able to execute them.

---

## 4. Install Traced with Helm

### Add the chart and install

```bash
helm install traced deploy/helm/traced/ \
  --namespace traced \
  --set secrets.anthropicApiKey="sk-ant-api03-..." \
  --set secrets.slackBotToken="xoxb-..." \
  --set secrets.slackAppToken="xapp-..." \
  --set secrets.collectorApiKey="$(openssl rand -hex 16)" \
  --set prometheus.url="http://prometheus-server.monitoring.svc:9090" \
  --set elasticsearch.hosts="http://elasticsearch.logging.svc:9200"
```

> **Tip:** For production, use an external secrets manager (e.g., AWS Secrets Manager, Vault) instead of `--set` for sensitive values.

### Verify pods are running

```bash
kubectl get pods -n traced
```

Expected output:
```
NAME                                READY   STATUS    RESTARTS   AGE
traced-collector-7b8f4c6d9-x2k1m   1/1     Running   0          30s
traced-cloud-5d9f8e7a2-p4n3q       1/1     Running   0          30s
```

---

## 5. Connect Your Observability Stack

### Prometheus

Traced needs HTTP access to the Prometheus query API. Find your Prometheus service:

```bash
kubectl get svc -A | grep prometheus
```

Common URLs by setup:

| Setup | Prometheus URL |
|-------|---------------|
| kube-prometheus-stack | `http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090` |
| Prometheus Operator | `http://prometheus-operated.monitoring.svc:9090` |
| Standalone | `http://prometheus-server.monitoring.svc:9090` |
| Thanos | `http://thanos-query.monitoring.svc:9090` |
| Amazon Managed Prometheus | `https://aps-workspaces.<region>.amazonaws.com/workspaces/<ws-id>` |

**Test connectivity from the collector pod:**

```bash
kubectl exec -n traced deploy/traced-collector -- \
  curl -s http://prometheus-server.monitoring.svc:9090/api/v1/query?query=up | head -c 200
```

### Elasticsearch

Find your Elasticsearch service:

```bash
kubectl get svc -A | grep elastic
```

Common URLs:

| Setup | Elasticsearch URL |
|-------|------------------|
| ECK (Elastic Operator) | `https://elasticsearch-es-http.logging.svc:9200` |
| Helm chart | `http://elasticsearch-master.logging.svc:9200` |
| OpenSearch | `https://opensearch-cluster-master.logging.svc:9200` |
| Amazon OpenSearch | `https://<domain>.<region>.es.amazonaws.com` |

If Elasticsearch requires authentication, set these in your Helm values:

```yaml
elasticsearch:
  hosts: "https://elasticsearch-es-http.logging.svc:9200"
  # For basic auth, set ELASTICSEARCH_API_KEY env var on the collector
```

**Test connectivity:**

```bash
kubectl exec -n traced deploy/traced-collector -- \
  curl -s http://elasticsearch-master.logging.svc:9200/_cluster/health | head -c 200
```

### CloudWatch (Optional)

If you use AWS CloudWatch, provide AWS credentials:

```yaml
cloudwatch:
  enabled: true
  region: "us-east-1"
```

The collector pod needs an IAM role or AWS credentials with `cloudwatch:GetMetricData` and `logs:FilterLogEvents` permissions. The easiest approach is IRSA (IAM Roles for Service Accounts) on EKS.

---

## 6. Configure Alert Webhooks

Traced can receive alerts from PagerDuty, Alertmanager, or any custom system. You need to expose the Cloud service's webhook endpoint.

### Expose webhooks via Ingress

Add to your Helm values:

```yaml
ingress:
  enabled: true
  className: "nginx"   # or alb, traefik, etc.
  annotations:
    # Add your ingress annotations here
  hosts:
    - host: traced.internal.yourcompany.com
      paths:
        - path: /webhooks
          pathType: Prefix
  tls:
    - secretName: traced-tls
      hosts:
        - traced.internal.yourcompany.com
```

Or port-forward for testing:

```bash
kubectl port-forward -n traced svc/traced-cloud 8322:8322
```

### PagerDuty

1. In PagerDuty, go to **Services** → your service → **Integrations**
2. Click **Add Integration** → **Generic Webhooks (v3)**
3. Set the webhook URL to: `https://traced.internal.yourcompany.com/webhooks/pagerduty`
4. Select events: `incident.triggered`, `incident.acknowledged`
5. Save

### Alertmanager

Add a webhook receiver to your Alertmanager config:

```yaml
# alertmanager.yml
receivers:
  - name: traced
    webhook_configs:
      - url: http://traced-cloud.traced.svc:8322/webhooks/alertmanager
        send_resolved: false  # Traced only needs firing alerts

route:
  routes:
    - match:
        severity: critical
      receiver: traced
      continue: true    # Don't stop other receivers (e.g., PagerDuty)
```

> **Tip:** Use `continue: true` so Traced receives the alert alongside your existing notification pipeline. Traced is additive — it investigates, it doesn't replace your alerting.

### Custom / Generic

For any system that can send HTTP webhooks:

```bash
curl -X POST https://traced.internal.yourcompany.com/webhooks/generic \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Description of the incident",
    "source": "your-system-name",
    "service": "affected-service",
    "severity": "high"
  }'
```

---

## 7. Configure RBAC for Approvals

Traced uses a safety tier system for remediation actions. By default, all Slack users are `responder` (can approve low-risk actions). To restrict who can approve high-risk actions:

Create `rbac.yaml` in the Cloud service's working directory:

```yaml
default_role: responder

users:
  # Slack User IDs → roles
  # Find IDs: click a profile → "..." → "Copy member ID"
  U0ADMIN001: admin       # Can approve up to high_risk
  U0ADMIN002: admin
  U0ONCALL01: responder   # Can approve low_risk only
  U0VIEWER01: viewer      # Can see but not approve
```

| Role | Can Approve |
|------|-------------|
| `viewer` | Nothing — read-only access |
| `responder` | `read_only`, `low_risk` (pod restart, scale up) |
| `admin` | Up to `high_risk` (rollback, drain node) |
| — | `blocked` actions are never automated |

Mount this as a ConfigMap:

```bash
kubectl create configmap traced-rbac --from-file=rbac.yaml -n traced
```

And add to the Cloud deployment's volume mounts in your Helm values.

---

## 8. Verify the Installation

### Check service health

```bash
# Collector — shows which backends are connected
kubectl exec -n traced deploy/traced-collector -- curl -s localhost:8321/health | python3 -m json.tool

# Expected:
# {
#   "status": "ok",
#   "services": {
#     "prometheus": "connected",
#     "elasticsearch": "connected",
#     "kubernetes": "configured",
#     "cloudwatch": "not_configured"
#   }
# }

# Cloud — shows Slack connection status
kubectl exec -n traced deploy/traced-cloud -- curl -s localhost:8322/health | python3 -m json.tool
```

### Check Slack connectivity

In your Slack channel, type:
```
@Traced
```

Traced should reply with a help message. If it doesn't respond, check the Cloud pod logs:

```bash
kubectl logs -n traced deploy/traced-cloud | grep slack
```

### Open the dashboard

Port-forward and open in your browser:

```bash
kubectl port-forward -n traced svc/traced-cloud 8322:8322
open http://localhost:8322/dashboard
```

---

## 9. Send Your First Test Alert

### Option A: Via Slack

In your channel, type:

```
@Traced payment-service is showing high latency — p99 above 3 seconds in the payments namespace
```

### Option B: Via webhook

```bash
kubectl exec -n traced deploy/traced-cloud -- curl -s -X POST localhost:8322/webhooks/generic \
  -H "Content-Type: application/json" \
  -d '{"message": "High latency on payment-service p99 > 3s", "source": "test", "service": "payment-service", "severity": "high"}'
```

### What to expect

1. Traced acknowledges the alert (Slack message or 202 response)
2. Three specialist agents start investigating in parallel:
   - **Observability Agent** queries Prometheus metrics and Elasticsearch logs
   - **Kubernetes Agent** checks pods, events, deployments
   - **Change Agent** looks for recent deployments and config changes
3. Claude synthesizes findings into a root cause with confidence score
4. If remediation is suggested, Slack shows approve/reject buttons
5. Results are persisted and visible on the dashboard

A typical investigation completes in 15-30 seconds.

---

## 10. Troubleshooting

### Collector can't reach Prometheus

```
"prometheus": "error"
```

**Fix:** Check the Prometheus URL and network policies. The collector pod must be able to reach the Prometheus service on port 9090. Test with:

```bash
kubectl exec -n traced deploy/traced-collector -- curl -v http://prometheus-server.monitoring.svc:9090/-/healthy
```

### Collector can't reach Elasticsearch

**Fix:** Same as above — verify the URL and that the collector namespace is allowed by any NetworkPolicies. For HTTPS with self-signed certs, you may need to disable TLS verification in the collector settings.

### Cloud service not starting

Check logs:

```bash
kubectl logs -n traced deploy/traced-cloud --tail=50
```

Common issues:
- `ANTHROPIC_API_KEY` not set or invalid → Sign up at console.anthropic.com
- `credit balance too low` → Add credits to your Anthropic account
- Missing `SLACK_BOT_TOKEN` or `SLACK_APP_TOKEN` → The cloud service starts without Slack, but the bot won't work

### Slack bot not responding

1. Verify Socket Mode is enabled in your Slack app settings
2. Verify the bot is invited to the channel (`/invite @Traced`)
3. Check that `app_mention` event is subscribed
4. Check Cloud pod logs for `slack_bot_not_started` errors

### Investigation produces no findings

This usually means the agents couldn't find relevant data in your Prometheus/ES:
- Ensure your services export metrics that Prometheus scrapes
- Ensure application logs are indexed in Elasticsearch
- Check the incident detail via the API — agent errors show exactly what went wrong:

```bash
curl http://localhost:8322/api/v1/incidents/<incident_id> | python3 -m json.tool
```

### Webhook returns 200 "ignored" instead of 202

The payload doesn't contain actionable events:
- **PagerDuty:** Only `incident.trigger` and `incident.acknowledge` are processed (not `resolve`)
- **Alertmanager:** Only `status: firing` alerts are processed (not `resolved`)

### Deduplication is suppressing alerts

Same alert within 15 minutes is deduplicated by default. To adjust:

```yaml
# In your deployment environment
TRACED_DEDUP_COOLDOWN_SECONDS=300  # 5 minutes instead of 15
```

---

## What's Next

- **Custom runbooks:** Add YAML runbooks in `runbooks/` for your specific incident patterns
- **Tune safety tiers:** Adjust which actions require which approval level
- **Connect PagerDuty/Alertmanager:** Route production alerts to Traced automatically
- **Monitor costs:** Check LLM token usage in the cloud service logs (`llm_response` events)

For questions or issues, reach out to the Traced team.
