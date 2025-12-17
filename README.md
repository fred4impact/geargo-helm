# GearGo Helm chart

This repository contains a Helm chart for deploying the GearGo application and an optional in-cluster PostgreSQL for testing.

Contents
- `Chart.yaml`, `values.yaml`, and `templates/` — the Helm chart
- `.github/workflows/ci-deploy.yml` — CI workflow: builds Docker image, pushes to ECR, deploys Helm

## Quick goals
- Provide a simple way to run GearGo in a Kubernetes cluster for development/test
- Optionally deploy a single-replica PostgreSQL inside the cluster (not intended for production)
- Provide a CI workflow to build/push Docker images to ECR and run Helm deploys

## Requirements
- Helm 3
- kubectl configured for your target cluster
- Docker (for local builds)

## Important files
- `values.yaml` — main configuration. Notable sections:
  - `postgres` — controls the in-cluster Postgres (enabled by default, password empty)
  - `database` — used by the app; `database.secretName` can be pointed to an external secret (RDS)
  - `image` — image repository and tag for the application

- `templates/postgres-*` — templates added to optionally deploy Postgres, PVC, and Secret

## Local validation (no cluster changes)
Run these from the repository root:
```bash
helm lint .
helm template geargo . --namespace geargo
```

## Install to a cluster (recommended: test namespace)
Set a strong Postgres password before installing. The chart enables Postgres by default but leaves
`postgres.password` empty to avoid committing secrets.

Example (zsh):
```bash
helm upgrade --install geargo . -n geargo --create-namespace \
  --set postgres.enabled=true \
  --set postgres.password='ReplaceWithStrongPassword!' \
  --set image.repository=123456789012.dkr.ecr.us-east-1.amazonaws.com/geargo \
  --set image.tag=v1.2.3
```

Check resources:
```bash
kubectl -n geargo get all
kubectl -n geargo get pvc
kubectl -n geargo logs -l app.kubernetes.io/component=postgres -c postgres
```

To port-forward Postgres locally:
```bash
kubectl -n geargo port-forward svc/geargo-postgres 5432:5432
PGPASSWORD='ReplaceWithStrongPassword!' psql -h 127.0.0.1 -p 5432 -U geargo geargo_db
```

## Using an external database (RDS/Aurora)
If you use a managed DB in production keep `postgres.enabled=false` and set the following in `values.yaml` or via `--set`:
- `database.host` — DB hostname
- `database.port` — DB port (default 5432)
- `database.secretName` — name of Kubernetes Secret containing DB credentials and `database-url`

The web and celery containers read `DATABASE_URL` from the secret referenced by `database.secretName` (or the generated secret when in-cluster Postgres is enabled).

## GitHub Actions workflow
The included workflow `.github/workflows/ci-deploy.yml` performs:
1. Build Docker image and push to ECR
2. Deploy Helm chart to the target cluster

Secrets required in the GitHub repository settings (Actions → Secrets and variables):
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `AWS_ACCOUNT_ID`
- `ECR_REPOSITORY` (e.g., `geargo`)
- `POSTGRES_PASSWORD` (strong password to pass into Helm)
- `KUBE_CONFIG_DATA` — base64 of your kubeconfig file used by kubectl/helm in CI

Notes:
- `KUBE_CONFIG_DATA` must be base64-encoded. On macOS/Linux: `cat ~/.kube/config | base64 | pbcopy` (or copy output and paste into the secret value).
- The workflow uses the pushed image tag `${{ github.sha }}` — if you prefer semver tags, adjust the workflow accordingly.

## Security notes
- Do not commit passwords or plaintext credentials to the repository.
- For production, prefer a managed DB (RDS/Aurora), multi-AZ, regular backups, and a production-grade Helm chart (for example, Bitnami PostgreSQL).
- Consider using SealedSecrets, External Secrets, or a secrets operator to manage credentials in the cluster.

## Troubleshooting
- Helm lint errors: fix template errors reported by `helm lint` — read the error and the template name it references.
- If PVC remains Pending: verify your cluster has a default StorageClass or set `postgres.persistence.storageClass` in `values.yaml`.
- App can't connect to DB: verify the secret name referenced by `database.secretName` contains `database-url` and correct credentials.

## Next steps I can help with
- Replace the in-chart Postgres with the Bitnami Postgres subchart (HA + backups)
- Add SealedSecrets / External Secrets integration
- Tweak the GitHub Actions workflow to use image tags (semver) and add deployment approvals

