# Kong Konnect – GitHub Actions Deployment Pipeline

Declarative GitOps pipeline for managing Kong Konnect configuration using **decK**.

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy-kong.yml   # CI/CD pipeline
└── kong/
    └── kong.yaml             # Declarative Kong config (services, routes, plugins)
```

---

## Pipeline Overview

| Trigger | Jobs Run | Purpose |
|---|---|---|
| Pull Request to `main` | `validate` → `diff` | Lint config + post diff as PR comment |
| Push / merge to `main` | `validate` → `deploy` | Sync config to Konnect + verify no drift |
| Manual (`workflow_dispatch`) | `validate` → `deploy` | On-demand deploy |

The pipeline only triggers when files under `kong/` change.

---

## Required GitHub Secrets & Variables

Go to **Settings → Secrets and variables → Actions** and add:

### Secrets (encrypted)
| Name | Description |
|---|---|
| `KONNECT_TOKEN` | Kong Konnect personal access token or system account token |

### Variables (plain text)
| Name | Example Value | Description |
|---|---|---|
| `KONNECT_CONTROL_PLANE_NAME` | `my-control-plane` | Name of your Konnect Control Plane |

### Generating a Konnect Token
1. Log in to [cloud.konghq.com](https://cloud.konghq.com)
2. Go to **Personal Access Tokens** (top-right menu) or **System Accounts** (for CI use)
3. Create a token with **Gateway Admin** permissions
4. Add it as the `KONNECT_TOKEN` secret in GitHub

---

## Kong Configuration (`kong/kong.yaml`)

Uses decK's declarative format (`_format_version: "3.0"`). All resources are tagged with `managed-by-deck` to scope what decK manages.

### Adding a new service + route

```yaml
services:
  - name: my-new-service
    url: https://my-backend.example.com
    tags:
      - managed-by-deck
    routes:
      - name: my-new-route
        paths:
          - /api/v3
        methods:
          - GET
        strip_path: true
        tags:
          - managed-by-deck
```

### Adding a plugin to a service

```yaml
services:
  - name: my-new-service
    # ... existing fields ...
    plugins:
      - name: rate-limiting
        config:
          minute: 60
          policy: local
        tags:
          - managed-by-deck
```

### Adding a global plugin

```yaml
plugins:
  - name: prometheus
    config:
      status_code_metrics: true
    tags:
      - managed-by-deck
      - global
```

---

## Local Development

### Install decK

```bash
# macOS
brew install kong/deck/deck

# Linux
curl -sL https://github.com/Kong/deck/releases/download/v1.40.0/deck_1.40.0_linux_amd64.tar.gz \
  | tar -xz -C /usr/local/bin deck
```

### Validate locally before pushing

```bash
# Validate syntax
deck file validate kong/kong.yaml

# Lint for best practices
deck file lint kong/kong.yaml

# Dry-run diff against your Konnect environment
export KONNECT_TOKEN=<your-token>
export KONNECT_CONTROL_PLANE_NAME=<your-cp-name>

deck gateway diff kong/kong.yaml \
  --konnect-token "$KONNECT_TOKEN" \
  --konnect-control-plane-name "$KONNECT_CONTROL_PLANE_NAME"
```

---

## How the PR Workflow Works

1. Open a PR that modifies `kong/kong.yaml`
2. The pipeline validates and runs `deck gateway diff`
3. The diff is posted automatically as a PR comment — reviewers can see exactly what will change before approving
4. On merge to `main`, `deck gateway sync` applies the changes
5. A post-sync verification step confirms zero drift

---

## Troubleshooting

| Issue | Fix |
|---|---|
| `401 Unauthorized` | Verify `KONNECT_TOKEN` secret is set and not expired |
| `Control plane not found` | Check `KONNECT_CONTROL_PLANE_NAME` variable matches exactly |
| Drift detected after sync | A manual change was made in the UI — re-run the workflow |
| Validation fails | Run `deck file validate kong/kong.yaml` locally to debug |
