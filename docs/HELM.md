# GearGo - Kubernetes Deployment Guide with Helm & ArgoCD

## 📋 Overview

This guide provides step-by-step instructions for deploying the GearGo application to AWS EKS (Elastic Kubernetes Service) using Helm charts and ArgoCD for GitOps-based continuous deployment.

---

## 🎯 Prerequisites

### 1. Required Tools
- [ ] AWS CLI installed and configured (`aws --version`)
- [ ] kubectl installed (`kubectl version --client`)
- [ ] Helm 3.x installed (`helm version`)
- [ ] eksctl installed (`eksctl version`)
- [ ] Docker installed and running
- [ ] Git installed
- [ ] ArgoCD CLI installed (`argocd version`)

### 2. AWS Account Setup
- [ ] AWS account with appropriate permissions
- [ ] IAM user/role with EKS, EC2, IAM permissions
- [ ] AWS credentials configured (`aws configure`)
- [ ] ECR (Elastic Container Registry) repository created

### 3. Application Requirements
- [ ] Docker images built and pushed to ECR
- [ ] Database migration scripts ready
- [ ] Environment variables documented
- [ ] Secrets management strategy defined

---

## 📦 Step 1: Create AWS EKS Cluster

### 1.1 Create EKS Cluster using eksctl

```bash
# Create cluster configuration file
cat > cluster-config.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: geargo-cluster
  region: us-west-2
  version: "1.28"

vpc:
  cidr: "10.0.0.0/16"

nodeGroups:
  - name: geargo-ng-1
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 4
    volumeSize: 20
    ssh:
      allow: true
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        awsLoadBalancerController: true
        ebs: true
        efs: true
        cloudWatch: true
EOF

# Create the cluster
eksctl create cluster -f cluster-config.yaml

# Verify cluster creation
kubectl get nodes
```

### 1.2 Configure kubectl

```bash
# Update kubeconfig
aws eks update-kubeconfig --name geargo-cluster --region us-west-2

# Verify connection
kubectl cluster-info
```

### 1.3 Install AWS Load Balancer Controller

```bash
# Create IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.0/docs/install/iam_policy.json

# Create IAM service account
eksctl create iamserviceaccount \
  --cluster=geargo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::YOUR_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=geargo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## 🐳 Step 2: Build and Push Docker Images to ECR

### 2.1 Create ECR Repository

```bash
# Set variables
AWS_REGION=us-west-2
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REPO=geargo-app

# Create ECR repository
aws ecr create-repository \
  --repository-name $ECR_REPO \
  --region $AWS_REGION \
  --image-scanning-configuration scanOnPush=true

# Get ECR login token
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

### 2.2 Build and Push Images

```bash
# Build Docker image
docker build -t geargo-app:latest .

# Tag image
docker tag geargo-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:latest
docker tag geargo-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v1.0.0

# Push to ECR
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v1.0.0
```

---

## 📊 Step 3: Set Up RDS PostgreSQL Database

### 3.1 Create RDS Instance

```bash
# Create RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier geargo-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username geargo_admin \
  --master-user-password YOUR_SECURE_PASSWORD \
  --allocated-storage 20 \
  --vpc-security-group-ids sg-xxxxxxxxx \
  --db-subnet-group-name geargo-subnet-group \
  --backup-retention-period 7 \
  --storage-encrypted \
  --region us-west-2
```

### 3.2 Configure Security Groups

- Allow inbound traffic from EKS cluster on port 5432
- Update RDS security group to accept connections from EKS nodes

---

## 🔐 Step 4: Create Kubernetes Secrets

### 4.1 Create Namespace

```bash
kubectl create namespace geargo
```

### 4.2 Create Secrets

```bash
# Database secret
kubectl create secret generic geargo-db-secret \
  --from-literal=db-host=geargo-db.xxxxx.us-west-2.rds.amazonaws.com \
  --from-literal=db-name=geargo_db \
  --from-literal=db-user=geargo_admin \
  --from-literal=db-password=YOUR_SECURE_PASSWORD \
  --namespace=geargo

# Django secret key
kubectl create secret generic geargo-django-secret \
  --from-literal=secret-key=YOUR_DJANGO_SECRET_KEY \
  --namespace=geargo

# Redis secret (if needed)
kubectl create secret generic geargo-redis-secret \
  --from-literal=redis-url=redis://redis-service:6379/0 \
  --namespace=geargo
```

---

