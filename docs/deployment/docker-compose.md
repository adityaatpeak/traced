# Docker Compose Deployment

The simplest way to run Traced locally or in a single-server environment.

## Quick Start

```bash
cp .env.example .env
# Edit .env with your API keys
docker-compose up --build
```

## Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `collector` | `tracedai-collector` | 8321 | Queries Prometheus, ES, K8s |
| `cloud` | `tracedai-cloud` | 8322 | AI agents, webhooks, dashboard |
| `prometheus` | `prom/prometheus:v2.53.0` | 9090 | Dev metrics backend |
| `elasticsearch` | `elasticsearch:8.17.0` | 9200 | Dev log backend |

## Connecting to Your Real Backends

To point the collector at your existing Prometheus and Elasticsearch instead of the dev containers, update your `.env`:

```bash
PROMETHEUS_URL=http://your-prometheus:9090
ELASTICSEARCH_HOSTS=http://your-elasticsearch:9200
```

Then remove the `prometheus` and `elasticsearch` services from `docker-compose.yml`, or simply don't use them.

## Volumes

| Volume | Purpose |
|--------|---------|
| `cloud-data` | Persists the SQLite incident database across restarts |

## Health Checks

The cloud service includes a Docker health check that polls `/health` every 30 seconds.

```bash
docker-compose ps   # Shows health status
```
