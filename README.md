# GitOps Config — ArgoCD Deployment Repository

![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-KIND-blue)
![Helm](https://img.shields.io/badge/Helm-Chart-0F1689)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)

This repository is the **single source of truth** for all Kubernetes deployments.
ArgoCD continuously watches this repo and reconciles the cluster to match what is declared here.

> **Application Repo:** [sample-app](https://github.com/kiran-kumatkar/sample-app)
> **Monitoring Config Repo:** [k8s-monitoring](https://github.com/kiran-kumatkar/k8s-monitoring)

---

## What is GitOps?

- **Git is the single source of truth** for all deployment configuration
- **Changes happen via git commits** — full audit trail and peer review
- **ArgoCD automatically reconciles** cluster state to match Git
- **Self-healing** — manual cluster changes are automatically reverted within 3 minutes
- **No kubectl in CI** — the cluster pulls from Git, credentials never leave the cluster

---

## Repository Structure

```
gitops-config/
├── apps/                              # ArgoCD Application definitions
│   ├── root-app.yaml                  # App of Apps — bootstraps everything
│   ├── gitops-project.yaml            # ArgoCD AppProject — RBAC boundaries
│   ├── dev-app.yaml                   # Dev environment Application
│   ├── staging-app.yaml               # Staging environment Application
│   ├── monitoring-app.yaml            # kube-prometheus-stack Application
│   └── monitoring-extras-app.yaml     # ServiceMonitor + PrometheusRules Application
├── environments/
│   ├── dev/                           # Dev — plain Kubernetes YAML
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── staging/
│       └── values.yaml                # Staging — Helm value overrides
└── helm/
    └── sample-app/                    # Helm chart (used by staging)
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── configmap.yaml
```

---

## Applications Managed by ArgoCD

| ArgoCD App | Source Repo | Path | Namespace | Method |
|------------|-------------|------|-----------|--------|
| `sample-app-dev` | gitops-config | `environments/dev` | dev | Plain YAML |
| `sample-app-staging` | gitops-config | `helm/sample-app` | staging | Helm Chart |
| `kube-prometheus-stack` | k8s-monitoring | `helm/values/` | monitoring | Helm Chart |
| `monitoring-extras` | k8s-monitoring | `.` (root) | monitoring | Plain YAML |

---

## Environments

| Environment | Namespace | Managed By | Replicas |
|-------------|-----------|------------|----------|
| Dev | `dev` | Plain Kubernetes YAML | 3 |
| Staging | `staging` | Helm Chart with value overrides | 3 |
| Monitoring | `monitoring` | kube-prometheus-stack Helm chart | — |

---

## ArgoCD Concepts Demonstrated

| Concept | Implementation |
|---------|---------------|
| **App of Apps** | `root-app.yaml` bootstraps all Applications from `apps/` folder |
| **AppProject** | RBAC boundaries — allowed repos, namespaces, resource types |
| **Automated sync** | ArgoCD syncs on every git push automatically |
| **Self-heal** | Manual kubectl changes reverted automatically |
| **Prune** | Resources deleted from Git are deleted from cluster |
| **Sync waves** | AppProject created before Applications (`wave: -1`) |
| **Helm integration** | Staging and monitoring use Helm charts |
| **Multiple sources** | monitoring-app uses Helm chart + values from separate repo |
| **Multi-environment** | Dev (plain YAML) vs Staging (Helm) — same app, different approach |

---

## GitOps Flow — Application Deployment

```
Developer pushes code to sample-app repo
              ↓
GitHub Actions builds Docker image
              ↓
Image pushed to DockerHub (kirankumatkar217/sample-app:v1.0.0-<sha>)
              ↓
GitHub Actions commits updated image tag to THIS repo
              ↓
ArgoCD detects new commit (polls every 3 minutes)
              ↓
New pods running with updated image — zero manual steps
```

## GitOps Flow — Monitoring Changes

```
Change pushed to k8s-monitoring repo
              ↓
ArgoCD detects change
              ↓
kube-prometheus-stack syncs new Helm values
monitoring-extras syncs updated rules and monitors
              ↓
Prometheus picks up new rules automatically
              ↓
Alertmanager routes firing alerts to email
```

---

## Bootstrap — Fresh Cluster Setup

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Create monitoring email secret (not stored in Git)
kubectl create namespace monitoring
kubectl create secret generic alertmanager-email-secret \
  --from-literal=smtp_password='your-gmail-app-password' \
  --namespace monitoring

# 3. Apply AppProject first
kubectl apply -f apps/gitops-project.yaml

# 4. Apply root-app — ArgoCD manages everything from this point
kubectl apply -f apps/root-app.yaml
```

---

## Access UIs

```bash
# ArgoCD
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "nodePort": 30080}]}}'
# https://localhost:9080

# Dev app
kubectl port-forward svc/sample-app 8888:80 -n dev
# http://localhost:8888

# Staging app
kubectl port-forward svc/sample-app 8889:80 -n staging
# http://localhost:8889

# Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# http://localhost:3000

# Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# http://localhost:9090

# Alertmanager
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# http://localhost:9093
```

---

## Useful ArgoCD Commands

```bash
argocd app list
argocd app get sample-app-dev
argocd app sync sample-app-dev
argocd app history sample-app-dev
argocd app rollback sample-app-dev
argocd app get kube-prometheus-stack --hard-refresh
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


## Related Repositories

| Repository | Purpose |
|------------|---------|
| [sample-app](https://github.com/kiran-kumatkar/sample-app) | Flask app source code, Dockerfile, GitHub Actions CI |
| [k8s-monitoring](https://github.com/kiran-kumatkar/k8s-monitoring) | Prometheus values, alert rules, ServiceMonitors |

