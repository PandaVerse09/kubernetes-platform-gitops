# Kubernetes Platform GitOps

The declarative single source of truth for all Kubernetes resources managed by Argo CD.

## Repository Structure
```
├── charts/
│   └── application/          # Helm v3 parameterized templates
│       ├── Chart.yaml
│       ├── values.yaml       # Base defaults
│       └── templates/        # Deployment, Service, Ingress, HPA, PDB, NetworkPolicy, Probes
├── environments/
│   ├── dev/values.yaml       # Local KIND cluster overrides
│   └── prod/values.yaml      # AWS EKS cluster overrides
├── argocd/
│   ├── application-dev.yaml  # Argo CD Application manifest for dev
│   └── application-prod.yaml # Argo CD Application manifest for prod
├── monitoring/
│   └── values-prometheus.yaml# Helm values for Prometheus Operator + Grafana
└── kind-config.yaml          # Local multi-port KIND cluster definition
```

## Quick Start (Local KIND Cluster)

### 1. Create KIND Cluster
```bash
kind create cluster --config kind-config.yaml --name platform-lab
```

### 2. Install NGINX Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### 3. Install Argo CD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Deploy Application via GitOps
```bash
kubectl apply -f argocd/application-dev.yaml
```
