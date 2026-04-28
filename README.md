# ElderPing Kubernetes Charts

This repository contains Helm charts and ArgoCD configuration for deploying the ElderPing platform to Kubernetes using GitOps methodology.

## Overview

ElderPing uses a GitOps-based deployment strategy with:
- **Helm Charts** for packaging Kubernetes manifests
- **ArgoCD** for continuous deployment and synchronization
- **Single Cluster** with two namespaces (elderping-dev, elderping-prod)
- **Automated CI/CD** via GitHub Actions

## Structure

```
elderping-k8s-charts/
├── microservices/       # Helm charts for each microservice
│   ├── auth-service/
│   │   ├── Chart.yaml
│   │   ├── values-dev.yaml
│   │   ├── values-prod.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── hpa.yaml
│   ├── health-service/
│   ├── reminder-service/
│   ├── alert-service/
│   └── ui-service/
├── databases/           # Database manifests (PostgreSQL StatefulSets)
│   ├── auth-db.yaml
│   ├── health-db.yaml
│   ├── reminder-db.yaml
│   └── alert-db.yaml
├── argocd/              # ArgoCD configuration
│   ├── projects/
│   │   └── elderping.yaml
│   └── environments/
│       ├── dev/
│       │   ├── microservices-appset.yaml
│       │   └── databases-appset.yaml
│       └── prod/
│           ├── microservices-appset.yaml
│           └── databases-appset.yaml
├── dev/                 # Legacy dev manifests (for reference)
├── prod/                # Legacy prod manifests (for reference)
├── sealed-secrets/      # SealedSecret YAML files
└── haproxy.cfg          # HAProxy configuration (legacy)
```

## Environments

### Development (elderping-dev)
- **Namespace**: `elderping-dev`
- **Image Tags**: `dev-latest`
- **Branch**: `develop`
- **Helm Values**: `values-dev.yaml`

### Production (elderping-prod)
- **Namespace**: `elderping-prod`
- **Image Tags**: `prod-latest` or release tags
- **Branch**: `main`
- **Helm Values**: `values-prod.yaml`

## Microservices

| Service | Port | Image Repository | Database |
|---------|------|------------------|----------|
| auth-service | 3000 | arunnsimon/elderpinq-auth-service | auth-db (PostgreSQL) |
| health-service | 3000 | arunnsimon/elderpinq-health-service | health-db (PostgreSQL) |
| reminder-service | 3000 | arunnsimon/elderpinq-reminder-service | reminder-db (PostgreSQL) |
| alert-service | 3000 | arunnsimon/elderpinq-alert-service | alert-db (PostgreSQL) |
| ui-service | 80 | arunnsimon/elderpinq-ui-service | N/A |

## Helm Charts

Each microservice has a Helm chart with:

### Chart.yaml
- API version: v2
- Chart metadata (name, description, version)
- Type: application

### Values Files
- **values-dev.yaml**: Development configuration
  - Image tag: dev-latest
  - Namespace: elderping-dev
  - Resource limits/requests
  - HPA configuration
  - Environment variables
  - Secret references

- **values-prod.yaml**: Production configuration
  - Image tag: prod-latest
  - Namespace: elderping-prod
  - Resource limits/requests
  - HPA configuration
  - Environment variables
  - Secret references

### Templates
- **deployment.yaml**: Kubernetes Deployment with:
  - Container specifications
  - Environment variables from Secrets
  - Resource requests/limits
  - Liveness and readiness probes

- **service.yaml**: Kubernetes Service (ClusterIP)

- **hpa.yaml**: HorizontalPodAutoscaler with:
  - Min/Max replicas
  - CPU/Memory utilization targets

## ArgoCD Configuration

### AppProject
- **Name**: elderping
- **Source Repository**: ElderPing/elderping-k8s-charts
- **Destinations**: All namespaces in-cluster
- **Cluster Resource Whitelist**: All resources

### ApplicationSets

**Development:**
- `dev-microservices-appset.yaml`: Auto-discovers all microservices in `microservices/*`
  - Uses `values-dev.yaml`
  - Deploys to `elderping-dev` namespace
  - Automated sync with prune and self-heal

- `dev-databases-appset.yaml`: Auto-discovers all databases in `databases/*`
  - Deploys to `elderping-dev` namespace
  - Automated sync with prune and self-heal

**Production:**
- `prod-microservices-appset.yaml`: Auto-discovers all microservices in `microservices/*`
  - Uses `values-prod.yaml`
  - Deploys to `elderping-prod` namespace
  - Automated sync with prune and self-heal

- `prod-databases-appset.yaml`: Auto-discovers all databases in `databases/*`
  - Deploys to `elderping-prod` namespace
  - Automated sync with prune and self-heal