---
If you want I can now update the README with cluster-specific examples (EKS / GKE) or add a short section describing how to rotate Postgres credentials safely.
# GearGo Helm Chart

A Helm chart for deploying GearGo - AI-Powered Rental Marketplace to Kubernetes.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- AWS EKS cluster (or compatible Kubernetes cluster)
- RDS PostgreSQL database
- Redis instance (or use external Redis)
- ECR repository with Docker images

## Installation

### 1. Create Required Secrets

Before installing the chart, create the required secrets:

```bash
# Database secret
kubectl create secret generic geargo-db-secret \
  --from-literal=database-url=postgresql://user:password@host:5432/dbname \
  --namespace=geargo

# Django secret key
kubectl create secret generic geargo-django-secret \
  --from-literal=secret-key=your-secret-key-here \
  --namespace=geargo
```

### 2. Install the Chart

```bash
# Add the repository (if using a chart repository)
helm repo add geargo https://charts.geargo.com
helm repo update

# Install with default values
helm install geargo ./helm/geargo \
  --namespace geargo \
  --create-namespace

# Or install with custom values
helm install geargo ./helm/geargo \
  --namespace geargo \
  --create-namespace \
  --set image.repository=YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/geargo-app \
  --set image.tag=v1.0.0 \
  --set database.host=geargo-db.xxxxx.us-west-2.rds.amazonaws.com
```

### 3. Upgrade the Chart

```bash
helm upgrade geargo ./helm/geargo \
  --namespace geargo \
  --set image.tag=v1.0.1
```

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Docker image repository | `YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/geargo-app` |
| `image.tag` | Docker image tag | `v1.0.0` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `replicaCount` | Number of web replicas | `2` |
| `web.replicaCount` | Number of web replicas | `2` |
| `celery.enabled` | Enable Celery worker | `true` |
| `celery.replicaCount` | Number of Celery replicas | `2` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `8000` |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.className` | Ingress class name | `alb` |
| `ingress.hosts[0].host` | Ingress host | `geargo.example.com` |
| `database.host` | Database host | `geargo-db.xxxxx.us-west-2.rds.amazonaws.com` |
| `database.secretName` | Database secret name | `geargo-db-secret` |
| `redis.external` | Use external Redis | `true` |
| `redis.host` | Redis host | `redis-service` |
| `autoscaling.enabled` | Enable HPA | `true` |
| `autoscaling.minReplicas` | Minimum replicas | `2` |
| `autoscaling.maxReplicas` | Maximum replicas | `10` |
| `migration.enabled` | Enable migration job | `true` |
| `persistence.static.enabled` | Enable static files PVC | `false` |
| `persistence.media.enabled` | Enable media files PVC | `false` |

## Values File Example

Create a `values-production.yaml` file:

```yaml
image:
  repository: 123456789012.dkr.ecr.us-west-2.amazonaws.com/geargo-app
  tag: "v1.0.0"

replicaCount: 3

web:
  replicaCount: 3
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 1000m
      memory: 1Gi

celery:
  enabled: true
  replicaCount: 2

database:
  host: geargo-db-prod.xxxxx.us-west-2.rds.amazonaws.com
  secretName: geargo-db-secret

ingress:
  enabled: true
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:123456789012:certificate/xxxxx
  hosts:
    - host: geargo.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
```

Then install with:

```bash
helm install geargo ./helm/geargo \
  --namespace geargo \
  --create-namespace \
  -f values-production.yaml
```

## Uninstallation

```bash
helm uninstall geargo --namespace geargo
```

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n geargo
kubectl describe pod <pod-name> -n geargo
kubectl logs <pod-name> -n geargo
```

### Check Services

```bash
kubectl get svc -n geargo
kubectl get ingress -n geargo
```

### Check Migration Job

```bash
kubectl get jobs -n geargo
kubectl logs job/<job-name> -n geargo
```

### Database Connection Issues

```bash
# Test database connection from a pod
kubectl run -it --rm debug --image=postgres:15-alpine --restart=Never -- \
  psql postgresql://user:password@host:5432/dbname
```

## Notes

- The migration job runs automatically before each deployment
- Static and media files use emptyDir by default (ephemeral)
- For production, consider using S3 for static/media files
- Database and Redis should be external services (RDS, ElastiCache)
- Secrets must be created manually before installation