## 📦 Step 5: Create Helm Chart Structure

### 5.1 Initialize Helm Chart

```bash
# Create chart directory
mkdir -p helm/geargo
cd helm/geargo

# Initialize Helm chart
helm create geargo

# Chart structure will be:
# geargo/
#   ├── Chart.yaml
#   ├── values.yaml
#   ├── templates/
#   │   ├── deployment.yaml
#   │   ├── service.yaml
#   │   ├── ingress.yaml
#   │   ├── configmap.yaml
#   │   ├── secret.yaml (optional - use external secrets)
#   │   └── hpa.yaml (horizontal pod autoscaler)
#   └── templates/tests/
```

### 5.2 Chart Components to Create

**Required Templates:**
- [ ] `deployment.yaml` - Django web application
- [ ] `deployment-celery.yaml` - Celery worker
- [ ] `service.yaml` - Kubernetes service
- [ ] `ingress.yaml` - AWS ALB ingress
- [ ] `configmap.yaml` - Application configuration
- [ ] `hpa.yaml` - Horizontal Pod Autoscaler
- [ ] `pdb.yaml` - Pod Disruption Budget
- [ ] `serviceaccount.yaml` - Service account for pods

**Optional Templates:**
- [ ] `job-migrate.yaml` - Database migration job
- [ ] `cronjob.yaml` - Scheduled tasks
- [ ] `networkpolicy.yaml` - Network policies

---

## 🎨 Step 6: Configure Helm Values

### 6.1 Key Values to Configure in `values.yaml`

```yaml
# Image configuration
image:
  repository: YOUR_ACCOUNT_ID.dkr.ecr.us-west-2.amazonaws.com/geargo-app
  tag: "v1.0.0"
  pullPolicy: IfNotPresent

# Replica configuration
replicaCount: 2

# Application configuration
app:
  name: geargo
  env: production
  
# Database configuration
database:
  host: geargo-db.xxxxx.us-west-2.rds.amazonaws.com
  name: geargo_db
  port: 5432
  secretName: geargo-db-secret

# Redis configuration
redis:
  enabled: true
  host: redis-service
  port: 6379

# Service configuration
service:
  type: ClusterIP
  port: 8000

# Ingress configuration
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:ACCOUNT:certificate/CERT_ID
  hosts:
    - host: geargo.example.com
      paths:
        - path: /
          pathType: Prefix

# Resource limits
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Celery worker
celery:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "200m"
```

---

## 🚀 Step 7: Install ArgoCD

### 7.1 Install ArgoCD in Kubernetes

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 7.2 Install ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# Login to ArgoCD
argocd login localhost:8080 --username admin --password <password-from-step-7.1>
```

### 7.3 Expose ArgoCD via LoadBalancer (Optional)

```bash
# Patch ArgoCD server service
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get LoadBalancer URL
kubectl get svc argocd-server -n argocd
```

---

## 🔄 Step 8: Set Up GitOps with ArgoCD

### 8.1 Create ArgoCD Application Manifest

```bash
# Create ArgoCD application directory
mkdir -p argocd/apps

# Create application.yaml
cat > argocd/apps/geargo-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: geargo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/geargo-helm.git
    targetRevision: main
    path: helm/geargo
    helm:
      valueFiles:
        - values.yaml
      values: |
        image:
          tag: "v1.0.0"
  destination:
    server: https://kubernetes.default.svc
    namespace: geargo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF
```

### 8.2 Create ArgoCD Project (Optional)

```bash
cat > argocd/project.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: geargo-project
  namespace: argocd
spec:
  description: GearGo Application Project
  sourceRepos:
    - '*'
  destinations:
    - namespace: geargo
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: ''
      kind: '*'
EOF
```

### 8.3 Apply ArgoCD Application

```bash
# Apply project (if created)
kubectl apply -f argocd/project.yaml

# Apply application
kubectl apply -f argocd/apps/geargo-app.yaml

# Or use ArgoCD CLI
argocd app create geargo \
  --repo https://github.com/YOUR_USERNAME/geargo-helm.git \
  --path helm/geargo \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace geargo \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

---

## 📝 Step 9: Deploy Application

### 9.1 Manual Helm Deployment (Testing)

```bash
# Add any custom values
helm install geargo ./helm/geargo \
  --namespace geargo \
  --create-namespace \
  --set image.tag=v1.0.0 \
  --set database.host=geargo-db.xxxxx.us-west-2.rds.amazonaws.com

# Upgrade deployment
helm upgrade geargo ./helm/geargo \
  --namespace geargo \
  --set image.tag=v1.0.1

# Check deployment status
helm status geargo -n geargo

# View all resources
kubectl get all -n geargo
```

