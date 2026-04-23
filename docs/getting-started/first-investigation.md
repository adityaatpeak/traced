# Your First Investigation

This guide walks through what happens when Traced investigates an incident, so you know what to expect and how to interpret the results.

## Triggering an Investigation

There are three ways to start an investigation:

=== "Slack"
    Tag the bot in any channel it's been invited to:
    ```
    @Traced payment-service is returning 503s — error rate above 10% in the payments namespace
    ```

=== "Webhook"
    Send an HTTP POST to the webhook endpoint:
    ```bash
    curl -X POST http://localhost:8322/webhooks/generic \
      -H "Content-Type: application/json" \
      -d '{"message": "503 errors on payment-service", "source": "monitoring", "service": "payment-service", "severity": "high"}'
    ```

=== "Alert Integration"
    Configure PagerDuty or Alertmanager to forward alerts automatically. See [Webhook Configuration](../api/webhooks.md).

## What Happens Next

### Phase 1: Agent Investigation (5-15 seconds)

Three specialist agents run in parallel:

| Agent | What it does | Tools it uses |
|-------|-------------|---------------|
| **Observability** | Queries error rates, latency, resource metrics | Prometheus (PromQL), Elasticsearch (log search) |
| **Kubernetes** | Checks pod health, events, deployments | K8s API (pods, events, describe) |
| **Change Detection** | Looks for recent deploys and config changes | K8s API (deployments, configmaps) |

Each agent uses Claude in a tool-calling loop — it decides what to query, analyzes results, and reports findings.

!!! info "Runbook matching"
    If the alert matches a known pattern (like "OOMKilled" or "CrashLoopBackOff"), a runbook is injected into the agents' prompts with specific queries and investigation steps. This makes investigations faster and more targeted.

### Phase 2: Root Cause Synthesis (3-5 seconds)

Claude analyzes all agent findings together and produces:

- **Root cause summary** — What went wrong and why
- **Confidence score** — 0-100%, reflecting how certain the analysis is
- **Evidence chain** — Ordered list of supporting evidence
- **Timeline** — Chronological sequence of events
- **Remediation actions** — Suggested fixes with safety tiers

### Phase 3: Approval & Remediation

If remediation actions are suggested, they appear in Slack with approve/reject buttons:

- :green_circle: **Low risk** (pod restart, scale up) — Any responder can approve
- :yellow_circle: **Medium risk** (rollback) — Requires explicit approval
- :red_circle: **High risk** (drain node) — Admin approval required
- :no_entry_sign: **Blocked** (delete namespace) — Manual only, no buttons

### Phase 4: Post-Fix Verification

After an approved action executes, Traced automatically:

1. Waits for K8s to reconcile (5-10 seconds)
2. Checks pod health — no CrashLoopBackOff, no high restart counts
3. Checks for warning events
4. Posts a resolution summary to the Slack thread

## Reading the Results

### On the Dashboard

Open `http://localhost:8322/dashboard` to see:

- **Incident list** — All investigations with status, source, and confidence
- **Detail panel** — Click any incident to see agent results, findings, actions, and verifications

### Via API

```bash
# List incidents
curl http://localhost:8322/api/v1/incidents | python3 -m json.tool

# Full detail for a specific incident
curl http://localhost:8322/api/v1/incidents/<incident_id> | python3 -m json.tool
```

The detail response includes:

- `agent_results` — Per-agent status, findings, and execution time
- `actions` — Suggested remediations with safety tier and approval status
- `verifications` — Post-fix health check results

## Tips for Better Results

- **Be specific in alerts** — "payment-service p99 latency above 3s in namespace payments" gives agents more to work with than "things are slow"
- **Ensure metrics exist** — Agents can only find what's instrumented. Standard K8s metrics (pod status, events) always work; application metrics (request rate, error rate) need Prometheus scraping
- **Add custom runbooks** — For your specific services, add YAML runbooks with suggested PromQL queries. See [Runbook Configuration](../configuration/runbooks.md)
