# Customer Setup Guide

> Get Traced connected to your cluster in under 20 minutes.

This guide walks you through deploying the Traced Collector into your Kubernetes cluster. The Collector is a lightweight service that securely connects your observability stack (Prometheus, Elasticsearch, Kubernetes API) to the Traced platform. All AI processing happens on our side — your data is scrubbed of sensitive information before it leaves your cluster.

---

## Table of Contents

1. [What Gets Deployed](#1-what-gets-deployed)
2. [Prerequisites](#2-prerequisites)
3. [Create a Namespace and Service Account](#3-create-a-namespace-and-service-account)
4. [Install the Collector](#4-install-the-collector)
5. [Connect Prometheus](#5-connect-prometheus)
6. [Connect Elasticsearch](#6-connect-elasticsearch)
7. [Enable Kubernetes Access](#7-enable-kubernetes-access)
8. [Connect CloudWatch (Optional)](#8-connect-cloudwatch-optional)
9. [Verify the Installation](#9-verify-the-installation)
10. [Configure Alert Forwarding](#10-configure-alert-forwarding)
11. [Security & Data Privacy](#11-security-data-privacy)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. What Gets Deployed

Only **one component** is deployed in your cluster: the **Traced Collector**.

```
┌─── Your Cluster ──────────────────────────────────┐
│                                                     │
│   ┌──────────────────┐                              │
│   │  Traced Collector │──→ Prometheus                │
│   │  (lightweight     │──→ Elasticsearch             │
│   │   read-only       │──→ Kubernetes API            │
│   │   proxy)          │──→ CloudWatch (optional)     │
│   └────────┬─────────┘                              │
│            │                                         │
│            │ Outbound HTTPS (API key auth)           │
│            │ Sensitive data scrubbed before sending  │
└────────────┼────────────────────────────────────────┘
             │
             ▼
    ┌─── Traced Cloud (managed by us) ───┐
    │  AI Agents · Slack Bot · Dashboard  │
    └─────────────────────────────────────┘
```

The Collector:

- **Reads only** — it queries your backends but never modifies them (unless remediation actions are approved by your team in Slack)
- **Scrubs sensitive data** — emails, IPs, credentials, tokens, and connection strings are redacted before any data leaves your cluster
- **Authenticates all requests** — only the Traced Cloud service can communicate with it, via a shared API key
- **Lightweight** — typically uses ~100MB RAM and minimal CPU

---

## 2. Prerequisites

| Requirement | Details |
|-------------|---------|
| Kubernetes cluster | 1.24+ (EKS, GKE, AKS, or self-managed) |
| Helm 3 | For installing the Collector chart |
| kubectl | Configured with access to the target cluster |
| Prometheus | Running and accessible within the cluster |
| Elasticsearch | Running and accessible within the cluster (for log search) |
| Collector API key | Provided by the Traced team during onboarding |

!!! note "No inbound traffic needed"
    The Collector only makes **outbound** connections to the Traced Cloud platform. It does not need an ingress, load balancer, or public URL.

---

## 3. Create a Namespace and Service Account

### Create the namespace

```bash
kubectl create namespace traced
```

### Set up RBAC (read-only)

The Collector needs read access to your cluster resources for investigation. Apply the following:

```yaml
# traced-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traced-collector
  namespace: traced
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: traced-collector-readonly
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "events", "configmaps", "nodes", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: traced-collector-readonly
subjects:
  - kind: ServiceAccount
    name: traced-collector
    namespace: traced
roleRef:
  kind: ClusterRole
  name: traced-collector-readonly
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f traced-rbac.yaml
```

### Optional: Enable Remediation Actions

If you want Traced to execute approved fixes (pod restart, rollback, scale), add these additional permissions:

```yaml
# traced-remediation-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: traced-collector-remediation
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete"]             # Pod restart (controller recreates)
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["patch"]              # Scale replicas, trigger rollback
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: traced-collector-remediation
subjects:
  - kind: ServiceAccount
    name: traced-collector
    namespace: traced
roleRef:
  kind: ClusterRole
  name: traced-collector-remediation
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f traced-remediation-rbac.yaml
```

!!! info "Safety guarantee"
    Even with remediation permissions, Traced **never executes actions without human approval** in Slack. Every action is classified by risk tier and requires your team member to click "Approve" before execution.

---

## 4. Install the Collector

You'll receive a **Collector API key** and a **Cloud endpoint URL** from the Traced team during onboarding.

```bash
helm install traced-collector charts/collector/ \
  --namespace traced \
  --set collectorApiKey="<your-collector-api-key>" \
  --set cloudEndpoint="https://cloud.traced.ai" \
  --set prometheus.url="http://prometheus-server.monitoring.svc:9090" \
  --set elasticsearch.hosts="http://elasticsearch-master.logging.svc:9200"
```

### Verify the pod is running

```bash
kubectl get pods -n traced

# Expected:
# NAME                                  READY   STATUS    RESTARTS   AGE
# traced-collector-7b8f4c6d9-x2k1m     1/1     Running   0          30s
```

---

## 5. Connect Prometheus

Find your Prometheus service URL:

```bash
kubectl get svc -A | grep prometheus
```

### Common Prometheus URLs

| Setup | URL |
|-------|-----|
| kube-prometheus-stack | `http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090` |
| Prometheus Operator | `http://prometheus-operated.monitoring.svc:9090` |
| Standalone Helm chart | `http://prometheus-server.monitoring.svc:9090` |
| Thanos Query | `http://thanos-query.monitoring.svc:9090` |
| Victoria Metrics | `http://victoria-metrics.monitoring.svc:8428` |
| Amazon Managed Prometheus | `https://aps-workspaces.<region>.amazonaws.com/workspaces/<ws-id>` |

### Test connectivity

```bash
kubectl exec -n traced deploy/traced-collector -- \
  curl -s "http://prometheus-server.monitoring.svc:9090/api/v1/query?query=up" | head -c 200
```

You should see `"status":"success"` in the response.

---

## 6. Connect Elasticsearch

Find your Elasticsearch service:

```bash
kubectl get svc -A | grep -i elastic
```

### Common Elasticsearch URLs

| Setup | URL |
|-------|-----|
| ECK (Elastic Operator) | `https://elasticsearch-es-http.logging.svc:9200` |
| Helm chart | `http://elasticsearch-master.logging.svc:9200` |
| OpenSearch | `https://opensearch-cluster-master.logging.svc:9200` |
| Amazon OpenSearch | `https://<domain>.<region>.es.amazonaws.com` |

If Elasticsearch requires authentication, provide it during installation:

```bash
--set elasticsearch.apiKey="<your-es-api-key>"
```

### Test connectivity

```bash
kubectl exec -n traced deploy/traced-collector -- \
  curl -s "http://elasticsearch-master.logging.svc:9200/_cluster/health" | head -c 200
```

---

## 7. Enable Kubernetes Access

No additional configuration needed — the Collector automatically uses the ServiceAccount created in Step 3 to query the Kubernetes API.

The Collector reads:

| Resource | Purpose |
|----------|---------|
| Pods & logs | Check pod health, restart counts, crash logs |
| Events | Find OOM kills, probe failures, scheduling issues |
| Deployments | Check rollout status, replica counts, recent changes |
| ConfigMaps | Detect configuration changes |
| Nodes | Check node health and resource pressure |

### Verify access

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:traced:traced-collector \
  --all-namespaces
# Should return: yes
```

---

## 8. Connect CloudWatch (Optional)

If you use AWS CloudWatch, enable it during installation:

```bash
--set cloudwatch.enabled=true \
--set cloudwatch.region="us-east-1"
```

Required IAM permissions: `cloudwatch:GetMetricData`, `logs:FilterLogEvents`, `logs:GetLogEvents`

**Recommended:** Use [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) on EKS to avoid storing AWS credentials.

---

## 9. Verify the Installation

Run the health check:

```bash
kubectl exec -n traced deploy/traced-collector -- curl -s localhost:8321/health | python3 -m json.tool
```

Expected:

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

All backends you configured should show `connected` or `configured`. Share this output with the Traced team to confirm onboarding is complete.

---

## 10. Configure Alert Forwarding

To trigger automatic investigations, configure your alerting tools to send webhooks to Traced.

### From Alertmanager

Add a webhook receiver to your Alertmanager config:

```yaml
receivers:
  - name: traced
    webhook_configs:
      - url: https://cloud.traced.ai/webhooks/alertmanager
        send_resolved: false

route:
  routes:
    - match:
        severity: critical
      receiver: traced
      continue: true    # Keep your existing alerting pipeline
```

!!! tip
    Use `continue: true` so alerts still go to PagerDuty, Slack, etc. Traced is additive — it investigates alongside your existing process.

### From PagerDuty

1. Go to **Services** → your service → **Integrations**
2. Add a **Generic Webhooks (v3)** integration
3. Set URL to: `https://cloud.traced.ai/webhooks/pagerduty`
4. Select events: `incident.triggered`

### From custom systems

```bash
curl -X POST https://cloud.traced.ai/webhooks/generic \
  -H "Content-Type: application/json" \
  -d '{"message": "High error rate on payment-service", "source": "your-monitoring", "service": "payment-service", "severity": "critical"}'
```

---

## 11. Security & Data Privacy

### What data leaves your cluster?

When the Traced platform investigates an incident, it requests data through the Collector. Before responding, the Collector automatically scrubs:

| Data type | Action |
|-----------|--------|
| Email addresses | Redacted to `[EMAIL_REDACTED]` |
| IP addresses | Redacted to `[IP_REDACTED]` |
| Credit card numbers | Redacted |
| JWT and bearer tokens | Redacted |
| AWS access keys | Redacted |
| Passwords and connection strings | Redacted |

### What the Collector does NOT do

- Does **not** store any data locally
- Does **not** send data proactively — only responds to requests
- Does **not** modify infrastructure (unless remediation is approved by your team)
- Does **not** require inbound network access

### Network requirements

| Direction | Destination | Port | Purpose |
|-----------|-------------|------|---------|
| Outbound | `cloud.traced.ai` | 443 | Communication with Traced platform |
| Internal | Prometheus | 9090 | Query metrics |
| Internal | Elasticsearch | 9200 | Search logs |
| Internal | K8s API | 443 | Inspect cluster state |

---

## 12. Troubleshooting

### Collector pod not starting

```bash
kubectl logs -n traced deploy/traced-collector --tail=50
```

| Symptom | Fix |
|---------|-----|
| `ImagePullBackOff` | Check image repository access and pull secrets |
| `CrashLoopBackOff` | Check logs — usually a wrong Prometheus/ES URL |
| `Pending` | Insufficient node resources — check `kubectl describe pod` |

### A service shows "error" in the health check

Verify the URL and test connectivity directly:

```bash
# For Prometheus
kubectl exec -n traced deploy/traced-collector -- curl -v "http://<prometheus-url>/-/healthy"

# For Elasticsearch
kubectl exec -n traced deploy/traced-collector -- curl -v "http://<elasticsearch-url>/_cluster/health"
```

Common causes: wrong URL, NetworkPolicy blocking traffic, authentication required.

### Kubernetes access denied

```bash
kubectl auth can-i list pods --as=system:serviceaccount:traced:traced-collector --all-namespaces
```

If this returns `no`, re-apply the RBAC from Step 3.

### Need help?

Contact the Traced team with:

- Output of `kubectl logs -n traced deploy/traced-collector --tail=100`
- Output of the health check endpoint
- Your Kubernetes version (`kubectl version --short`)
