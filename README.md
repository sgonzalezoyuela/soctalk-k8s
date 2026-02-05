# SocTalk Kubernetes Deployment

Kustomize-based Kubernetes deployment for SocTalk.

## Directory Structure

```
k8s/
├── base/                    # Base resources (shared across environments)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── postgres/            # PostgreSQL StatefulSet
│   ├── api/                 # FastAPI backend
│   ├── frontend/            # SvelteKit dashboard
│   └── orchestrator/        # LangGraph workflow worker
│
├── overlays/
│   ├── dev/                 # Development environment
│   │   ├── kustomization.yaml
│   │   ├── secrets.env.example
│   │   └── patches/
│   │
│   └── prod/                # Production environment
│       ├── kustomization.yaml
│       ├── secrets.env.example
│       ├── ingress.yaml
│       └── patches/
│
└── components/
    └── mock-endpoint/       # Optional attack simulator
```

## Quick Start

### Prerequisites

- Kubernetes cluster (1.25+)
- kubectl configured
- kustomize (or kubectl with kustomize support)

### Development Deployment

1. **Create secrets file:**
   ```bash
   cp k8s/overlays/dev/secrets.env.example k8s/overlays/dev/secrets.env
   ```

2. **Edit secrets:**
   ```bash
   # Edit with your actual values
   vim k8s/overlays/dev/secrets.env
   ```

3. **Deploy:**
   ```bash
   kubectl apply -k k8s/overlays/dev
   ```

4. **Verify:**
   ```bash
   kubectl -n soctalk get pods
   kubectl -n soctalk get svc
   ```

5. **Port forward for local access:**
   ```bash
   # Frontend
   kubectl -n soctalk port-forward svc/frontend 5173:5173
   
   # API
   kubectl -n soctalk port-forward svc/api 8000:8000
   ```

### Production Deployment

1. **Create secrets:**
   ```bash
   cp k8s/overlays/prod/secrets.env.example k8s/overlays/prod/secrets.env
   # Edit with production values
   ```

2. **Update Ingress:**
   Edit `k8s/overlays/prod/ingress.yaml` with your domain names.

3. **Update image tags:**
   Edit `k8s/overlays/prod/kustomization.yaml` to use specific version tags.

4. **Deploy:**
   ```bash
   kubectl apply -k k8s/overlays/prod
   ```

## Configuration

### Secrets

Secrets are managed via `secretGenerator` in kustomize. Required secrets:

| Secret | Keys | Used By |
|--------|------|---------|
| `soctalk-db-secrets` | `POSTGRES_PASSWORD`, `DATABASE_URL` | postgres, api, orchestrator |
| `soctalk-llm-secrets` | `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` | orchestrator |
| `soctalk-soc-secrets` | `WAZUH_*`, `CORTEX_*`, `THEHIVE_*`, `MISP_*` | orchestrator |

### Images

Base uses placeholder images (`REGISTRY/soctalk-*`). Override in overlays:

```yaml
# overlays/dev/kustomization.yaml
images:
  - name: REGISTRY/soctalk-api
    newName: ghcr.io/yourorg/soctalk-api
    newTag: dev
```

### ConfigMaps

Non-secret configuration is in ConfigMaps:

- `postgres-config` - Database settings
- `api-config` - API settings (CORS, logging)
- `frontend-config` - Frontend settings (API URL)
- `orchestrator-config` - LLM, polling, thresholds, integrations

## Components

### Mock Endpoint (Testing)

Add the attack simulator for testing:

```bash
# Deploy alongside dev overlay
kubectl apply -k k8s/overlays/dev
kubectl apply -k k8s/components/mock-endpoint
```

Or add to overlay:
```yaml
# overlays/dev/kustomization.yaml
components:
  - ../../components/mock-endpoint
```

## Customization

### Adding a New Overlay

1. Create directory: `k8s/overlays/myenv/`
2. Copy from dev: `cp -r k8s/overlays/dev/* k8s/overlays/myenv/`
3. Customize kustomization.yaml, patches, and secrets

### External Database

To use an external database (RDS, CloudSQL, etc.):

1. Remove postgres resources from overlay:
   ```yaml
   # overlays/myenv/kustomization.yaml
   resources:
     - ../../base
   
   # Remove postgres
   patches:
     - target:
         kind: StatefulSet
         name: postgres
       patch: |
         $patch: delete
         kind: StatefulSet
         metadata:
           name: postgres
   ```

2. Update `DATABASE_URL` in secrets to point to external DB.

## Troubleshooting

### Check pod logs
```bash
kubectl -n soctalk logs -f deployment/api
kubectl -n soctalk logs -f deployment/orchestrator
```

### Check events
```bash
kubectl -n soctalk get events --sort-by='.lastTimestamp'
```

### Restart deployments
```bash
kubectl -n soctalk rollout restart deployment/api
kubectl -n soctalk rollout restart deployment/orchestrator
```

### Check secrets
```bash
kubectl -n soctalk get secrets
kubectl -n soctalk describe secret soctalk-db-secrets
```

## Resource Requirements

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| postgres | 100m | 500m | 256Mi | 512Mi |
| api | 100m | 200m | 128Mi | 256Mi |
| frontend | 50m | 100m | 64Mi | 128Mi |
| orchestrator | 100m | 500m | 256Mi | 512Mi |

Production overlay increases these limits.
