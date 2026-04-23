# RBAC Configuration

Traced uses role-based access control to determine who can approve remediation actions in Slack.

## Roles

| Role | Permissions |
|------|-------------|
| **viewer** | Can see investigations in the dashboard. Cannot approve any actions. |
| **responder** | Can approve `read_only` and `low_risk` actions (pod restart, scale up). |
| **admin** | Can approve up to `high_risk` actions (rollback, drain node). |

`blocked` actions are never approvable regardless of role.

## Configuration

Create a `rbac.yaml` file:

```yaml
# Default role for users not explicitly listed
default_role: responder

# Map Slack user IDs to roles
users:
  U0SRE_LEAD1: admin
  U0SRE_LEAD2: admin
  U0ONCALL_01: responder
  U0ONCALL_02: responder
  U0MANAGER1: viewer
```

!!! tip "Finding Slack User IDs"
    Click a user's profile in Slack → click the **⋯** menu → **Copy member ID**

## Deploying the Config

### Docker Compose

Mount the file into the cloud container:

```yaml
# docker-compose.yml
cloud:
  volumes:
    - ./rbac.yaml:/app/rbac.yaml
```

### Kubernetes

Create a ConfigMap and mount it:

```bash
kubectl create configmap traced-rbac --from-file=rbac.yaml -n traced
```

Add to the Cloud deployment:

```yaml
volumes:
  - name: rbac-config
    configMap:
      name: traced-rbac
volumeMounts:
  - name: rbac-config
    mountPath: /app/rbac.yaml
    subPath: rbac.yaml
```

## Default Behavior

If no `rbac.yaml` exists, all users get the `responder` role by default. This means everyone can approve low-risk actions, but medium and high-risk actions will be denied until an admin is configured.
