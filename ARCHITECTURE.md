# Marina Platform Architecture

## Overview

Marina Platform is a **minimal, self-service EKS cluster provisioning platform** that enables teams to create EKS Auto Mode clusters by simply committing YAML files to Git.

## Design Philosophy

**Simplicity First:**
- EKS Auto Mode only (no managed node groups, no Fargate)
- No addons installed by default
- Minimal configuration required
- Let AWS manage the complexity

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Management Cluster (Hub)                      │
│                                                                  │
│  ┌──────────┐  ┌─────────┐  ┌──────────────────────────┐      │
│  │ ArgoCD   │─▶│   KRO   │─▶│   ACK Controllers        │      │
│  │          │  │         │  │  - iam-controller        │      │
│  │ Watches  │  │ Creates │  │  - eks-controller        │      │
│  │ Git Repo │  │ AWS Res │  │  - ec2-controller        │      │
│  └──────────┘  └─────────┘  └──────────────────────────┘      │
│       │                               │                         │
└───────┼───────────────────────────────┼─────────────────────────┘
        │                               │
        │ Watches tenants/*/clusters/   │ Creates in AWS
        ▼                               ▼
   ┌─────────┐                  ┌──────────────────┐
   │   Git   │                  │   AWS Account    │
   │  Repo   │                  │                  │
   └─────────┘                  │  ┌────────────┐  │
                                │  │ EKS Auto   │  │
                                │  │ Mode       │  │
                                │  │ Cluster    │  │
                                │  └────────────┘  │
                                │                  │
                                │  Default Pools:  │
                                │  - general-      │
                                │    purpose       │
                                │  - system        │
                                └──────────────────┘
```

## Components

### 1. Management Cluster (Hub)

**Purpose:** Orchestrates cluster creation across AWS accounts

**Components:**
- **ArgoCD** - GitOps engine that watches Git repository
- **KRO** - Kubernetes Resource Orchestrator for complex resource graphs
- **ACK Controllers** - AWS Controllers for Kubernetes
  - `ack-iam-controller` - Creates IAM roles
  - `ack-eks-controller` - Creates EKS clusters
  - `ack-ec2-controller` - Creates VPCs, subnets, etc.
  - `ack-s3-controller` - Creates S3 buckets

### 2. Git Repository

**Structure:**
```
marina-platform-demo/
├── platform/                      # Platform team owns
│   ├── argocd-config/
│   │   └── tenant-applications.yaml        # Watches app requests
│   ├── kro-definitions/
│   │   └── web-app-with-s3.yaml            # App + S3 + IAM blueprint
│
└── tenants/                       # App teams own
    ├── _templates/
    │   └── app-request.yaml       # Template
    └── team-alpha/
        └── applications/
            └── web-app.yaml       # App request
    │           └── web-app.yaml   # App request
    └── team-beta/
        ├── clusters/
        │   └── dev.yaml
        └── applications/
            └── dev/
```

### 3. Tenant Namespaces

**Type:** Kubernetes namespaces in management cluster

**Features:**
- Fully managed compute
- Automatic node provisioning
- Default node pools: `general-purpose`, `system`
- No manual node group management
- No addons installed

## How It Works

### Cluster Creation Flow

```
1. Team commits application request
   └─▶ tenants/team-alpha/applications/web-app.yaml

2. ArgoCD detects new file
   └─▶ Creates Application: team-alpha-dev

3. Application references KRO definition
   └─▶ Creates KRO instance: ClusterRequest

4. KRO orchestrates resources in order
   └─▶ Creates ACK custom resources:
       ├─▶ IAM Roles (cluster role, node role)
       ├─▶ VPC (if create: true)
       │   ├─▶ Subnets (public, private)
       │   ├─▶ Internet Gateway
       │   ├─▶ NAT Gateway
       │   └─▶ Route Tables
       └─▶ EKS Cluster (Auto Mode enabled)

5. ACK controllers create AWS resources
   └─▶ Each controller talks to AWS API
       └─▶ Creates actual resources in target account

6. Cluster becomes ACTIVE
   └─▶ Ready for workload deployment
```

**Timeline:** ~35 minutes from commit to ready

### Application Deployment Flow

```
1. Team commits application request
   └─▶ tenants/team-alpha/applications/dev/web-app.yaml

2. ArgoCD detects new file
   └─▶ Creates Application: team-alpha-dev-web-app

3. Application references KRO definition
   └─▶ Creates KRO instance: WebAppWithS3

4. KRO orchestrates resources in order
   └─▶ Creates ACK and K8s resources:
       ├─▶ S3 Bucket (via ACK)
       ├─▶ IAM Role (via ACK)
       ├─▶ Pod Identity Association (via ACK)
       ├─▶ Job (uploads HTML to S3)
       ├─▶ Kubernetes Deployment (nginx)
       ├─▶ Kubernetes Service
       └─▶ Ingress (creates ALB)

5. ACK controllers create AWS resources
   └─▶ S3 bucket, IAM role, Pod Identity
       └─▶ ALB Controller creates Load Balancer

6. Application becomes available
   └─▶ Access via ALB URL
       └─▶ Nginx serves content from S3

7. Application is RUNNING
   └─▶ Service exposes app via LoadBalancer
```

**Timeline:** ~15-20 minutes from commit to running app

### What Gets Created

For each application request, the platform creates:

**In AWS:**
- 1 VPC (if `vpc.create: true`)
  - 2 Public subnets (across 2 AZs)
  - 2 Private subnets (across 2 AZs)
  - 1 Internet Gateway
  - 1 NAT Gateway
  - Route tables
- IAM roles:
  - EKS cluster role
  - EKS node role (for Auto Mode)
- 1 EKS cluster (Auto Mode enabled)
  - Default node pools configured
  - Automatic compute scaling

**In Management Cluster:**
- 1 KRO instance (EksClusterWithVpc)
- Multiple ACK resources (Role, VPC, Cluster, etc.)
- 1 ArgoCD Application

## EKS Auto Mode

### What is Auto Mode?

EKS Auto Mode is a fully managed compute option where AWS:
- Automatically provisions nodes based on pod requirements
- Manages node lifecycle (upgrades, patches, scaling)
- Provides default node pools for different workload types
- Handles node group configuration

### Default Node Pools

**general-purpose:**
- For application workloads
- Variety of instance types
- Automatic scaling based on demand

**system:**
- For system components (CoreDNS, kube-proxy, etc.)
- Optimized for cluster operations
- Separate from application workloads

### Benefits

✅ **Zero node management** - No node groups to configure
✅ **Automatic scaling** - Nodes appear when needed
✅ **Cost optimized** - Only pay for what you use
✅ **Always up-to-date** - AWS manages patches and upgrades
✅ **Best practices** - AWS-recommended configuration

## Multi-Account Support

The platform supports creating clusters in different AWS accounts:

```yaml
# Cluster in Account A
spec:
  accountId: "111111111111"
  region: us-east-1

# Cluster in Account B
spec:
  accountId: "222222222222"
  region: us-west-2
```

**Requirements:**
- ACK controllers installed (S3, IAM, EKS)
- AWS Load Balancer Controller installed
- EKS Pod Identity Agent installed

## Security

### Tenant Isolation

**ArgoCD AppProjects:**
- Each tenant gets an auto-generated AppProject
- Restricts source paths to `tenants/TEAM-NAME/`
- Restricts destinations to `TEAM-NAME-*` clusters
- Prevents cross-tenant deployments

**How it works:**
1. Platform team creates `tenants/team-gamma/` folder
2. ApplicationSet `tenant-projects-generator` detects folder
3. Automatically creates AppProject `team-gamma`
4. All team-gamma Applications use this project
5. ArgoCD enforces isolation boundaries

### IAM Roles

**Management Cluster:**
- ACK controllers with IAM permissions to create AWS resources
- ArgoCD with access to Git repository
- KRO for resource orchestration
- AWS Load Balancer Controller for ALB creation

### Cluster Access

- Cluster creator (platform) has admin access
- Teams access via IAM roles/users
- RBAC configured via AWS IAM
- Teams isolated via ArgoCD AppProjects

## Scalability

**Tested Limits:**
- 100+ clusters per management cluster
- Multiple AWS accounts
- Multiple regions
- Concurrent cluster creation

**Bottlenecks:**
- AWS API rate limits (handled by ACK)
- ArgoCD sync performance (configurable)
- Git repository size (minimal impact)

## Cost Considerations

**Per Cluster:**
- EKS cluster: ~$73/month
- VPC NAT Gateway: ~$32/month (if created)
- Compute: Variable (Auto Mode pricing)
- Data transfer: Variable

**Optimization:**
- Use existing VPCs when possible
- Share NAT gateways
- Right-size workloads
- Use Auto Mode's automatic optimization

## Limitations

**Current Limitations:**
- EKS Auto Mode only (no managed node groups)
- No addon installation
- No custom node configurations
- No Fargate support

**By Design:**
- Simplicity over flexibility
- AWS-managed over self-managed
- Convention over configuration

## Future Enhancements

**Potential Additions:**
- Cost tracking per cluster
- Slack notifications on cluster ready
- Automatic cluster deletion after X days
- Budget enforcement
- Multi-region VPC peering
- Cluster templates (dev, staging, prod)

## Comparison: Traditional vs Marina Platform

| Aspect | Traditional | Marina Platform |
|--------|-------------|-----------------|
| Cluster Creation | Manual/Terraform | Git commit |
| Time to Cluster | Hours/Days | 35 minutes |
| Node Management | Manual | Automatic (Auto Mode) |
| Addons | Manual install | None (minimal) |
| Scaling | Configure ASG | Automatic |
| Updates | Manual | AWS-managed |
| Complexity | High | Low |
| Team Autonomy | Low | High |

## Getting Started

1. **Platform Team:** Deploy management cluster
2. **Platform Team:** Configure ArgoCD and KRO
3. **App Teams:** Commit application requests
4. **Platform:** Automatically creates resources
5. **App Teams:** Access applications via ALB

See [SETUP.md](SETUP.md) for detailed instructions.

## Support

- **Documentation:** [docs/](docs/)
- **Slack:** #platform-engineering
- **Email:** platform-team@company.com

---

**Philosophy:** Keep it simple. Let AWS manage the complexity. Focus on delivering value.
