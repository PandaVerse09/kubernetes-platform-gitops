# Enterprise Kubernetes Platform Engineering & GitOps System

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30-blue?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Terraform](https://img.shields.io/badge/Terraform-v1.15-purple?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![ArgoCD](https://img.shields.io/badge/GitOps-Argo%20CD-orange?logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Prometheus](https://img.shields.io/badge/Observability-Prometheus-red?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Visuals-Grafana-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Trivy](https://img.shields.io/badge/Security-Aqua%20Trivy-blue)](https://trivy.dev/)
[![Docker](https://img.shields.io/badge/Containers-Docker%20Desktop-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

An end-to-end, production-grade cloud platform engineered with a modern GitOps delivery model, automated Infrastructure as Code (IaC), DevSecOps vulnerability gating, dynamic autoscaling, and Google SRE Golden Signal observability.

This repository serves as the **declarative single source of truth and control plane** for the entire platform.

---

## 🏛️ Architecture Overview

```mermaid
graph TB
    subgraph "1. Infrastructure as Code (Terraform)"
        VPC[AWS Multi-AZ VPC\nPublic & Private Subnets + NAT GW]
        ECR[Amazon ECR\nScan-on-Push & Lifecycle Policy]
        EKS[Amazon EKS v1.30\nManaged Node Groups + OIDC IRSA]
        VPC --> EKS
        ECR -.-> EKS
    end

    subgraph "2. Continuous Integration & Security (GitHub Actions)"
        DevCommit[Developer git push] --> CIWorkflow[GitHub Actions CI Pipeline]
        CIWorkflow --> UnitTests[Unit & Integration Tests / Jest]
        UnitTests --> TrivyScan[Aqua Trivy CVE Vulnerability Scanner]
        TrivyScan --> DockerBuild[Hardened Multi-Stage Docker Build]
        DockerBuild --> RegistryPush[Push Image to Registry]
        RegistryPush --> GitOpsPromotion[Automated SHA Tag GitOps Update]
    end

    subgraph "3. Continuous Delivery & Reconciliation (GitOps via Argo CD)"
        GitOpsRepo[(GitOps Repo: Helm Charts)]
        GitOpsPromotion -.->|Auto-commit / PR| GitOpsRepo
        ArgoCD[Argo CD Controller] -->|Watches Git & Reconciles| GitOpsRepo
        ArgoCD -->|Syncs Dev Namespace| DevEnv[Dev Environment\n2 Replicas + HPA]
        ArgoCD -->|Syncs Prod Namespace| ProdEnv[Prod Environment\n3 Replicas + Strict PDB]
    end

    subgraph "4. Ingress, Resilience & Observability"
        Users((End Users)) --> Ingress[NGINX Ingress Controller]
        Ingress --> AppPods[Application Pods / Node.js 20]
        AppPods --> ServiceMonitor[Prometheus ServiceMonitor]
        ServiceMonitor --> PrometheusServer[Prometheus Scraper]
        PrometheusServer --> GrafanaDashboards[Grafana RED Dashboards]
        AppPods -.->|CPU Utilization > 60%| HPAController[Horizontal Pod Autoscaler]
        HPAController -.->|Autoscales 2 -> 5 Replicas| AppPods
        AppPods -.->|Process Crash| KubeletRestart[Self-Healing Kubelet Auto-Restart]
    end
```

---

## 📦 The 3-Tier Platform Ecosystem

The platform is architected into three decoupled, single-responsibility repositories:

| Repository | GitHub Link | Purpose & Tech Stack |
| :--- | :--- | :--- |
| **Declarative GitOps (This Repo)** | [kubernetes-platform-gitops](https://github.com/PandaVerse09/kubernetes-platform-gitops) | Parameterized Helm v3 charts, environment values (`dev` & `prod`), Argo CD Application manifests, PDB, HPA, NetworkPolicies, and Prometheus stack configuration. |
| **Microservice Application** | [kubernetes-platform-app](https://github.com/PandaVerse09/kubernetes-platform-app) | Node.js 20 REST API, Liveness/Readiness probes, Prometheus RED metrics, Chaos testing endpoints, multi-stage non-root Dockerfile, and GitHub Actions CI. |
| **Cloud Infrastructure (IaC)** | [kubernetes-platform-infra](https://github.com/PandaVerse09/kubernetes-platform-infra) | Modular Terraform configurations for AWS VPC (multi-AZ, NAT Gateways), ECR with CVE scanning, and AWS EKS v1.30 managed cluster with OIDC for IAM Roles for Service Accounts (IRSA). |

---

## 📂 Repository Structure

```text
kubernetes-platform-gitops/
├── argocd/
│   ├── application-dev.yaml      # Argo CD Application manifest for Dev
│   └── application-prod.yaml     # Argo CD Application manifest for Prod
├── charts/
│   └── application/              # Reusable Helm v3 templates
│       ├── Chart.yaml
│       ├── values.yaml           # Base defaults
│       └── templates/            # Deployment, Service, Ingress, HPA, PDB, NetworkPolicy, ServiceMonitor
├── environments/
│   ├── dev/values.yaml           # Local cluster overrides (2 replicas, HPA, localhost ingress)
│   └── prod/values.yaml          # Production overrides (3 replicas, strict PDB, 70% HPA)
├── monitoring/
│   └── values-prometheus.yaml    # Helm values for Prometheus Operator + Grafana
└── kind-config.yaml              # Multi-port KIND cluster definition
```

---

## 🛡️ Key Enterprise Highlights

### 1. Shift-Left Security & Hardening
- **Vulnerability Scanning**: Automated Aqua Trivy static analysis scans container images for OS and application CVEs before push.
- **Non-Root Runtime**: Containers execute under a dedicated unprivileged user (`node:node`) with minimized attack surface based on Alpine Linux.
- **Zero-Trust Network Isolation**: Kubernetes `NetworkPolicy` objects restrict ingress traffic strictly to the application port and authorized namespaces.

### 2. Declarative GitOps Workflow
- **Single Source of Truth**: Cluster state is 100% defined in Git. No engineer or operator has manual write access (`kubectl apply`) to cluster state in production.
- **Drift Detection & Self-Healing**: Argo CD reconciles any unauthorized out-of-band changes back to the declared Git state within seconds.
- **Multi-Environment Promotion**: Independent environment configurations for `dev` (lightweight, rapid iteration) and `prod` (high availability with Pod Disruption Budgets and higher autoscaling limits).

### 3. High Availability & Resilience
- **Zero-Downtime Deployments**: Rolling updates managed via Kubernetes Deployment controllers paired with Pod Disruption Budgets (`minAvailable: 2`).
- **Dynamic Horizontal Autoscaling**: Kubernetes HPA dynamically provisions additional replicas (up to 5 in Dev, 10 in Prod) when synthetic or organic CPU thresholds exceed target utilization.
- **Autonomous Self-Healing**: Active `/health` (Liveness) and `/ready` (Readiness) probes ensure traffic is instantly routed away from failing containers while kubelet restarts the crashed instance.

### 4. SRE & Golden Signal Observability
- **RED Methodology**: Native instrumentation exposing Request Rate, Error counts, and Duration histograms (`http_request_duration_seconds`).
- **ServiceMonitor Integration**: Prometheus Operator automatically discovers endpoints and scrapes metrics every 15 seconds.
- **Centralized Dashboards**: Pre-integrated Grafana for real-time visualization of container resource saturation, network I/O, and HTTP request throughput.

---

## 📸 Platform Verification & Live Telemetry

### 1. Dynamic Horizontal Autoscaling (HPA Scaling to 5 Replicas)
*Real-time demonstration of the Kubernetes HPA scaling the deployment from 2 to 5 pods during simulated traffic surges:*
![Argo CD HPA Autoscaling](docs/images/argocd-hpa-scaling.png)

### 2. SRE Observability & 99.999% Availability SLO (Grafana)
*Production Grafana telemetry monitoring API server latency, error budgets, work queue saturation, and SLIs:*
![Grafana 99.999% SLO Dashboard](docs/images/grafana-slo-dashboard.jpg)

### 3. Declarative GitOps Application State (Argo CD)
*Automated continuous reconciliation and healthy sync status managed by Argo CD:*
![Argo CD GitOps Applications](docs/images/argocd-gitops-overview.png)

---

## 🖥️ Platform Endpoints & Local Access Guide

> ℹ️ **Note for GitHub Visitors:** The endpoints below run on your local machine (`localhost`) when executing this cluster locally. In a live AWS EKS deployment, these map to AWS Application Load Balancers (ALBs) and Amazon Route 53 DNS records (e.g., `api.production.pandaverse.dev`).

| Component | Local Endpoint | Port-Forward Command | Authentication & Secret Retrieval |
| :--- | :--- | :--- | :--- |
| **Application Web Dashboard** | `http://localhost/` | *(Exposed via NGINX Ingress on port 80)* | Interactive UI with live telemetry & Chaos Lab |
| **Argo CD Web UI** | `https://localhost:8081` | `kubectl port-forward svc/argocd-server -n argocd 8081:443` | User: `admin` • Secret: Run command below *(Retrieved from cluster)* |
| **Grafana Observability** | `http://localhost:3000` | `kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80` | User: `admin` • Secret: Run command below *(Retrieved from cluster)* |
| **Prometheus Scrape Engine** | `http://localhost/metrics` | *(Exposed via Ingress)* | Native Prometheus exposition format |
| **Health Probe** | `http://localhost/health` | *(Exposed via Ingress)* | Kubernetes Liveness Probe endpoint |

#### Retrieving Cluster Secrets Securely:
```bash
# Retrieve Argo CD admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Retrieve Grafana admin password:
kubectl -n monitoring get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

---

## 🎯 Live Demonstration Runbook

### Scenario A: Zero-Downtime Self-Healing (Chaos Test)
1. Open the Argo CD UI at `https://localhost:8081` and navigate to the `kubernetes-platform-app-dev` application.
2. Open the App Dashboard at `http://localhost/` and click **"Simulate Fatal Crash"** (or execute `curl.exe -X POST http://localhost/api/v1/admin/crash`).
3. **Observe**: The targeted container terminates; the secondary replica absorbs 100% of user traffic with zero dropped requests; the Kubernetes kubelet automatically respawns a healthy pod within seconds.

### Scenario B: Dynamic Horizontal Autoscaling (HPA)
1. In the App Dashboard at `http://localhost/`, click **"Burn CPU (Trigger HPA Autoscaler)"** to generate load.
2. In your terminal, watch the autoscaler: `kubectl get hpa -n dev -w`.
3. **Observe**: As CPU exceeds the 60% threshold, the HPA controller scales the deployment from **2 pods ➔ 3 pods ➔ 5 pods** in real-time.

### Scenario C: GitOps Continuous Delivery
1. Make any configuration change in this repository (e.g., replica count or resource limits) and run `git push`.
2. **Observe**: Argo CD detects the Git commit hash, transitions to `Progressing`, and executes an automated rolling update.

---

## 🚀 Quick Start (Local Setup)

```bash
# 1. Create multi-port cluster
kind create cluster --config kind-config.yaml --name devops-cluster

# 2. Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 3. Install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Deploy Dev & Prod via GitOps
kubectl apply -f argocd/application-dev.yaml
kubectl apply -f argocd/application-prod.yaml
```

---

## 👤 Author
**PandaVerse09** • DevOps & Platform Engineering Lab
