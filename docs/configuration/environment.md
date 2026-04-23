# Environment Variables

All Traced configuration is done via environment variables. Both services use `pydantic-settings` with `.env` file support.

## Cloud Service

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `TRACED_DEBUG` | `false` | Enable debug mode |
| `TRACED_LOG_LEVEL` | `INFO` | Logging level (DEBUG, INFO, WARNING, ERROR) |
| `TRACED_AGENT_TIMEOUT_SECONDS` | `30` | Max time per agent investigation |
| `TRACED_MAX_PARALLEL_AGENTS` | `5` | Max agents running simultaneously |
| `TRACED_DB_PATH` | `data/traced_incidents.db` | SQLite database file path |

### Anthropic (Claude)

| Variable | Default | Description |
|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | *required* | Your Anthropic API key |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model to use |
| `ANTHROPIC_MAX_TOKENS` | `4096` | Max tokens per LLM response |

### Slack

| Variable | Default | Description |
|----------|---------|-------------|
| `SLACK_BOT_TOKEN` | — | Bot User OAuth Token (`xoxb-...`) |
| `SLACK_APP_TOKEN` | — | App-Level Token for Socket Mode (`xapp-...`) |
| `SLACK_SIGNING_SECRET` | — | Signing secret (optional for Socket Mode) |

### Collector Connection

| Variable | Default | Description |
|----------|---------|-------------|
| `COLLECTOR_URL` | `http://localhost:8321` | Collector service URL |
| `COLLECTOR_API_KEY` | `traced-dev-key` | API key for collector auth |
| `COLLECTOR_TIMEOUT_SECONDS` | `25.0` | HTTP timeout for collector calls |

## Collector Service

### Core Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `COLLECTOR_HOST` | `0.0.0.0` | Bind address |
| `COLLECTOR_PORT` | `8321` | Listen port |
| `COLLECTOR_API_KEY` | `traced-dev-key` | API key for authentication |
| `COLLECTOR_LOG_LEVEL` | `INFO` | Logging level |

### Prometheus

| Variable | Default | Description |
|----------|---------|-------------|
| `PROMETHEUS_URL` | `http://localhost:9090` | Prometheus server URL |
| `PROMETHEUS_AUTH_TOKEN` | — | Bearer token for auth (optional) |

### Elasticsearch

| Variable | Default | Description |
|----------|---------|-------------|
| `ELASTICSEARCH_HOSTS` | `http://localhost:9200` | Elasticsearch URL(s) |
| `ELASTICSEARCH_API_KEY` | — | API key for auth (optional) |
| `ELASTICSEARCH_INDEX_PATTERN` | `logs-*` | Default index pattern for log search |

### Kubernetes

| Variable | Default | Description |
|----------|---------|-------------|
| `KUBERNETES_IN_CLUSTER` | `true` | Use in-cluster config (ServiceAccount) |
| `KUBE_CONFIG_PATH` | `~/.kube/config` | Kubeconfig path (when not in-cluster) |

### CloudWatch (Optional)

| Variable | Default | Description |
|----------|---------|-------------|
| `CLOUDWATCH_ENABLED` | `false` | Enable CloudWatch integration |
| `AWS_REGION` | `us-east-1` | AWS region |
| `AWS_ACCESS_KEY_ID` | — | AWS credentials (or use IRSA) |
| `AWS_SECRET_ACCESS_KEY` | — | AWS credentials |
