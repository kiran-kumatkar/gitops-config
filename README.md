# GitOps Config — ArgoCD Deployment Repository

![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-KIND-blue)
![Helm](https://img.shields.io/badge/Helm-Chart-0F1689)
![GitHub Actions](https://img.shields.io/badge/CI-GitHub_Actions-2088FF)

This repository is the **single source of truth** for all Kubernetes deployments.
ArgoCD continuously watches this repo and reconciles the cluster to match what is declared here.

> **Application Repository:** [sample-app](https://github.com/kiran-kumatkar/sample-app)

---

## What is GitOps?

GitOps is an operational framework where:

- **Git is the single source of truth** for all deployment configuration
- **Changes happen via git commits** — full audit trail and history
- **ArgoCD automatically reconciles** the cluster state to match Git
- **Self-healing** — any manual change to the cluster is automatically reverted
- **No kubectl in CI** — the cluster pulls from Git, credentials never leave the cluster

---

## Repository Structure

```
gitops-config/
├── apps/                          # ArgoCD Application definitions
│   ├── root-app.yaml              # App of Apps — bootstraps everything
│   ├── gitops-project.yaml        # ArgoCD AppProject — RBAC and boundaries
│   ├── dev-app.yaml               # Dev environment Application
│   └── staging-app.yaml           # Staging environment Application
├── environments/
│   ├── dev/                       # Dev — plain Kubernetes YAML manifests
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── staging/
│       └── values.yaml            # Staging — Helm value overrides only
└── helm/
    └── sample-app/                # Helm chart (used by staging)
        ├── Chart.yaml
        ├── values.yaml            # Base default values
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── configmap.yaml
```

---

## Environments

| Environment | Namespace | Managed By | Replicas |
|-------------|-----------|------------|----------|
| Dev | `dev` | Plain Kubernetes YAML | 3 |
| Staging | `staging` | Helm Chart | 3 |

Both environments run the same Docker image — only values differ per environment.

---

## ArgoCD Concepts Demonstrated

| Concept | Where |
|---------|-------|
| **App of Apps** | `apps/root-app.yaml` — one manifest bootstraps all Applications |
| **AppProject** | `apps/gitops-project.yaml` — RBAC, allowed repos and namespaces |
| **Automated sync** | ArgoCD syncs on every git push automatically |
| **Self-heal** | Manual kubectl changes reverted within 3 minutes |
| **Prune** | Resources deleted from Git are deleted from cluster |
| **Sync waves** | AppProject created before Applications (`wave: -1`) |
| **Helm integration** | Staging uses Helm chart with per-environment value overrides |
| **Multi-environment** | Dev uses plain YAML, Staging uses Helm — same app, different approach |

---

## GitOps Flow

```
git push to sample-app repo
          ↓
GitHub Actions builds Docker image
          ↓
Image pushed to DockerHub (kirankumatkar217/sample-app:v1.0.0-<sha>)
          ↓
GitHub Actions commits updated image tag to THIS repo
          ↓
ArgoCD detects new commit (polls every 3 minutes)
          ↓
ArgoCD syncs dev and staging namespaces
          ↓
New pods running with updated image
```

---

## Bootstrap — Fresh Cluster Setup

Everything can be bootstrapped with just 3 commands:

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Apply AppProject first (Applications reference this)
kubectl apply -f apps/gitops-project.yaml

# 3. Apply root-app — ArgoCD manages everything from this point
kubectl apply -f apps/root-app.yaml
```

ArgoCD will automatically discover and create:
- `sample-app-dev` → deploys to `dev` namespace
- `sample-app-staging` → deploys to `staging` namespace

---

## Access ArgoCD UI

```bash
# Patch service to NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "nodePort": 30080}]}}'

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Open in browser
# https://localhost:9080
# Username: admin
```

---

## Useful ArgoCD Commands

```bash
# List all applications
argocd app list

# Get application details and sync status
argocd app get sample-app-dev
argocd app get sample-app-staging

# Manually trigger sync
argocd app sync sample-app-dev

# View deployment history
argocd app history sample-app-dev

# Rollback to previous version
argocd app rollback sample-app-dev

# View all managed resources
argocd app resources sample-app-dev
```

---

## Verify Deployments

```bash
# Check pods in dev
kubectl get pods -n dev

# Check pods in staging
kubectl get pods -n staging

# Test dev app
kubectl port-forward svc/sample-app 8888:80 -n dev
curl http://localhost:8888
# {"message": "GitOps demo app", "version": "v1.0.0", "environment": "dev"}

# Test staging app
kubectl port-forward svc/sample-app 8889:80 -n staging
curl http://localhost:8889
# {"message": "GitOps demo app", "version": "v1.0.0", "environment": "staging"}
```

---

## Related Repository

| Repository | Purpose |
|------------|---------|
| [sample-app](https://github.com/kiran-kumatkar/sample-app) | Application source code, Dockerfile, GitHub Actions CI pipeline |
