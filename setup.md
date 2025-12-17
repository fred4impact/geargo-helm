# EKS Cluster Setup and Application Deployment Guide

This guide walks you through configuring your EKS cluster, installing ArgoCD, deploying the Geargo application, and setting up monitoring with Prometheus, Grafana, and Node Exporter.

## 📋 Prerequisites
- provision aws infra using https://github.com/fred4impact/02-eks-microservices-deployment.git
- EKS cluster provisioned via Terraform
- AWS CLI configured with appropriate credentials
- `kubectl` installed
- `helm` installed
- Access to the bastion host (if using) or local machine with AWS credentials

---

## 🔧 Step 1: Configure kubectl for EKS Cluster

### 1.1 Connect to Bastion Host (if using)

```bash
# Get bastion public IP from Terraform outputs
cd setup-eks-cluster
terraform output bastion_public_ip

# SSH into bastion
ssh -i ~/.ssh/your-key.pem ec2-user@<bastion-public-ip>

# Configure kubectl on bastion
./configure-kubectl.sh

AWS Configure 
Add 
ACCESS KEYS  ———————
SECRET ACCESS  KEYS ———————
```

### 1.2 Configure kubectl Locally (Alternative)

```bash
# Get cluster name from Terraform outputs
cd setup-eks-cluster
terraform output cluster_name

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>

# Verify access
kubectl get nodes
kubectl get pods --all-namespaces
```

### 1.3 Verify Cluster Access

```bash
# Check cluster info
kubectl cluster-info

# List all nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system
```

---

## 🚀 Step 2: Install Helm (if not already installed)

### 2.1 Install Helm on Bastion/Local Machine

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

---

## 🔄 Step 3: Install ArgoCD

### 3.1 Add ArgoCD Helm Repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 3.2 Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD using Helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=LoadBalancer \
  --set server.ingress.enabled=true \
  --set server.ingress.ingressClassName=nginx \
  --set server.ingress.hosts[0]=argocd.yourdomain.com \
  --set controller.replicas=1 \
  --set redis-ha.enabled=false \
  --set redis.enabled=true

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 3.3 Get ArgoCD Admin Password

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Save this password - you'll need it to login.

### 3.4 Access ArgoCD UI

#### Option A: Port Forward (Quick Access)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access: `https://localhost:8080`
- Username: `admin`
- Password: (from step 3.3)

#### Option B: LoadBalancer (if configured)

```bash
# Get LoadBalancer URL
kubectl get svc argocd-server -n argocd

# Access via the EXTERNAL-IP or DNS name
```

### 3.5 Install ArgoCD CLI (Optional)

```bash
# On Linux
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# On macOS
brew install argocd

# Login to ArgoCD
argocd login localhost:8080 --username admin --password <password>
```

---

## 📦 Step 4: Deploy Geargo Application using Helm

### 4.1 Add Geargo Helm Repository

```bash
# Add the geargo-helm repository
helm repo add geargo https://raw.githubusercontent.com/fred4impact/geargo-helm/master
helm repo update


### build and push docker  image 

aws ecr get-login-password --region "us-east-1" | docker login --username AWS --password-stdin "578690312663.dkr.ecr.us-east-1.amazonaws.com/geargo.dkr.ecr.us-east-1.amazonaws.com"



# build (optional)
docker build -t geargo:latest .

# tag for ECR
docker tag geargo:latest 578690312663.dkr.ecr.us-east-1.amazonaws.com/geargo:latest


# push
docker push 578690312663.dkr.ecr.us-east-1.amazonaws.com/geargo:latest

```

### 4.2 Clone and Review Helm Chart (Recommended)

```bash
# Clone the repository to review values
git clone https://github.com/fred4impact/geargo-helm.git
cd geargo-helm

# Review the chart structure
ls -la geargo/

# Check available values
cat geargo/values.yaml
```

### 4.3 Install Geargo Application

```bash
# Create namespace for geargo
kubectl create namespace geargo

# Install using Helm (from cloned repo)
cd geargo-helm
helm install geargo ./geargo \
  --namespace geargo \
  --set image.repository=<your-ecr-repo-url> \
  --set image.tag=latest \
  --set service.type=LoadBalancer

# Or install directly from GitHub (if chart is published)
helm install geargo geargo/geargo \
  --namespace geargo \
  --create-namespace
```

### 4.4 Verify Geargo Deployment

```bash
# Check pods
kubectl get pods -n geargo

# Check services
kubectl get svc -n geargo

# Check deployment status
kubectl get deployments -n geargo
```

### 4.5 Access Geargo Application

```bash
# Get LoadBalancer URL
kubectl get svc -n geargo

# Access via the EXTERNAL-IP
```

---

## 📊 Step 5: Install Prometheus and Grafana

### 5.1 Add Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 5.2 Install kube-prometheus-stack (Prometheus + Grafana)

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus and Grafana stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=30d \
  --set grafana.adminPassword=admin \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.service.type=LoadBalancer

# Wait for pods to be ready
kubectl wait --for=condition=available --timeout=300s deployment/prometheus-operator -n monitoring
```

### 5.3 Get Grafana Admin Credentials

```bash
# Default username is 'admin'
# Get password
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d && echo
```

### 5.4 Access Grafana UI

#### Option A: Port Forward

```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```

Access: `http://localhost:3000`
- Username: `admin`
- Password: (from step 5.3)

#### Option B: LoadBalancer

```bash
# Get LoadBalancer URL
kubectl get svc prometheus-grafana -n monitoring
```

### 5.5 Access Prometheus UI

