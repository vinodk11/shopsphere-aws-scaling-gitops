# ShopSphere GitOps Repository (`shopsphere-gitops`)

This repository is the single source of truth for the desired state of ShopSphere Kubernetes workloads running on Amazon EKS, continuously reconciled by **Argo CD** in **Stage 10**.

---

## 🏛️ Architecture Overview

```
Developer Push
     ↓
Application Repo (shopsphere-aws-scaling)
     ↓
Jenkins CI Pipeline (Build, Test, SAST, SCA, Trivy Container Scan, Push to ECR)
     ↓
Jenkins GitOps Bot commits new image tag to this repository
     ↓
shopsphere-gitops (environments/production/kustomization.yaml)
     ↓
Argo CD detects OutOfSync
     ↓
Argo CD automated reconciliation (RollingUpdate to EKS)
     ↓
Pod readiness checks & AWS ALB Target Health
```

---

## 📁 Repository Structure

```
shopsphere-gitops/
├── apps/                                   # Base Kubernetes manifests (environment-agnostic)
│   ├── frontend-service/                   # Frontend Web Gateway & Auth UI
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── kustomization.yaml
│   ├── product-service/                    # 16-Product Catalog Service
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   └── kustomization.yaml
│   ├── order-service/                      # SQS Asynchronous Order Processing Service
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── serviceaccount.yaml             # IRSA IAM binding
│   │   └── kustomization.yaml
│   └── user-service/                       # User Auth & Session Service
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── hpa.yaml
│       └── kustomization.yaml
│
├── environments/
│   └── production/                         # Production Kustomize overlay
│       ├── kustomization.yaml              # Production overlay with image tags & patches
│       ├── app-config.yaml                 # Non-sensitive ConfigMaps
│       └── target-group-binding.yaml       # AWS Load Balancer Controller TargetGroupBindings
│
└── argocd/                                 # Argo CD Declarative Configurations
    ├── project.yaml                        # Argo CD AppProject with RBAC & destination constraints
    └── applications/
        └── shopsphere-production.yaml      # Argo CD Application pointing to environments/production
```

---

## 🚀 How Desired State is Updated

### 1. Automated CI/CD (Jenkins GitOps Update)
When code is committed to the application repository, the Jenkins pipeline:
1. Builds, scans, and pushes immutable Docker images to ECR:
   `165772574557.dkr.ecr.us-east-1.amazonaws.com/shopsphere-product:v10.1-a1b2c3d`
2. Updates `environments/production/kustomization.yaml` in this GitOps repository:
   ```yaml
   images:
     - name: shopsphere-product
       newName: 165772574557.dkr.ecr.us-east-1.amazonaws.com/shopsphere-stage9-product
       newTag: v10.1-a1b2c3d
   ```
3. Commits and pushes the update to `main`.
4. Argo CD detects the change and automatically reconciles the EKS cluster.

### 2. Manual Promotion / Verification
To test Kustomize rendering locally:
```bash
kubectl kustomize environments/production
```

---

## 🔄 Rollback Procedure

To roll back a faulty deployment, revert the git commit in this repository:
```bash
git log -n 5 --oneline
git revert HEAD --no-edit
git push origin main
```
Argo CD immediately detects the reversion and redeploys the previous known-healthy container image.

---

## 🛡️ Self-Healing & Drift Correction

Argo CD is configured with `selfHeal: true`:
```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: true
```
If an operator accidentally modifies a runtime resource in EKS (e.g., `kubectl scale deployment/frontend-service --replicas=1`), Argo CD detects drift within seconds and automatically restores the desired state (`replicas: 2`).
