# ElderPinq Kubernetes Manifests

This repository contains all Kubernetes manifests for the ElderPinq microservices application, organized by environment (dev and prod).

## Structure

```
elderping-k8s/
├── databases/           # PostgreSQL databases (shared across environments)
│   ├── auth-db.yaml
│   ├── health-db.yaml
│   ├── reminder-db.yaml
│   └── alert-db.yaml
├── dev/                 # Development environment manifests
│   ├── namespace.yaml
│   ├── gateway.yaml
│   ├── gateway-nodeport.yaml
│   ├── httproute.yaml
│   ├── secrets.yaml
│   └── services/
│       ├── auth-service.yaml
│       ├── health-service.yaml
│       ├── reminder-service.yaml
│       ├── alert-service.yaml
│       └── ui-service.yaml
├── prod/                # Production environment manifests
│   ├── namespace.yaml
│   ├── gateway.yaml
│   ├── gateway-nodeport.yaml
│   ├── httproute.yaml
│   ├── secrets.yaml
│   └── services/
│       ├── auth-service.yaml
│       ├── health-service.yaml
│       ├── reminder-service.yaml
│       ├── alert-service.yaml
│       └── ui-service.yaml
└── haproxy.cfg          # HAProxy configuration for routing
```

## Environments

### Development (elderping-dev)
- Namespace: `elderping-dev`
- Image tags: `dev-*` pattern (e.g., `elderpinq-auth-service:dev-abc123`)
- Gateway NodePort: `32588`
- HAProxy port: `8080`

### Production (elderping-prod)
- Namespace: `elderping-prod`
- Image tags: Semver pattern (e.g., `elderpinq-auth-service:v1.0.1`)
- Gateway NodePort: `32587`
- HAProxy port: `80`

## Deployment

### Apply Development Environment
```bash
kubectl apply -f databases/
kubectl apply -f dev/
```

### Apply Production Environment
```bash
kubectl apply -f databases/
kubectl apply -f prod/
```

### Apply Specific Service
```bash
# Dev
kubectl apply -f dev/services/auth-service.yaml

# Prod
kubectl apply -f prod/services/auth-service.yaml
```

## HAProxy Configuration

The `haproxy.cfg` file routes traffic between dev and prod environments:

- **Production**: Port 80 → NodePort 32587 (elderping-prod)
- **Development**: Port 8080 → NodePort 32588 (elderping-dev)

Update the backend server IPs in `haproxy.cfg` to match your Kubernetes node IPs.

## Secrets

The `secrets.yaml` files contain base64 encoded secrets. Replace the default values with actual secrets:

```bash
# Generate base64 encoded secret
echo -n 'yourpassword' | base64
```

Required secrets:
- `db-password`: Database password
- `jwt-secret`: JWT secret for authentication

## Services

### Microservices
- **auth-service**: Authentication and user management (port 3000)
- **health-service**: Health tracking (port 3000)
- **reminder-service**: Medication reminders (port 3000)
- **alert-service**: Alert management (port 3000)
- **ui-service**: React frontend (port 80)

### Databases
- **auth-db**: PostgreSQL for auth service
- **health-db**: PostgreSQL for health service
- **reminder-db**: PostgreSQL for reminder service
- **alert-db**: PostgreSQL for alert service

## Access

After deployment:
- **Production**: http://<haproxy-ip>/
- **Development**: http://<haproxy-ip>:8080/
- **HAProxy Stats**: http://<haproxy-ip>:8404/stats (admin/admin)

## Image Updates

After the CI pipeline pushes new Docker images, update the image tags in the service manifests:

```bash
# Update dev image tag
kubectl set image deployment/auth-service auth-service=arunnsimon/elderpinq-auth-service:dev-abc123 -n elderping-dev

# Update prod image tag
kubectl set image deployment/auth-service auth-service=arunnsimon/elderpinq-auth-service:v1.0.1 -n elderping-prod
```

Or edit the manifests directly and reapply:
```bash
kubectl apply -f dev/services/auth-service.yaml
kubectl apply -f prod/services/auth-service.yaml
```