```bash
# Port forward to Prometheus
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```

Access: `http://localhost:9090`

---

## 📈 Step 6: Install Node Exporter

### 6.1 Install Node Exporter using Helm

```bash
# Add node-exporter repository (if separate)
helm repo add prometheus-node-exporter https://prometheus-community.github.io/helm-charts
helm repo update

# Install node-exporter
helm install node-exporter prometheus-community/prometheus-node-exporter \
  --namespace monitoring \
  --set serviceMonitor.enabled=true

# Verify installation
kubectl get pods -n monitoring | grep node-exporter
```

**Note:** If you installed `kube-prometheus-stack` in Step 5, Node Exporter is likely already included. Verify with:

```bash
kubectl get daemonset -n monitoring
kubectl get pods -n monitoring | grep node-exporter
```

### 6.2 Verify Node Exporter Metrics

```bash
# Check if node-exporter is running on each node
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-node-exporter

# Port forward to test metrics endpoint
kubectl port-forward -n monitoring <node-exporter-pod-name> 9100:9100

# Test metrics (in another terminal)
curl http://localhost:9100/metrics
```

---

## 🔗 Step 7: Configure ArgoCD to Deploy Geargo (GitOps)

### 7.1 Create ArgoCD Application for Geargo

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: geargo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/fred4impact/geargo-helm.git
    targetRevision: master
    path: geargo
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: geargo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### 7.2 Verify ArgoCD Application

```bash
# Check application status
kubectl get application -n argocd

# Get detailed status
argocd app get geargo-app

# Sync application manually (if needed)
argocd app sync geargo-app
```

---

## 🎯 Step 8: Configure Service Discovery and Ingress

### 8.1 Install AWS Load Balancer Controller (if not already installed)

```bash
# Add EKS Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 8.2 Install NGINX Ingress Controller (Optional)

```bash
# Add NGINX Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

---

## ✅ Step 9: Verification and Testing

### 9.1 Verify All Components

```bash
# Check all namespaces
kubectl get namespaces

# Check pods in each namespace
kubectl get pods -n argocd
kubectl get pods -n geargo
kubectl get pods -n monitoring

# Check services
kubectl get svc --all-namespaces

# Check ingress
kubectl get ingress --all-namespaces
```

### 9.2 Test Application Endpoints

```bash
# Get Geargo service URL
kubectl get svc -n geargo

# Test application (replace with your LoadBalancer URL)
curl http://<geargo-loadbalancer-url>

# Check ArgoCD
kubectl get svc -n argocd

# Check Grafana
kubectl get svc -n monitoring | grep grafana
```

### 9.3 Verify Monitoring

```bash
# Check Prometheus targets
# Access Prometheus UI and go to Status > Targets

# Check Grafana dashboards
# Access Grafana UI and verify pre-configured dashboards are available

# Verify Node Exporter metrics
kubectl get pods -n monitoring | grep node-exporter
```

---

## 🔧 Step 10: Post-Installation Configuration

### 10.1 Configure Grafana Dashboards

1. Access Grafana UI
2. Navigate to Dashboards > Browse
3. Import pre-configured dashboards:
   - Kubernetes Cluster Monitoring
   - Node Exporter Full
   - Pod Monitoring

### 10.2 Set Up Prometheus Alerts (Optional)

```bash
# Create alert rules
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: geargo-alerts
  namespace: monitoring
spec:
  groups:
  - name: geargo
    rules:
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
EOF
```

### 10.3 Configure ArgoCD Notifications (Optional)

```bash
# Enable notifications in ArgoCD
kubectl patch configmap argocd-notifications-cm -n argocd --type merge -p '{"data":{"service.slack":"webhookURL: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"}}'
```

---

## 📝 Summary

After completing these steps, you should have:

✅ **EKS Cluster** - Configured and accessible  
✅ **ArgoCD** - Installed and accessible via UI  
✅ **Geargo Application** - Deployed via Helm  
✅ **Prometheus** - Monitoring stack installed  
✅ **Grafana** - Visualization and dashboards ready  
✅ **Node Exporter** - Node metrics collection active  
✅ **GitOps** - ArgoCD managing application deployments  

---

## 🆘 Troubleshooting

### Common Issues

1. **Pods not starting:**
   ```bash
   kubectl describe pod <pod-name> -n <namespace>
   kubectl logs <pod-name> -n <namespace>
   ```

2. **ArgoCD sync issues:**
   ```bash
   argocd app get geargo-app
   argocd app logs geargo-app
   ```

3. **Prometheus not scraping:**
   ```bash
   kubectl get servicemonitor -n monitoring
   kubectl get prometheus -n monitoring
   ```

4. **LoadBalancer not created:**
   ```bash
   # Check AWS Load Balancer Controller logs
   kubectl logs -n kube-system deployment/aws-load-balancer-controller
   ```

---

## 🔗 Useful Commands Reference

```bash
# Get all resources
kubectl get all --all-namespaces

# Port forward to services
kubectl port-forward svc/<service-name> -n <namespace> <local-port>:<service-port>

# View logs
kubectl logs -f <pod-name> -n <namespace>

# Describe resources
kubectl describe <resource-type> <resource-name> -n <namespace>

# Helm commands
helm list -A
helm status <release-name> -n <namespace>
helm uninstall <release-name> -n <namespace>

# ArgoCD commands
argocd app list
argocd app get <app-name>
argocd app sync <app-name>
```

---

## 📚 Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Geargo Helm Chart](https://github.com/fred4impact/geargo-helm)

---

**Last Updated:** 2025-01-27

