# Tenant Onboarding Guide

Welcome to the Marina Platform! This guide will help you get started with creating your first EKS Auto Mode cluster.

## Prerequisites

- GitHub account with access to `marina-platform-demo` repository
- AWS account ID for your team
- Email addresses for team members

## Step 1: Request Tenant Folder

Contact the platform team to create your tenant folder:

**Slack**: #platform-engineering
**Email**: platform-team@company.com

Provide:
- Team name (e.g., `team-alpha`)
- AWS account ID

The platform team will:
1. Create your folder structure:
```
tenants/YOUR-TEAM/
├── clusters/
└── applications/
```
2. Commit to Git
3. AppProject `YOUR-TEAM` is automatically created (~30 seconds)
4. You're ready to create clusters!

## Step 2: Clone Repository

```bash
git clone https://github.com/badrish-s/marina-platform-demo.git
cd marina-platform-demo
```

## Step 3: Create Your First Cluster

### 3.1 Copy Template

```bash
cp tenants/_templates/cluster-request.yaml tenants/YOUR-TEAM/clusters/dev.yaml
```

### 3.2 Note on Clusters

In this simplified setup, all applications run in the management cluster with namespace isolation. You don't need to create separate clusters.

```yaml
apiVersion: platform.marina.com/v1
kind: ClusterRequest
metadata:
  name: YOUR-TEAM-dev
  labels:
    tenant: YOUR-TEAM
    environment: dev
    managed-by: marina-platform
spec:
  kubernetesVersion: "1.31"
  region: us-east-1
  
  vpc:
    create: true
  
  computeConfig:
    autoMode: true  # EKS Auto Mode with default node pools
  
  tags:
    cost-center: "YOUR-CC"
    project: "YOUR-PROJECT"
    owner: "YOUR-TEAM"
    environment: "dev"
```

**That's it!** EKS Auto Mode handles all compute automatically with default node pools (general-purpose, system).

### 3.3 Commit and Push

```bash
git add tenants/YOUR-TEAM/clusters/dev.yaml
git commit -m "Request dev cluster for YOUR-TEAM"
git push origin main
```

## Step 4: Monitor Cluster Creation

### 4.1 Check ArgoCD

```bash
# Login to ArgoCD
argocd login <argocd-url> --username admin

# Watch application
argocd app get YOUR-TEAM-dev --watch
```

### 4.2 Check Status

```bash
# Check KRO instance
kubectl get clusterrequests -A

# Check EKS cluster creation
kubectl get clusters.eks.services.k8s.aws -A

# Check ArgoCD registration
kubectl get secret YOUR-TEAM-dev -n argocd
```

### 4.3 Timeline

| Time | Event |
|------|-------|
| T+0 | Git push |
| T+3min | ArgoCD detects change |
| T+5min | IAM roles created |
| T+10min | VPC created |
| T+25min | EKS cluster creating |
| T+35min | Cluster ACTIVE |

**Total: ~35 minutes** (No addons to install!)

## Step 5: Access Your Cluster

### 5.1 Update kubeconfig

```bash
aws eks update-kubeconfig --name YOUR-TEAM-dev --region us-east-1
```

### 5.2 Verify Access

```bash
# Check nodes (managed by EKS Auto Mode)
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Cluster is ready for your workloads!
```

## Step 6: Deploy Your Application

### 6.1 Copy Application Template

```bash
cp tenants/_templates/app-request.yaml tenants/YOUR-TEAM/applications/dev/my-app.yaml
```

### 6.2 Edit Application

```yaml
apiVersion: v1alpha1
kind: WebAppWithS3
metadata:
  name: web-app
spec:
  tenantName: YOUR-TEAM      # e.g., team-alpha
  bucketSuffix: "YYYYMMDD"   # e.g., 20241028 or your AWS account ID
  clusterName: CLUSTER-NAME  # e.g., marina-integ-test-eks-cluster
  replicas: 2
    limits:
      cpu: "500m"
      memory: "512Mi"
```

**What you get:**
- Kubernetes Deployment with nginx
- S3 bucket for static content
- IAM role with Pod Identity
- Application Load Balancer
- Public URL to access your app

### 6.3 Deploy

```bash
git add tenants/YOUR-TEAM/applications/web-app.yaml
git commit -m "Deploy web-app"
git push
```

**Timeline:** ~5-10 minutes (ALB creation takes time)

For more details, see [deploying-applications.md](deploying-applications.md)

## Step 7: Request Additional Clusters

Need staging or prod? Just create another file:

```bash
cp tenants/YOUR-TEAM/clusters/dev.yaml tenants/YOUR-TEAM/clusters/staging.yaml
# Edit staging.yaml
git add tenants/YOUR-TEAM/clusters/staging.yaml
git commit -m "Request staging cluster"
git push
```

## Common Tasks

### Scaling

**No action needed!** EKS Auto Mode automatically scales compute based on your workload demands. Just deploy your applications and the cluster handles the rest.

### Delete Application

Delete the application file:

```bash
git rm tenants/YOUR-TEAM/clusters/dev.yaml
git commit -m "Delete dev cluster"
git push
```

ArgoCD will automatically clean up all resources.

## Troubleshooting

### Cluster Creation Stuck

```bash
# Check KRO logs
kubectl logs -n kro-system deployment/kro -f

# Check ACK controller logs
kubectl logs -n ack-system deployment/ack-eks-controller -f

# Check for errors
kubectl describe clusterrequest YOUR-TEAM-dev -n argocd
```

### Can't Access Cluster

```bash
# Verify cluster is ACTIVE
aws eks describe-cluster --name YOUR-TEAM-dev

# Check IAM permissions
aws sts get-caller-identity

# Update kubeconfig again
aws eks update-kubeconfig --name YOUR-TEAM-dev --region us-east-1
```

### Pods Not Scheduling

```bash
# Check node status
kubectl get nodes

# Check pod events
kubectl describe pod <pod-name> -n <namespace>

# EKS Auto Mode will automatically provision nodes as needed
```

## Getting Help

- **Documentation**: [docs/](.)
- **Slack**: #platform-engineering
- **Email**: platform-team@company.com
- **Office Hours**: Tuesdays 2-3pm

## Best Practices

✅ **Use descriptive names** - `team-alpha-dev`, not `cluster1`
✅ **Use GitOps** - Deploy apps via ArgoCD
✅ **Tag resources** - For cost tracking and organization
✅ **Start small** - Begin with dev, then staging, then prod
✅ **Monitor costs** - Check AWS Cost Explorer regularly
✅ **Trust Auto Mode** - Let EKS handle compute scaling automatically
✅ **Use existing VPCs** - When possible, to reduce networking costs

## Next Steps

1. Deploy your first application
2. Set up CI/CD pipeline
3. Configure monitoring and alerting
4. Request staging cluster
5. Plan production deployment

Welcome to the platform! 🚀
