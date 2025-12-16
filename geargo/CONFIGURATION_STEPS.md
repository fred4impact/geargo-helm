# GearGo Helm Chart - Configuration & Deployment Steps

This guide provides step-by-step instructions for deploying the GearGo Helm chart on AWS EKS using ArgoCD.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [AWS EKS Cluster Setup](#aws-eks-cluster-setup)
3. [Pre-Deployment Configuration](#pre-deployment-configuration)
4. [ArgoCD Installation & Configuration](#argocd-installation--configuration)
5. [Helm Chart Configuration](#helm-chart-configuration)
6. [ArgoCD Application Setup](#argocd-application-setup)
7. [Deployment & Verification](#deployment--verification)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Tools

```bash
# Install required CLI tools
# AWS CLI
aws --version  # Should be v2.x or later

# kubectl
kubectl version --client

# Helm
helm version

# ArgoCD CLI (optional but recommended)
# macOS
brew install argocd

# Linux
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

### AWS Resources Required

- ✅ AWS EKS Cluster (1.19+)
- ✅ ECR Repository for Docker images
- ✅ RDS PostgreSQL Database
- ✅ ElastiCache Redis (or external Redis)
- ✅ AWS Certificate Manager (ACM) certificate for HTTPS (optional)
- ✅ Route53 hosted zone (optional, for custom domain)
- ✅ IAM roles and policies for EKS cluster

---

## AWS EKS Cluster Setup

### Step 1: Configure kubectl for Your EKS Cluster

```bash
# Update kubeconfig
aws eks update-kubeconfig --region us-west-2 --name your-eks-cluster-name

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### Step 2: Install AWS Load Balancer Controller

The Helm chart uses AWS ALB Ingress, so you need the AWS Load Balancer Controller:

```bash
# Add EKS Helm chart repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=your-eks-cluster-name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify installation
kubectl get deployment aws-load-balancer-controller -n kube-system
```

**Note:** Ensure your EKS cluster has the necessary IAM roles and policies for the Load Balancer Controller.

---

## Pre-Deployment Configuration

### Step 3: Create Namespace

```bash
kubectl create namespace geargo
```

### Step 4: Configure ECR Image Pull Secrets

If your ECR repository requires authentication:

```bash
# Get AWS account ID
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=us-west-2

# Create ECR login secret
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ${AWS_REGION}) \
  --namespace=geargo
```

### Step 5: Create Database Secret

```bash
# Create database secret
kubectl create secret generic geargo-db-secret \
  --from-literal=db-user=your-db-username \
  --from-literal=db-password=your-db-password \
  --namespace=geargo

# Verify secret
kubectl get secret geargo-db-secret -n geargo
```

### Step 6: Create Django Secret Key

```bash
# Generate a secure Django secret key
DJANGO_SECRET_KEY=$(python3 -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())')

# Create Django secret
kubectl create secret generic geargo-django-secret \
  --from-literal=secret-key="${DJANGO_SECRET_KEY}" \
  --namespace=geargo

# Verify secret
kubectl get secret geargo-django-secret -n geargo
```

### Step 7: Verify External Services

Ensure your RDS and Redis services are accessible from the EKS cluster:

```bash
# Test RDS connectivity (from a test pod)
kubectl run -it --rm postgres-test --image=postgres:15-alpine --restart=Never --namespace=geargo -- \
  psql -h geargo-db.xxxxx.us-west-2.rds.amazonaws.com -U your-db-username -d geargo_db

# Test Redis connectivity
kubectl run -it --rm redis-test --image=redis:7-alpine --restart=Never --namespace=geargo -- \
  redis-cli -h your-redis-endpoint.cache.amazonaws.com -p 6379 ping
```

---

## ArgoCD Installation & Configuration

### Step 8: Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready (this may take 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Check ArgoCD pods
kubectl get pods -n argocd
```

### Step 9: Get ArgoCD Admin Password

```bash
# Get initial admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD Admin Password: ${ARGOCD_PASSWORD}"

# Save password for later use
echo "${ARGOCD_PASSWORD}" > argocd-admin-password.txt
chmod 600 argocd-admin-password.txt
```

### Step 10: Access ArgoCD UI

#### Option A: Port Forward (Local Access)

```bash
# Port forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access ArgoCD UI
# URL: https://localhost:8080
# Username: admin
# Password: <from step 9>
```

#### Option B: Expose via LoadBalancer (AWS)

```bash
# Patch ArgoCD server service to use LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get LoadBalancer URL
kubectl get svc argocd-server -n argocd

# Access via the LoadBalancer URL
# Note: You may need to configure DNS or use the AWS-provided URL
```

### Step 11: Login to ArgoCD CLI

```bash
# Login to ArgoCD
argocd login localhost:8080 --username admin --password "${ARGOCD_PASSWORD}" --insecure

# Or if using LoadBalancer
# argocd login <loadbalancer-url> --username admin --password "${ARGOCD_PASSWORD}" --insecure
```

### Step 12: Configure ArgoCD Repository Access

If your Helm chart is in a private Git repository, configure repository credentials:

```bash
# Add Git repository (if private, you'll need to provide credentials)
argocd repo add https://github.com/YOUR_USERNAME/geargo-helm.git \
  --name geargo-helm \
  --username YOUR_GITHUB_USERNAME \
  --password YOUR_GITHUB_TOKEN

# Or for SSH
argocd repo add git@github.com:YOUR_USERNAME/geargo-helm.git \
  --name geargo-helm \
  --ssh-private-key-path ~/.ssh/id_rsa
```

---

## Helm Chart Configuration

### Step 13: Create Custom Values File

Create a production values file: `values-production.yaml`

```yaml
# values-production.yaml

# Application name
nameOverride: ""
fullnameOverride: ""

# Image configuration
image:
  repository: YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/geargo-app
  pullPolicy: IfNotPresent
  tag: "v1.0.0"

# Image pull secrets (for ECR)
imagePullSecrets:
  - name: ecr-registry-secret

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Replica count
replicaCount: 3

# Web application deployment
web:
  enabled: true
  replicaCount: 3
  
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 1000m
      memory: 1Gi
  
  livenessProbe:
    httpGet:
      path: /
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  
  readinessProbe:
    httpGet:
      path: /
      port: http
    initialDelaySeconds: 10
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3

# Celery worker deployment
celery:
  enabled: true
  replicaCount: 2
  
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 500m
      memory: 512Mi

# Service configuration
service:
  type: ClusterIP
  port: 8000
  targetPort: 8000
  annotations: {}

# Ingress configuration
ingress:
  enabled: true
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:YOUR_ACCOUNT_ID:certificate/YOUR_CERT_ID
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
  hosts:
    - host: geargo.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

# Database configuration
database:
  host: geargo-db.xxxxx.us-west-2.rds.amazonaws.com
  port: 5432
  name: geargo_db
  secretName: geargo-db-secret
  secretKeys:
    username: db-user
    password: db-password

# Redis configuration
redis:
  external: true
  host: your-redis-endpoint.cache.amazonaws.com
  port: 6379

# Application configuration
app:
  name: geargo
  env: production
  debug: "False"
  
  django:
    secretKeySecretName: geargo-django-secret
    secretKeySecretKey: secret-key
    allowedHosts: "geargo.yourdomain.com,*.elb.amazonaws.com"
    csrfTrustedOrigins: "https://geargo.yourdomain.com"
  
  staticFiles:
    enabled: true
    # Optional: Use S3 for static files
    # s3Bucket: geargo-static-files
    # s3Region: us-west-2
  
  mediaFiles:
    enabled: true
    # Optional: Use S3 for media files
    # s3Bucket: geargo-media-files
    # s3Region: us-west-2

# Environment variables
env: []

# Secrets (references to existing secrets)
secrets:
  database:
    name: geargo-db-secret
    create: false
  
  django:
    name: geargo-django-secret
    create: false

# Horizontal Pod Autoscaler
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 2

# Database migration job
migration:
  enabled: true
  image:
    repository: YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/geargo-app
    tag: "v1.0.0"
    pullPolicy: IfNotPresent
  runOnUpgrade: true
  backoffLimit: 3
  ttlSecondsAfterFinished: 86400

# Logging
logging:
  enabled: true
  cloudwatch:
    enabled: true
    logGroup: /aws/eks/geargo
    region: us-west-2
```

**Important:** Replace the following placeholders in `values-production.yaml`:
- `YOUR_ACCOUNT_ID` - Your AWS Account ID
- `YOUR_CERT_ID` - Your ACM Certificate ID (if using HTTPS)
- `geargo.yourdomain.com` - Your domain name
- `geargo-db.xxxxx.us-west-2.rds.amazonaws.com` - Your RDS endpoint
- `your-redis-endpoint.cache.amazonaws.com` - Your ElastiCache Redis endpoint

---

## ArgoCD Application Setup

### Step 14: Create ArgoCD Application Manifest

Create `argocd-application.yaml`:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: geargo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  labels:
    app: geargo
    environment: production
spec:
  project: default
  
  # Source repository configuration
  source:
    # Option 1: If Helm chart is in a Git repository
    repoURL: https://github.com/YOUR_USERNAME/geargo-helm.git
    targetRevision: main  # or your default branch
    path: helm/geargo
    helm:
      valueFiles:
        - values-production.yaml
      # Or use inline values
      # values: |
      #   image:
      #     repository: YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/geargo-app
      #     tag: "v1.0.0"
    
    # Option 2: If using a Helm repository
    # repoURL: https://charts.geargo.com
    # chart: geargo
    # targetRevision: 1.0.0
    # helm:
    #   valueFiles:
    #     - values-production.yaml
  
  # Destination cluster and namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: geargo
  
  # Sync policy
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
```

### Step 15: Deploy ArgoCD Application

```bash
# Apply ArgoCD application
kubectl apply -f argocd-application.yaml

# Check application status
kubectl get application geargo -n argocd

# Watch application sync
argocd app get geargo

# Or view in ArgoCD UI
# Navigate to the ArgoCD UI and check the "geargo" application
```

### Step 16: Manual Sync (if needed)

If automatic sync is disabled or you want to trigger a manual sync:

```bash
# Sync application manually
argocd app sync geargo

# Or via kubectl
kubectl patch application geargo -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"main"}}}'
```

---

## Deployment & Verification

### Step 17: Verify Deployment

```bash
# Check namespace
kubectl get namespace geargo

# Check all resources
kubectl get all -n geargo

# Check pods
kubectl get pods -n geargo
kubectl get pods -n geargo -l app=geargo

# Check services
kubectl get svc -n geargo

# Check ingress
kubectl get ingress -n geargo

# Check HPA
kubectl get hpa -n geargo

# Check PDB
kubectl get pdb -n geargo

# Check migration job
kubectl get jobs -n geargo
```

### Step 18: Check Pod Logs

```bash
# Web application logs
kubectl logs -n geargo -l app=geargo,component=web --tail=100

# Celery worker logs
kubectl logs -n geargo -l app=geargo,component=celery --tail=100

# Migration job logs
kubectl logs -n geargo -l app=geargo,component=migration --tail=100
```

### Step 19: Verify Database Migration

```bash
# Check migration job status
kubectl get jobs -n geargo

# View migration job logs
MIGRATION_JOB=$(kubectl get jobs -n geargo -l app=geargo,component=migration -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n geargo job/${MIGRATION_JOB}
```

### Step 20: Test Application Access

```bash
# Get ingress URL
kubectl get ingress -n geargo

# Test health endpoint (if available)
INGRESS_URL=$(kubectl get ingress -n geargo -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
curl -I http://${INGRESS_URL}/

# Or if using HTTPS
curl -I https://${INGRESS_URL}/
```

### Step 21: Monitor Application

```bash
# Watch pods
kubectl get pods -n geargo -w

# Watch HPA
kubectl get hpa -n geargo -w

# Describe pod (if issues)
kubectl describe pod <pod-name> -n geargo

# Check events
kubectl get events -n geargo --sort-by='.lastTimestamp'
```

---

## Troubleshooting

### Issue: Pods Not Starting

```bash
# Check pod status
kubectl get pods -n geargo

# Describe pod for details
kubectl describe pod <pod-name> -n geargo

# Check pod logs
kubectl logs <pod-name> -n geargo

# Common issues:
# - Image pull errors: Check ECR credentials and imagePullSecrets
# - Database connection: Verify RDS security groups and secrets
# - Resource limits: Check if nodes have enough resources
```

### Issue: Database Connection Failed

```bash
# Verify database secret
kubectl get secret geargo-db-secret -n geargo -o yaml

# Test database connectivity from pod
kubectl run -it --rm db-test --image=postgres:15-alpine --restart=Never --namespace=geargo -- \
  psql -h geargo-db.xxxxx.us-west-2.rds.amazonaws.com -U your-db-username -d geargo_db

# Check RDS security group allows EKS cluster access
# Ensure RDS is in the same VPC or has proper network configuration
```

### Issue: Ingress Not Creating ALB

```bash
# Check ingress status
kubectl describe ingress -n geargo

# Check AWS Load Balancer Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Verify AWS Load Balancer Controller is running
kubectl get deployment aws-load-balancer-controller -n kube-system

# Check IAM roles and policies for Load Balancer Controller
```

### Issue: ArgoCD Sync Failing

```bash
# Check application status
argocd app get geargo

# Check application events
argocd app history geargo

# View detailed sync status
argocd app get geargo --show-operation

# Check repository access
argocd repo list

# Verify Helm chart path and values file location
```

### Issue: Migration Job Failing

```bash
# Check migration job
kubectl get jobs -n geargo

# View migration logs
kubectl logs -n geargo job/<migration-job-name>

# Check if database is accessible
# Verify database secret is correct
# Ensure database user has proper permissions
```

### Issue: High Memory/CPU Usage

```bash
# Check resource usage
kubectl top pods -n geargo

# Check HPA status
kubectl get hpa -n geargo

# Adjust resources in values.yaml if needed
# Review autoscaling configuration
```

### Common Commands for Debugging

```bash
# Get all resources in namespace
kubectl get all -n geargo

# Describe service
kubectl describe svc <service-name> -n geargo

# Get events
kubectl get events -n geargo --sort-by='.lastTimestamp'

# Check ConfigMap
kubectl get configmap -n geargo
kubectl describe configmap <configmap-name> -n geargo

# Check Secrets (without revealing values)
kubectl get secrets -n geargo
kubectl describe secret <secret-name> -n geargo

# Port forward for local testing
kubectl port-forward svc/geargo -n geargo 8000:8000
```

---

## Post-Deployment Tasks

### Step 22: Configure DNS (if using custom domain)

```bash
# Get ALB DNS name from ingress
ALB_DNS=$(kubectl get ingress -n geargo -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')

# Create Route53 record (example)
aws route53 change-resource-record-sets \
  --hosted-zone-id YOUR_HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "geargo.yourdomain.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "'${ALB_DNS}'"}]
      }
    }]
  }'
```

### Step 23: Set Up Monitoring (Optional)

```bash
# If using Prometheus, create ServiceMonitor
# If using CloudWatch, ensure proper IAM roles
# Configure logging to CloudWatch Logs
```

### Step 24: Backup Configuration

```bash
# Backup values file
cp values-production.yaml values-production.yaml.backup

# Export current configuration
helm get values geargo -n geargo > current-values.yaml
```

---

## Updating the Deployment

### Update via ArgoCD (Recommended)

```bash
# Update values in Git repository
# ArgoCD will automatically sync if automated sync is enabled

# Or trigger manual sync
argocd app sync geargo
```

### Update via Helm (Direct)

```bash
# Update values file
# Then upgrade
helm upgrade geargo ./helm/geargo \
  -n geargo \
  -f values-production.yaml

# Or update specific values
helm upgrade geargo ./helm/geargo \
  -n geargo \
  --set image.tag=v1.0.1
```

---

## Rollback

### Rollback via ArgoCD

```bash
# View application history
argocd app history geargo

# Rollback to previous revision
argocd app rollback geargo <revision-number>
```

### Rollback via Helm

```bash
# View release history
helm history geargo -n geargo

# Rollback to previous revision
helm rollback geargo <revision-number> -n geargo
```

---

## Cleanup

### Remove Application

```bash
# Delete ArgoCD application
kubectl delete application geargo -n argocd

# Or via ArgoCD CLI
argocd app delete geargo
```

### Remove Helm Release

```bash
# Uninstall Helm release
helm uninstall geargo -n geargo
```

### Remove Namespace and Resources

```bash
# Delete namespace (this will delete all resources)
kubectl delete namespace geargo

# Note: Secrets and PVCs may persist, delete manually if needed
```

---

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Helm Documentation](https://helm.sh/docs/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)

---

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review pod logs and events
3. Check ArgoCD application status
4. Verify all prerequisites are met

---

**Last Updated:** $(date)
**Chart Version:** 1.0.0
**Kubernetes Version:** 1.19+

