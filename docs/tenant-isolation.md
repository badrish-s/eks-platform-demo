# Tenant Isolation

## Overview

Tenant isolation is enforced using ArgoCD AppProjects. Each tenant automatically gets their own project with restricted permissions when their folder is created in Git.

## How It Works

### Automatic AppProject Generation
1. Platform team creates tenant folder: `tenants/team-gamma/`
2. Commits to Git
3. ApplicationSet `tenant-projects-generator` detects the folder
4. Automatically creates AppProject `team-gamma`
5. Tenant can now deploy clusters and apps

### AppProject Restrictions
Each tenant's AppProject restricts:
- **Source paths**: Only `tenants/TEAM-NAME/` folder
- **Destination clusters**: Only `TEAM-NAME-*` clusters
- **Destination namespaces**: Only `TEAM-NAME-*` namespaces

### Enforcement
- Team Alpha can only deploy to `team-alpha-dev`, `team-alpha-prod`, etc.
- Team Beta can only deploy to `team-beta-dev`, `team-beta-prod`, etc.
- Cross-tenant deployments are blocked by ArgoCD

## Setup

### Deploy AppProject Generator
```bash
kubectl apply -f platform/argocd-config/tenant-projects-generator.yaml
```

This watches Git and auto-creates AppProjects for all tenant folders.

### Add New Tenant (Fully Automated)
```bash
# 1. Create tenant folder structure
mkdir -p tenants/team-gamma/{clusters,applications}

# 2. Commit to Git
git add tenants/team-gamma
git commit -m "Add team-gamma tenant"
git push

# 3. Wait ~30 seconds - AppProject is auto-created!
kubectl get appproject team-gamma -n argocd
```

No manual scripts needed!

## What's Protected

✅ Team Alpha cannot deploy to Team Beta clusters
✅ Team Alpha cannot read Team Beta resources
✅ Teams cannot modify platform configuration
✅ Teams cannot create dangerous resources (ResourceQuota, LimitRange)
✅ AppProjects are automatically created and managed via GitOps

## Verification

```bash
# List all projects (should see one per tenant)
kubectl get appprojects -n argocd

# Check team-alpha project
kubectl get appproject team-alpha -n argocd -o yaml

# Check ApplicationSet that generates projects
kubectl get applicationset tenant-projects-generator -n argocd
```

## How to Remove a Tenant

```bash
# Delete tenant folder from Git
git rm -r tenants/team-gamma
git commit -m "Remove team-gamma tenant"
git push

# AppProject will be automatically deleted (prune: true)
```
