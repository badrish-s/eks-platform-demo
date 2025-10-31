# EKS Platform Demo - Setup Guide

## Quick Setup

### Step 1: Create GitHub Repository

```bash
# Using GitHub CLI
gh repo create eks-platform-demo \
  --private \
  --description "Self-service EKS platform with KRO & ACK" \
  --clone

cd eks-platform-demo
```

### Step 2: Copy Files from This Directory

```bash
# From your eks-cluster-mgmt directory
cp -r eks-platform-demo/* ~/workspace/eks-platform-demo/
cd ~/workspace/eks-platform-demo
```

### Step 3: Verify Repository URLs

All ArgoCD config files are already configured for:
```
https://github.com/badrish-s/eks-platform-demo
```

No changes needed!

### Step 4: Initial Commit

```bash
git add .
git commit -m "Initial platform setup"
git push origin main
```

## What's Included

### Directory Structure

```
eks-platform-demo/
├── README.md                          # Platform overview
├── .gitignore                         # Git ignore rules
├── SETUP.md                           # This file
│
├── platform/                          # Platform team manages
│   ├── argocd-config/                 # ✅ Created
│   │   └── tenant-applications.yaml
│   ├── kro-definitions/               # ✅ Created
│   │   └── web-app-with-s3.yaml
│
├── tenants/                           # ✅ Created
│   ├── _templates/
│   │   ├── cluster-request.yaml
│   │   └── app-request.yaml
│   ├── team-alpha/
│   │   ├── clusters/
│   │   │   └── dev.yaml
│   │   └── applications/
│   │       └── dev/
│   │           └── web-app.yaml
│   └── team-beta/
│       ├── clusters/
│       │   └── dev.yaml
│       └── applications/
│           └── dev/
│
└── docs/                              # ✅ Created
    ├── tenant-onboarding.md
    ├── deploying-applications.md
    └── tenant-isolation.md
```

**Status:** All essential files are complete and ready to use!

## Next Steps

### Deploy Management Cluster

You'll need a management cluster with:
- ArgoCD
- KRO (Kubernetes Resource Orchestrator)
- ACK Controllers (IAM, EKS, EC2, S3)

See platform documentation for management cluster setup.

## Testing the Platform

### 1. Configure ArgoCD

```bash
# Get ArgoCD credentials
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login
argocd login <argocd-url> --username admin

# Add repository
argocd repo add https://github.com/YOUR-USERNAME/eks-platform-demo.git \
  --username YOUR-USERNAME \
  --password YOUR-GITHUB-TOKEN
```

### 3. Deploy Platform Components

```bash
# Deploy KRO definitions
kubectl apply -f platform/kro-definitions/eks-auto-mode-cluster.yaml
kubectl apply -f platform/kro-definitions/web-app-with-s3.yaml

# Deploy ArgoCD ApplicationSet
kubectl apply -f platform/argocd-config/tenant-applications.yaml
```

### 4. Setup IAM Roles

```bash
export MGMT_ACCOUNT_ID=123456789012
# ACK controllers are pre-installed in the management cluster
# No additional IAM setup needed for single-cluster demo
```

### 5. Watch Tenant Resources

```bash
# Check AppProjects (auto-created for each tenant)
kubectl get appprojects -n argocd

# Check Applications
argocd app list

# You should see:
# - team-alpha-project (AppProject generator)
# - team-beta-project (AppProject generator)
# - team-alpha-dev (cluster)
# - team-beta-dev (cluster)
# - team-alpha-dev-web-app (application)
```

### 6. Monitor Creation

```bash
# Watch cluster creation
kubectl get clusterrequests -A -w

# Check EKS clusters
kubectl get clusters.eks.services.k8s.aws -A

# Check ArgoCD secrets
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster
```

## Customization

### Add Your Team

```bash
# Create team folder
mkdir -p tenants/YOUR-TEAM/clusters
mkdir -p tenants/YOUR-TEAM/applications/dev

# Copy template
cp tenants/_templates/cluster-request.yaml \
   tenants/YOUR-TEAM/clusters/dev.yaml

# Edit and customize
code tenants/YOUR-TEAM/clusters/dev.yaml

# Commit
git add tenants/YOUR-TEAM
git commit -m "Add YOUR-TEAM"
git push
```

### Modify Cluster Template

Edit `tenants/_templates/cluster-request.yaml` to change defaults:
- Kubernetes version
- Region
- Default addons
- Budget limits

### Deploy Applications

After cluster is created, deploy applications using KRO:
```bash
# Copy template
cp tenants/_templates/app-request.yaml \
   tenants/YOUR-TEAM/applications/dev/my-app.yaml

# Edit with your app details
vim tenants/YOUR-TEAM/applications/dev/my-app.yaml

# Commit and push
git add tenants/YOUR-TEAM/applications/dev/my-app.yaml
git commit -m "Deploy my-app"
git push

# ArgoCD will automatically deploy the app with S3 and ALB
```

## Key Features

✅ **Automatic Tenant Isolation** - AppProjects auto-created per tenant
✅ **Self-Service Clusters** - Teams commit YAML, get EKS clusters
✅ **Self-Service Applications** - Deploy apps with AWS resources (S3, IAM, etc.)
✅ **GitOps Native** - All changes via Git commits
✅ **Fully Automated** - No manual steps for tenant onboarding

## Support

Questions? Contact:
- **Slack**: #platform-engineering
- **Email**: platform-team@company.com
- **GitHub Issues**: https://github.com/YOUR-USERNAME/eks-platform-demo/issues

## What You Get

✅ **Platform Configuration** - ArgoCD ApplicationSet for automation
✅ **KRO Definitions** - Application blueprints
✅ **Documentation** - Complete guides for platform and tenants
✅ **Examples** - Working example for team-alpha

Everything needed for a production-ready platform!
