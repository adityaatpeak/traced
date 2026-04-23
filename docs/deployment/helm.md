# Helm / Kubernetes Deployment

Deploy Traced to your Kubernetes cluster using the included Helm chart.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.x
- Prometheus and Elasticsearch accessible from the cluster

## Install

```bash
helm install traced deploy/helm/traced/ \
  --namespace traced --create-namespace \
  --set secrets.anthropicApiKey="sk-ant-..." \
  --set secrets.slackBotToken="xoxb-..." \
  --set secrets.slackAppToken="xapp-..." \
  --set secrets.collectorApiKey="$(openssl rand -hex 16)" \
  --set prometheus.url="http://prometheus-server.monitoring.svc:9090" \
  --set elasticsearch.hosts="http://elasticsearch-master.logging.svc:9200"
```

## Chart Values

See `deploy/helm/traced/values.yaml` for all options. Key values:

### Images

```yaml
collector:
  image:
    repository: traced/collector
    tag: "0.1.0"
  replicas: 1
  resources:
    requests: { cpu: 100m, memory: 256Mi }
    limits: { cpu: 500m, memory: 512Mi }

cloud:
  image:
    repository: traced/cloud
    tag: "0.1.0"
  replicas: 1
  resources:
    requests: { cpu: 200m, memory: 512Mi }
    limits: { cpu: "1", memory: 1Gi }
```

### Backend Connections

```yaml
prometheus:
  url: "http://prometheus-server.monitoring.svc:9090"
elasticsearch:
  hosts: "http://elasticsearch-master.logging.svc:9200"
kubernetes:
  inCluster: true
cloudwatch:
  enabled: false
```

### Ingress (for webhooks)

```yaml
ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: traced.internal.example.com
      paths:
        - path: /webhooks
          pathType: Prefix
  tls:
    - secretName: traced-tls
      hosts:
        - traced.internal.example.com
```

## Upgrade

```bash
helm upgrade traced deploy/helm/traced/ --namespace traced --reuse-values \
  --set cloud.image.tag="0.2.0" \
  --set collector.image.tag="0.2.0"
```

## Uninstall

```bash
helm uninstall traced --namespace traced
```