## Databases

Each database is deployed as a StatefulSet with:
- **PostgreSQL** container
- **PersistentVolumeClaim** (1Gi, NFS storage class)
- **Service** (ClusterIP)
- **ConfigMap** with init.sql script

Databases:
- **auth-db**: users_db for auth-service
- **health-db**: health_db for health-service
- **reminder-db**: reminder_db for reminder-service
- **alert-db**: alert_db for alert-service

## CI/CD Pipeline

### Development Workflow
1. Developer pushes code to `develop` branch
2. GitHub Actions runs security scans (SAST, SCA, Trivy)
3. Docker image built and tagged as `dev-latest`
4. Image pushed to Docker Hub
5. CD workflow updates `values-dev.yaml` with new tag
6. ArgoCD detects change and syncs to `elderping-dev`

### Production Workflow
1. Release created on `main` branch
2. GitHub Actions runs security scans (SAST, SCA, Trivy)
3. Docker image built and tagged as `prod-latest` or release tag
4. Image pushed to Docker Hub
5. CD workflow updates `values-prod.yaml` with new tag
6. ArgoCD detects change and syncs to `elderping-prod`

## Secrets Management

### Kubernetes Secrets
- **Secret Name**: `elderping-secrets`
- **Keys**:
  - `db-password`: Database password
  - `jwt-secret`: JWT secret for authentication

### Sealed Secrets (Optional)
For production, use Bitnami Sealed Secrets:
- Encrypt secrets with cluster public key
- Commit encrypted SealedSecret YAML to Git
- Controller decrypts automatically in cluster

See `docs/SEALED_SECRETS_SETUP.md` for setup instructions.

## Deployment

### Initial Setup
```bash
# Apply ArgoCD project
kubectl apply -f argocd/projects/elderping.yaml

# Apply ApplicationSets
kubectl apply -f argocd/environments/dev/microservices-appset.yaml
kubectl apply -f argocd/environments/dev/databases-appset.yaml
kubectl apply -f argocd/environments/prod/microservices-appset.yaml
kubectl apply -f argocd/environments/prod/databases-appset.yaml
```

### Manual Helm Install (for testing)
```bash
# Install auth-service in dev
helm install auth-service microservices/auth-service \
  --namespace elderping-dev \
  --create-namespace \
  -f microservices/auth-service/values-dev.yaml

# Install auth-service in prod
helm install auth-service microservices/auth-service \
  --namespace elderping-prod \
  --create-namespace \
  -f microservices/auth-service/values-prod.yaml
```

### Upgrade Existing Release
```bash
helm upgrade auth-service microservices/auth-service \
  --namespace elderping-dev \
  -f microservices/auth-service/values-dev.yaml
```

## Monitoring

The ElderPing platform includes:
- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization and dashboards
- **Loki**: Log aggregation and querying

Services expose metrics endpoints for Prometheus scraping.

## Access

After deployment via ArgoCD:
- **ArgoCD UI**: Port-forward to argocd-server service
- **Services**: Access via Kong Gateway (kgateway)
- **Grafana**: Port-forward to grafana service
- **Prometheus**: Port-forward to prometheus service

## Resource Configuration

### Default Resource Requests
- **Backend Services**: 100m CPU, 128Mi memory
- **UI Service**: 50m CPU, 64Mi memory

### Default Resource Limits
- **Backend Services**: 500m CPU, 256Mi memory
- **UI Service**: 200m CPU, 128Mi memory

### HPA Configuration
- **Backend Services**: 2-5 replicas, 80% CPU target
- **UI Service**: 2-5 replicas, 70% CPU target
- **Alert Service**: 1-3 replicas, 80% CPU target

## Troubleshooting

### ArgoCD Sync Issues
```bash
# Check Application status
kubectl get applications -n argocd

# Check Application sync status
kubectl describe application <app-name> -n argocd

# Force sync
kubectl argocd app sync <app-name>
```

### Pod Issues
```bash
# Check pod logs
kubectl logs <pod-name> -n elderping-dev

# Check pod events
kubectl describe pod <pod-name> -n elderping-dev

# Check pod status
kubectl get pods -n elderping-dev
```

### Database Issues
```bash
# Check StatefulSet status
kubectl get statefulset -n elderping

# Check PVC status
kubectl get pvc -n elderping

# Check database logs
kubectl logs <db-pod-name> -n elderping
```

## Contributing

1. Create feature branch from main
2. Make changes to Helm charts or ArgoCD configuration
3. Test changes in dev environment
4. Commit with descriptive message
5. Create pull request to main
6. After merge, ArgoCD will sync to prod

## License

Proprietary - ElderPing Platform