### 9.2 GitOps Deployment (Production)

```bash
# Commit Helm charts to Git repository
git add helm/ argocd/
git commit -m "Add Helm charts and ArgoCD configuration"
git push origin main

# ArgoCD will automatically sync (if automated sync is enabled)
# Or manually sync via UI/CLI
argocd app sync geargo
```

---

## 🔍 Step 10: Verify Deployment

### 10.1 Check Pod Status

```bash
# Check all pods
kubectl get pods -n geargo

# Check pod logs
kubectl logs -f deployment/geargo-web -n geargo
kubectl logs -f deployment/geargo-celery -n geargo

# Describe pod for issues
kubectl describe pod <pod-name> -n geargo
```

### 10.2 Check Services

```bash
# List services
kubectl get svc -n geargo

# Check ingress
kubectl get ingress -n geargo

# Get ALB URL
kubectl get ingress geargo-ingress -n geargo -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 10.3 Run Database Migrations

```bash
# Create migration job
kubectl create job --from=cronjob/geargo-migrate geargo-migrate-$(date +%s) -n geargo

# Or run manually
kubectl run geargo-migrate --image=$ECR_REPO:latest \
  --restart=Never \
  --env="DATABASE_URL=postgresql://..." \
  --command -- python manage.py migrate
```

---

## 📊 Step 11: Set Up Monitoring & Logging

### 11.1 Install Prometheus & Grafana

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Access Grafana
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```

### 11.2 Set Up CloudWatch Logs

```bash
# Install CloudWatch Logs agent (Fluent Bit)
helm repo add eks https://aws.github.io/eks-charts
helm install aws-for-fluent-bit eks/aws-for-fluent-bit \
  --namespace logging \
  --create-namespace \
  --set cloudWatchLogs.enabled=true \
  --set cloudWatchLogs.region=us-west-2
```

---

## 🔒 Step 12: Security Hardening

### 12.1 Network Policies

```bash
# Create network policy to restrict pod communication
kubectl apply -f helm/geargo/templates/networkpolicy.yaml
```

### 12.2 Pod Security Standards

```bash
# Enable Pod Security Standards
kubectl label namespace geargo pod-security.kubernetes.io/enforce=restricted
```

### 12.3 RBAC Configuration

```bash
# Review and apply RBAC rules
kubectl apply -f helm/geargo/templates/rbac.yaml
```

---

## 🔄 Step 13: CI/CD Integration

### 13.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to EKS
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build and push Docker image
        # ... build steps
      - name: Update Helm chart version
        # ... update chart version
      - name: Commit and push changes
        # ... git commit
      - name: Trigger ArgoCD sync
        # ... sync ArgoCD app
```

---

## 📋 Step 14: Post-Deployment Checklist

- [ ] Application is accessible via ALB URL
- [ ] Database migrations completed successfully
- [ ] Static files are being served correctly
- [ ] Celery workers are processing tasks
- [ ] Health checks are passing
- [ ] Logs are being collected
- [ ] Monitoring dashboards are set up
- [ ] SSL certificates are configured
- [ ] Backup strategy is in place
- [ ] Disaster recovery plan is documented

---

## 🛠️ Troubleshooting

### Common Issues

1. **Pods not starting**
   ```bash
   kubectl describe pod <pod-name> -n geargo
   kubectl logs <pod-name> -n geargo
   ```

2. **Database connection issues**
   - Check RDS security groups
   - Verify database secret
   - Test connection from pod

3. **Image pull errors**
   - Verify ECR permissions
   - Check image pull secrets
   - Confirm image exists in ECR

4. **ArgoCD sync issues**
   ```bash
   argocd app get geargo
   argocd app logs geargo
   ```

---

## 📚 Additional Resources

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Helm Documentation](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

## 🎯 Next Steps

1. Set up production-grade monitoring
2. Implement backup and restore procedures
3. Configure auto-scaling policies
4. Set up staging environment
5. Implement blue-green deployments
6. Add service mesh (Istio/Linkerd) if needed
7. Set up cost monitoring and optimization

---

**Note:** Replace all placeholder values (YOUR_ACCOUNT_ID, YOUR_SECURE_PASSWORD, etc.) with actual values before executing commands.

