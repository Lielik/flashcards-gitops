# 🚀 Flashcards Application - GitOps Repository

This repository contains the GitOps configuration for the Flashcards learning platform, managing application deployment through ArgoCD on Amazon EKS.

## 📋 Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Architecture](#architecture)
- [Deployment Components](#deployment-components)
- [Secret Management](#secret-management)
- [Monitoring & Observability](#monitoring--observability)
- [Deployment Process](#deployment-process)

---

## 🎯 Overview

This GitOps repository implements a declarative, version-controlled approach to Kubernetes deployments using ArgoCD. The repository manages:

- **Application Deployment**: Flashcards REST API and NGINX frontend
- **Database Management**: MongoDB with persistent storage
- **Platform Services**: cert-manager for TLS certificate automation
- **Secret Management**: Sealed Secrets for secure credential handling
- **Monitoring**: Prometheus ServiceMonitor for metrics collection

### Key Features

- **GitOps Workflow**: All changes deployed through Git commits
- **Automated Sync**: ArgoCD automatically syncs cluster state with Git
- **Helm-Based**: Uses Helm charts for templating and configuration management
- **Security First**: Encrypted secrets using Sealed Secrets
- **High Availability**: Multiple replicas for API and NGINX components
---

## 📁 Repository Structure

```
.
├── flashcards/                    # Main application Helm chart
│   ├── Chart.yaml                 # Chart metadata and dependencies
│   ├── Chart.lock                 # Dependency lock file
│   ├── values.yaml                # Default configuration values
│   ├── charts/                    # Subchart dependencies
│   │   └── mongodb-18.0.7.tgz    # MongoDB Helm chart
│   └── templates/                 # Kubernetes manifests
│       ├── api-deployment.yaml    # Flask API deployment
│       ├── api-service.yaml       # API service definition
│       ├── nginx-deployment.yaml  # NGINX deployment
│       ├── nginx-service.yaml     # NGINX service definition
│       ├── nginx-configmap.yaml   # NGINX configuration
│       ├── ingress.yaml           # Ingress resource for external access
│       ├── mongo-uri-sealed.yaml  # Sealed secret for MongoDB URI
│       ├── mongo-auth-sealed.yaml # Sealed secret for MongoDB credentials
│       └── servicemonitor.yaml    # Prometheus metrics scraping config
│
└── platform/                      # Platform-level services
    └── cert-manager/              # Certificate management
        ├── Chart.yaml             # cert-manager chart metadata
        ├── values.yaml            # cert-manager configuration
        └── templates/
            └── clusterissuer.yaml # Let's Encrypt issuer configuration
```
---

## ✅ Prerequisites

Before working with this repository, ensure you have:

- **EKS Cluster**: Running Kubernetes cluster (provisioned via infrastructure repo)
- **ArgoCD**: Installed and configured on the cluster
- **kubectl**: Configured with cluster access
- **kubeseal**: For creating new sealed secrets (if needed)
- **Helm**: For local testing and validation

### Verify Prerequisites

```bash
# Check cluster access
kubectl get nodes

# Verify ArgoCD is running
kubectl get pods -n argocd

# Check sealed-secrets controller
kubectl get pods -n kube-system | grep sealed-secrets
```
---
## 🏗 Architecture

### Application Components

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
└────────────────────────┬────────────────────────────────────┘
                         │
                    ┌────▼─────┐
                    │  Ingress │  (NGINX Ingress Controller)
                    │  Controller│  + TLS Termination
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         ┌────▼────┐          ┌────▼────┐
         │  NGINX  │          │  NGINX  │  (Replicas: 2)
         │  Pod 1  │          │  Pod 2  │  Serves static files
         └────┬────┘          └────┬────┘  Routes to API
              │                     │
              └──────────┬──────────┘
                         │
                    ┌────▼─────┐
                    │   API    │
                    │ Service  │
                    └────┬─────┘
                         │
              ┌──────────┴──────────┐
              │                     │
         ┌────▼────┐          ┌────▼────┐
         │  Flask  │          │  Flask  │  (Replicas: 2)
         │  API 1  │          │  API 2  │  REST endpoints
         └────┬────┘          └────┬────┘  /metrics for monitoring
              │                     │
              └──────────┬──────────┘
                         │
                    ┌────▼─────┐
                    │ MongoDB  │
                    │ Service  │
                    └────┬─────┘
                         │
                    ┌────▼─────┐
                    │ MongoDB  │  (StatefulSet)
                    │   Pod    │  Persistent storage (1Gi)
                    └──────────┘
```

### Deployment Flow

```
Developer Push → GitLab → ArgoCD Detection → Auto-Sync → Kubernetes Apply
     ↓
  Git Commit
     ↓
values.yaml or Chart.yaml changed
     ↓
ArgoCD polls repository (3min interval)
     ↓
Detects difference between Git and Cluster
     ↓
Applies changes (automated sync enabled)
     ↓
Health checks verify deployment
```
---
## 🔧 Deployment Components

### 1. Flashcards Application Chart

**Location**: `flashcards/`

An umbrella Helm chart that deploys the complete application stack.

#### Components:

**Flask API** (`templates/api-deployment.yaml`)
- **Replicas**: 2 for high availability
- **Image**: Pulled from AWS ECR
- **Environment Variables**:
  - `PORT`: 5000
  - `DEBUG`: false
  - `MONGO_URI`: From sealed secret
- **Health Checks**:
  - Liveness probe: `/health` endpoint
  - Readiness probe: `/health` endpoint
- **Resources**:
  - Requests: 100m CPU, 128Mi memory
  - Limits: 500m CPU, 512Mi memory

**NGINX** (`templates/nginx-deployment.yaml`)
- **Replicas**: 2 for load distribution
- **Purpose**: 
  - Serves static files (index.html)
  - Reverse proxy to Flask API
  - SSL/TLS termination support
- **Configuration**: Mounted from ConfigMap
- **Resources**:
  - Requests: 50m CPU, 64Mi memory
  - Limits: 200m CPU, 128Mi memory

**MongoDB Database** (subchart from Bitnami)
- **Type**: StatefulSet for data persistence
- **Storage**: 1Gi persistent volume (gp2 storage class)
- **Authentication**: Enabled with credentials from sealed secret
- **Resources**:
  - Requests: 250m CPU, 512Mi memory
  - Limits: 1000m CPU, 2Gi memory

**Ingress** (`templates/ingress.yaml`)
- **Controller**: NGINX Ingress
- **TLS**: cert-manager integration for Let's Encrypt certificates
- **Domain**: flashcards.ddns.net
- **Annotations**:
  - Force SSL redirect
  - cert-manager cluster issuer

### 2. Platform Services

**cert-manager** (`platform/cert-manager/`)
- **Purpose**: Automated TLS certificate management
- **Provider**: Let's Encrypt (production)
- **Challenge Type**: HTTP-01
- **Email**: liel.frank@gmail.com

**ClusterIssuer Configuration**:
```yaml
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - http01:
          ingress:
            class: alb
```
---
## 🔐 Secret Management

This project uses **Sealed Secrets** for GitOps-friendly secret management. Secrets are encrypted and can be safely stored in Git.

### Sealed Secrets in Repository

1. **MongoDB URI** (`mongo-uri-sealed.yaml`)
   - Contains: Connection string with credentials
   - Format: `mongodb://user:password@host:port/database`
   - Used by: Flask API

2. **MongoDB Authentication** (`mongo-auth-sealed.yaml`)
   - Contains:
     - `mongodb-root-password`: Root user password
     - `mongodb-passwords`: User passwords
     - `mongodb-username`: Application username
     - `mongodb-database`: Database name
   - Used by: MongoDB StatefulSet


### Security Best Practices

- ✅ **Only sealed secrets** are committed to Git
- ✅ **Original secrets** are never committed
- ✅ **Public certificate** is safe to commit
- ❌ **Private key** stays only on the cluster
- ✅ **Rotation**: Create new sealed secrets when credentials change
---
## 📊 Monitoring & Observability

### Prometheus Integration

**ServiceMonitor** (`templates/servicemonitor.yaml`)

Configures Prometheus to scrape metrics from the Flask API:

```yaml
spec:
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

### Available Metrics

The Flask API exposes the following Prometheus metrics:

- `flashcard_api_requests_total`: Total API requests (labeled by method, endpoint, status)
- `flashcard_api_request_duration_seconds`: Request duration histogram
- `flashcard_decks_total`: Current number of decks
- `flashcard_cards_total`: Current number of cards
- `flashcard_cards_mastered`: Number of mastered cards
- `flashcard_users_total`: Total number of users

### Grafana Dashboards

Custom dashboards can be created to visualize:
- API request rates and error rates
- Response time percentiles (p50, p95, p99)
- Database connection pool usage
- Learning progress metrics (cards mastered over time)
---
## 🚀 Deployment Process

### Initial Deployment

1. **Prerequisites Check**:
   ```bash
   # Ensure ArgoCD is running
   kubectl get pods -n argocd
   
   # Verify sealed-secrets controller
   kubectl get pods -n kube-system | grep sealed-secrets
   ```

2. **ArgoCD Application Creation**:
   
   ArgoCD applications are automatically created via ApplicationSet in the infrastructure repository. The ApplicationSet creates two applications:

   - `cert-manager` (wave: 0, deployed first)
   - `flashcards` (wave: 2, deployed after platform services)

3. **Monitor Deployment**:
   ```bash
   # Watch ArgoCD sync status
   kubectl get applications -n argocd -w
   
   # Check application pods
   kubectl get pods -n apps
   
   # View ArgoCD UI
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   # Access: https://localhost:8080
   ```
---
### Making Changes

1. **Update Configuration**:
   ```bash
   # Edit values.yaml for configuration changes
   vim flashcards/values.yaml
   ```

2. **Update Application Version**:
   ```bash
   # Edit Chart.yaml to update appVersion
   vim flashcards/Chart.yaml
   
   # Change appVersion to new tag from ECR:
   # appVersion: "v1.0.12"
   ```

3. **Commit and Push**:
   ```bash
   git add .
   git commit -m "Update flashcards API to v1.0.12"
   git push origin main
   ```

4. **ArgoCD Auto-Sync**:
   
   ArgoCD will automatically detect changes within ~3 minutes and apply them. Sync policies configured:
   - `prune: true` - Remove resources no longer in Git
   - `selfHeal: true` - Correct manual changes to match Git
   - `CreateNamespace: true` - Auto-create namespaces if missing
---
### CI/CD Integration

The Jenkins pipeline in the application repository automatically updates this GitOps repository:

```groovy
// Jenkins pipeline "Deploy" stage:
1. Build and tag Docker image (e.g., v1.0.12)
2. Push image to ECR
3. Clone this GitOps repository
4. Update flashcards/Chart.yaml appVersion field
5. Commit and push changes
6. ArgoCD detects change and deploys
```
---
### Useful Commands

```bash
# View all resources in apps namespace
kubectl get all -n apps

# Watch pod status in real-time
kubectl get pods -n apps -w

# Port-forward to API for local testing
kubectl port-forward -n apps svc/flashcards-app-app-api 5000:5000

# Port-forward to MongoDB for debugging
kubectl port-forward -n apps svc/flashcards-mongodb 27017:27017

# Force ArgoCD to sync immediately
argocd app sync flashcards-app --force

# View ArgoCD application tree
argocd app get flashcards-app --show-operation

# Delete and recreate application (use with caution!)
kubectl delete application flashcards-app -n argocd
# Then re-apply from infrastructure repo
```
---
### Getting Help

If issues persist:

1. **Check ArgoCD UI**: Visual representation of deployment status
2. **Review Application Logs**: Often contain error details
3. **Verify Prerequisites**: Ensure all platform components are healthy
4. **Compare with Working State**: Use Git history to identify changes
---
**Repository**: flashcards-gitops  
**Cluster**: portfolio-dev-eks  
**Namespace**: apps  
**ArgoCD URL**: Access via port-forward or LoadBalancer  
**Application URL**: https://flashcards.ddns.net