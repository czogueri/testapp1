# 🚀 Production-Grade Kubernetes CI/CD Pipeline

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34.3-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-v2.10-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Jenkins](https://img.shields.io/badge/Jenkins-LTS-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](https://www.jenkins.io/)
[![Nginx](https://img.shields.io/badge/Nginx-Ingress-009639?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)
[![GitHub](https://img.shields.io/badge/GitHub-GitOps-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/)

> A fully operational, production-grade CI/CD pipeline deployed on a self-managed 3-node Kubernetes cluster — built, debugged, and maintained from the ground up.

---

## 📸 Live Project Screenshots

### ArgoCD — Application Healthy & Synced
![ArgoCD Dashboard](screenshots/argocd-dashboard.png)
*ArgoCD showing `myapp` in a Healthy and Synced state, automatically tracking the GitHub repository and last synced 15 minutes ago.*

---

### Kubernetes — 3 Nginx Pods Running Across All Nodes
![Pods Running](screenshots/pods-running.png)
*All 3 nginx pods running across `pve2`, `minty`, and `ubuntu-vm` nodes — one pod per node enforced by Pod Anti-Affinity rules for high availability.*

---

### Jenkins — CI Pipeline Ready
![Jenkins Dashboard](screenshots/jenkins-dashboard.png)
*Jenkins CI running at `jenkins.local:9090` with the `CI-CD-pipeline` job configured and ready to trigger on every GitHub push via webhook.*

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Workflow                        │
│                                                                 │
│   Edit Code  ──►  Git Push  ──►  GitHub Repository             │
└─────────────────────────┬───────────────────────────────────────┘
                          │
              ┌───────────▼───────────┐
              │      GitHub Webhook   │
              │   (triggers on push)  │
              └───────────┬───────────┘
                          │
          ┌───────────────▼──────────────┐
          │         Jenkins CI           │
          │   • Validates K8s manifests  │
          │   • Runs pipeline stages     │
          │   • Reports build status     │
          └───────────────┬──────────────┘
                          │
          ┌───────────────▼──────────────┐
          │          ArgoCD CD           │
          │   • Watches GitHub repo      │
          │   • Detects drift/changes    │
          │   • Auto-syncs to cluster    │
          └───────────────┬──────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   3-Node K3s Cluster                            │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  ubuntu-vm   │  │    minty     │  │     pve2     │         │
│  │ Control Plane│  │    Worker    │  │    Worker    │         │
│  │              │  │              │  │              │         │
│  │ nginx-pod-1  │  │ nginx-pod-2  │  │ nginx-pod-3  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                 │
│         MetalLB LoadBalancer  |  Nginx Ingress Controller      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Stack & Tools

| Category | Tool | Purpose |
|---|---|---|
| **Container Orchestration** | K3s (Kubernetes v1.34.3) | Manages containerized workloads across 3 nodes |
| **Continuous Deployment** | ArgoCD v2.10 | GitOps engine — syncs cluster state to Git |
| **Continuous Integration** | Jenkins LTS | Validates manifests, runs pipeline on every push |
| **Load Balancing** | MetalLB v0.15.3 | Provides external IPs for bare-metal cluster |
| **Ingress** | Nginx Ingress Controller | Routes external HTTP/S traffic to services |
| **Source Control** | GitHub | Single source of truth for all manifests |
| **App Runtime** | Nginx | Sample application deployed via pipeline |
| **Package Manager** | Helm | Used for service deployments |

---

## 🖥️ Cluster Topology

| Node | Role | IP | OS |
|---|---|---|---|
| `ubuntu-vm` | Control Plane | 192.168.1.181 | Ubuntu 24.04 |
| `minty` | Worker | 192.168.1.210 | Linux |
| `pve2` | Worker | 192.168.1.204 | Proxmox VE |

### Network IP Allocation (MetalLB Pool: 192.168.1.240–250)

| Service | External IP | Port |
|---|---|---|
| Nginx Ingress Controller | 192.168.1.245 | 80, 443 |
| ArgoCD | 192.168.1.243 | 80 |
| Jenkins | 192.168.1.246 | 9090 |

---

## ⚙️ CI/CD Pipeline Flow

### 1. Continuous Integration — Jenkins

Every `git push` to the `main` branch triggers Jenkins via GitHub Webhook:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/czogueri/CI-CD-pipeline.git'
            }
        }
        stage('Validate Manifests') {
            steps {
                sh 'kubectl apply --dry-run=client -f nginx-deployment.yaml'
                sh 'kubectl apply --dry-run=client -f nginx-service.yaml'
            }
        }
        stage('Deploy via ArgoCD Sync') {
            steps {
                sh 'echo "ArgoCD will auto-sync from GitHub"'
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded - ArgoCD will deploy changes' }
        failure { echo 'Pipeline failed - check logs' }
    }
}
```

### 2. Continuous Deployment — ArgoCD (GitOps)

ArgoCD continuously watches the GitHub repository and automatically syncs any changes to the cluster:

```bash
# ArgoCD application configuration
argocd app create myapp \
  --repo https://github.com/czogueri/CI-CD-pipeline.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace testapp1 \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

**Key GitOps principles implemented:**
- ✅ Git as single source of truth
- ✅ Automated drift detection and correction
- ✅ Auto-pruning of removed resources
- ✅ Self-healing — reverts manual cluster changes to match Git

---

## 🌐 Application Deployment

The sample Nginx application is deployed with **high availability** across all 3 nodes using Pod Anti-Affinity:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: testapp1
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: nginx-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx:latest
        resources:
          limits:
            cpu: "1000m"
            memory: "2Gi"
          requests:
            cpu: "500m"
            memory: "1Gi"
```

**Result:** One pod per node — if any node goes down, the app stays online on the remaining nodes.

### Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: testapp1
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

---

## 🔧 Real-World Problems Solved

This project involved diagnosing and resolving production-grade Kubernetes issues:

### 1. Nginx Ingress IP Change After Reboot
**Problem:** MetalLB reassigned a different IP to the Ingress controller after restart, breaking `myapp.local` DNS resolution.

**Fix:** Assigned a static IP via MetalLB annotation:
```bash
kubectl annotate svc ingress-nginx-controller \
  -n ingress-nginx \
  metallb.universe.tf/loadBalancerIPs=192.168.1.245
```

---

### 2. ArgoCD TLS Handshake Failure
**Problem:** ArgoCD repo-server failing with TLS handshake errors when connecting to GitHub despite outbound internet working correctly.

**Fix:** Patched ArgoCD server to run in insecure mode, disabled repo-server TLS via ConfigMap, and connected repo via Kubernetes secret:
```bash
kubectl create secret generic github-repo -n argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/czogueri/CI-CD-pipeline.git \
  --from-literal=username=<username> \
  --from-literal=password=<token>

kubectl label secret github-repo -n argocd \
  argocd.argoproj.io/secret-type=repository
```

---

## 📁 Repository Structure

```
CI-CD-pipeline/
├── screenshots/
│   ├── argocd-dashboard.png     # ArgoCD healthy & synced
│   ├── pods-running.png         # 3 pods across 3 nodes
│   └── jenkins-dashboard.png    # Jenkins CI pipeline
├── nginx-deployment.yaml        # App deployment with HA anti-affinity
├── nginx-service.yaml           # ClusterIP service
├── nginx-ingress.yaml           # Ingress routing rule
└── Jenkinsfile                  # CI pipeline definition
```

---

## 🚦 How to Deploy

### Prerequisites
- 3-node K3s cluster
- MetalLB configured with IP pool
- Nginx Ingress Controller installed
- ArgoCD installed and running
- Jenkins installed and running
- GitHub repository with manifests

### Deploy the Application

```bash
# 1. Clone the repo
git clone https://github.com/czogueri/CI-CD-pipeline.git
cd CI-CD-pipeline # repo folder

# 2. Create namespace
kubectl create namespace testapp1

# 3. Register repo with ArgoCD
argocd repo add https://github.com/czogueri/CI-CD-pipeline.git \
  --username <username> \
  --password <token> \
  --insecure-skip-server-verification

# 4. Create ArgoCD application
argocd app create myapp \
  --repo https://github.com/czogueri/CI-CD-pipeline.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace testapp1 \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# 5. Verify deployment
kubectl get pods -n testapp1 -o wide
argocd app get myapp
```

### Test the Pipeline

```bash
# Make a change
nano nginx-deployment.yaml  # e.g. change replica count

# Push to GitHub
git add .
git commit -m "Update replicas"
git push origin main

# Watch ArgoCD auto-deploy
kubectl get pods -n testapp1 -w
```

---

## 📊 Key Metrics

| Metric | Value |
|---|---|
| Cluster Nodes | 3 |
| App Replicas | 3 (one per node) |
| Deployment Method | GitOps (ArgoCD) |
| CI Trigger | GitHub Webhook → Jenkins |
| Average Sync Time | < 3 minutes |
| Ingress IP | Static (MetalLB annotated) |

---

## 📚 Skills Demonstrated

- ✅ **Kubernetes** — Multi-node cluster management, StatefulSets, Deployments, PVCs, Ingress, Services
- ✅ **GitOps** — ArgoCD, drift detection, auto-sync, self-healing
- ✅ **CI/CD** — Jenkins pipeline, GitHub webhooks, automated validation
- ✅ **Networking** — MetalLB, Nginx Ingress, hostname-based routing, static IP assignment
- ✅ **Troubleshooting** — OOM kills, pod scheduling, TLS issues, disk pressure, PV node affinity
- ✅ **Security** — Kubernetes secrets management, GitHub PAT authentication
- ✅ **High Availability** — Pod anti-affinity, multi-node distribution, resource limits

---

## 👤 Author

**czogueri**
- GitHub: [@czogueri](https://github.com/czogueri)

---

*Built on a self-managed bare-metal Kubernetes cluster — no managed cloud services.*
