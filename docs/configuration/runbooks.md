# Runbook Configuration

Runbooks are YAML playbooks that give agents targeted investigation guidance for known incident patterns. When an alert matches a runbook, agents receive specific queries, investigation steps, and expected findings instead of exploring blindly.

## Built-in Runbooks

Traced ships with six runbooks in the `runbooks/` directory:

| Runbook | Triggers on | Key investigation |
|---------|------------|-------------------|
| `oom_kill` | OOM, OOMKilled, memory limit | Memory usage trends, container limits, restart counts |
| `high_latency` | latency, slow, p99, timeout | Latency percentiles, error rates, CPU throttling |
| `crashloop_backoff` | CrashLoopBackOff, crash, restart | Exit codes, previous container logs, recent deploys |
| `connection_exhaustion` | connection pool, too many connections | Pool usage, active connections, downstream health |
| `disk_pressure` | disk pressure, no space left, DiskPressure | Node disk utilization, PVC usage, eviction events |
| `certificate_expiry` | certificate expir, TLS error, x509 | TLS handshake errors, cert-manager status |

## Creating a Custom Runbook

Create a YAML file in `runbooks/`:

```yaml
name: database_replication_lag
description: >
  Database replication lag is increasing, causing stale reads
  and potential data inconsistency.
severity_hint: high
tags: [database, replication, postgres, mysql]

trigger_patterns:
  - "replication.*lag"
  - "replica.*behind"
  - "slave.*delay"
  - "standby.*lag"

agents:
  observability:
    investigation_steps:
      - Check replication lag metrics over the last hour
      - Look for correlation with write throughput spikes
      - Check replica CPU and IO utilization
    queries:
      - query: "pg_replication_lag_seconds"
        description: "PostgreSQL replication lag in seconds"
      - query: "mysql_slave_seconds_behind_master"
        description: "MySQL replica delay"
    expected_findings:
      - Replication lag increasing over time
      - Possible write throughput spike on primary

  kubernetes:
    investigation_steps:
      - Check database pod resource utilization
      - Look for pod restarts on replica instances
    expected_findings:
      - Replica pod under CPU or IO pressure

  change_detection:
    investigation_steps:
      - Check for recent schema migrations or bulk data loads
    expected_findings:
      - Recent deployment with migration

suggested_remediations:
  - description: "Restart the lagging replica to re-sync"
    command: "kubectl delete pod $REPLICA_POD -n $NS"
    safety_tier: medium_risk
    blast_radius: "Replica unavailable during restart; reads may be affected"
  - description: "Scale up read replicas to distribute load"
    safety_tier: low_risk
    blast_radius: "Adds capacity; no disruption"
```

## Runbook Format Reference

### Top-level fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier (snake_case) |
| `description` | No | Human-readable description |
| `trigger_patterns` | Yes | List of regex patterns to match against alert messages (case-insensitive) |
| `severity_hint` | No | Default severity (`critical`, `high`, `medium`, `low`) |
| `tags` | No | Tags for organization |
| `agents` | No | Per-agent investigation guidance |
| `suggested_remediations` | No | Known fixes with safety tiers |

### Agent playbook fields

Each entry under `agents` (keyed by agent name: `observability`, `kubernetes`, `change_detection`):

| Field | Description |
|-------|-------------|
| `investigation_steps` | Ordered list of steps for the agent to follow |
| `queries` | Suggested queries with `query` and `description` |
| `expected_findings` | What the agent should look for |

### Remediation fields

| Field | Description |
|-------|-------------|
| `description` | What the action does |
| `command` | kubectl or API command (with `$VARIABLES` for placeholders) |
| `safety_tier` | `read_only`, `low_risk`, `medium_risk`, `high_risk`, `blocked` |
| `blast_radius` | What could go wrong |

## How Matching Works

When an alert comes in, Traced checks every runbook's `trigger_patterns` against the alert message. Patterns are regular expressions matched case-insensitively. The runbook with the most patterns matched wins (highest specificity).

If no runbook matches, agents still investigate — they just use their general-purpose system prompts without runbook-specific guidance.
