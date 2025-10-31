# Marina Platform Demo

A self-service platform for creating EKS Auto Mode clusters using GitOps principles.

## Overview

This platform enables application teams to:
1. **Create EKS Auto Mode clusters** by committing a YAML file
2. **Deploy applications** (simple K8s or with AWS resources like S3) using KRO ResourceGraphs

Everything is GitOps-based - commit YAML files, and the platform handles provisioning automatically.

**Key Features:**
- ✅ **Self-service app deployments** with KRO ResourceGraphs
- ✅ **Single management cluster** with namespace isolation
- ✅ **Complex multi-resource apps** (app + S3 + IAM + ALB)
- ✅ **GitOps-based** - all changes via Git commits
- ✅ **~35 minutes** from commit to ready cluster
- ✅ **GitOps-based** (all changes via Git)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Management Cluster (Hub)                      │
│                                                                  │
│  ┌──────────┐  ┌─────────┐  ┌──────────────────────────┐      │
│  │ ArgoCD   │─▶│   KRO   │─▶│   ACK Controllers        │      │
│  │          │  │         │  │  - iam-controller        │      │
│  │ GitOps   │  │ Resource│  │  - eks-controller        │      │
│  │ Engine   │  │ Graphs  │  │  - ec2-controller        │      │
│  └──────────┘  └─────────┘  └──────────────────────────┘      │
│                                      │                           │
└──────────────────────────────────────┼───────────────────────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │   AWS Account   │
                              │                 │
                              │  Tenant Clusters│
                              └─────────────────┘
```

## Quick Start

### For Platform Team

1. **Deploy KRO ResourceGraphDefinitions**
   ```bash
   # Cluster provisioning
   kubectl apply -f platform/kro-definitions/eks-auto-mode-cluster.yaml
   
   # Application types
   kubectl apply -f platform/kro-definitions/web-app-with-s3.yaml
   ```

2. **Deploy ArgoCD ApplicationSets**
   ```bash
   # Auto-generate tenant AppProjects
   # Watch for application requests
   kubectl apply -f platform/argocd-config/tenant-applications.yaml
   ```

3. **Deploy KRO Definitions**
   ```bash
   kubectl apply -f platform/kro-definitions/web-app-with-s3.yaml
   ```

4. **Onboard New Tenant (Fully Automated)**
   ```bash
   # Just create folder and commit - AppProject auto-created!
   mkdir -p tenants/team-gamma/{clusters,applications}
   git add tenants/team-gamma
   git commit -m "Add team-gamma"
   git push
   ```

### For App Teams

1. **Request a Cluster**
   ```bash
   cp tenants/_templates/cluster-request.yaml tenants/YOUR-TEAM/clusters/dev.yaml
   # Edit the file with your requirements
   git add tenants/YOUR-TEAM/clusters/dev.yaml
   git commit -m "Request dev cluster"
   git push
   ```

2. **Wait ~35 minutes** - Your cluster will be automatically created!

3. **Deploy an Application**
   ```bash
   # Copy the template
   cp tenants/_templates/app-request.yaml tenants/YOUR-TEAM/applications/dev/my-app.yaml
   
   # Edit with your app details
   vim tenants/YOUR-TEAM/applications/dev/my-app.yaml
   
   # Commit and push
   git add tenants/YOUR-TEAM/applications/dev/my-app.yaml
   git commit -m "Deploy my-app to dev"
   git push
   ```

4. **Access Your Application**
   ```bash
   aws eks update-kubeconfig --name YOUR-TEAM-dev --region us-east-1
   kubectl get pods -n YOUR-TEAM-dev
   kubectl get svc -n YOUR-TEAM-dev
   ```

## Repository Structure

```
marina-platform-demo/
├── platform/                           # Platform team manages
│   ├── argocd-config/
│   │   └── tenant-applications.yaml   # ✅ Watches app requests
│   ├── kro-definitions/
│   │   └── web-app-with-s3.yaml       # ✅ App + S3 + IAM
├── tenants/                            # App teams manage
│   ├── _templates/
│   │   └── app-request.yaml           # ✅ App template
│   └── team-alpha/
│       └── applications/
│           └── web-app.yaml           # ✅ App request
└── docs/                               # Documentation
    └── tenant-onboarding.md
```

**Status:** All essential files are complete and ready to use!

## Key Features

✅ **Self-Service Clusters** - Teams create EKS clusters by committing YAML files
✅ **Self-Service Applications** - Deploy apps using KRO ResourceGraphs
✅ **Complex Multi-Resource Apps** - App + S3 + IAM + ALB in one YAML
✅ **GitOps Native** - All changes tracked in Git
✅ **EKS Auto Mode** - Fully managed compute with automatic scaling
✅ **Automated** - No manual intervention needed
✅ **Scalable** - Supports hundreds of teams and clusters

## Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Technical architecture and how it works
- **[SETUP.md](SETUP.md)** - Platform deployment instructions
- **[QUICK_REFERENCE.md](QUICK_REFERENCE.md)** - Common commands cheat sheet
- **[docs/deploying-applications.md](docs/deploying-applications.md)** - How to deploy apps with KRO
- **[docs/tenant-isolation.md](docs/tenant-isolation.md)** - Tenant isolation and security
- **[docs/tenant-onboarding.md](docs/tenant-onboarding.md)** - Guide for app teams
- **[platform/kro-definitions/README.md](platform/kro-definitions/README.md)** - KRO ResourceGraph documentation

## Components

- **ArgoCD** - GitOps continuous delivery (watches Git for changes)
- **KRO** - Kubernetes Resource Orchestrator (creates complex multi-resource apps)
- **ACK** - AWS Controllers for Kubernetes (IAM, EKS, EC2, S3)
- **EKS Auto Mode** - Fully managed Kubernetes with automatic compute

## How It Works

### Cluster Provisioning
1. Team commits `tenants/TEAM/clusters/ENV.yaml`
2. ArgoCD ApplicationSet detects the file
3. Creates an ArgoCD Application
4. KRO ResourceGraph provisions: VPC, IAM roles, EKS cluster
5. ~35 minutes later, cluster is ready

### Application Deployment
1. Team commits `tenants/TEAM/applications/ENV/APP.yaml`
2. ArgoCD ApplicationSet detects the file
3. Creates an ArgoCD Application targeting the team's cluster
4. KRO ResourceGraph provisions: S3, IAM Role, Pod Identity, Deployment, Service, ALB
5. ~5-10 minutes later, app is running and accessible via ALB

## Support

- **Slack**: #platform-engineering
- **Email**: platform-team@company.com
- **Issues**: https://github.com/badrish-s/marina-platform-demo/issues

## License

Internal use only - Company Confidential
