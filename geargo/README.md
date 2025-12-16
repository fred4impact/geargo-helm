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

