# Quick Reference - Marina Platform

## Create a Cluster (3 Steps)

### 1. Copy Template
```bash
cp tenants/_templates/cluster-request.yaml tenants/YOUR-TEAM/clusters/dev.yaml
```

### 2. Edit File
```yaml
apiVersion: platform.marina.com/v1
kind: ClusterRequest
metadata:
  name: YOUR-TEAM-dev
  labels:
    tenant: YOUR-TEAM
    environment: dev
spec:
  kubernetesVersion: "1.31"
  region: us-east-1
  vpc:
    create: true
  computeConfig:
    autoMode: true
  tags:
    owner: "YOUR-TEAM"
```

### 3. Commit and Push
```bash
git add tenants/YOUR-TEAM/clusters/dev.yaml
git commit -m "Request dev cluster"
git push
```

**Done!** Cluster ready in ~35 minutes.

---

## Access Your Cluster

```bash
# Update kubeconfig
aws eks update-kubeconfig --name YOUR-TEAM-dev --region us-east-1

# Verify
kubectl get nodes
kubectl get pods -A
```

---

## Use Existing VPC

```yaml
spec:
  vpc:
    create: false
    vpcId: vpc-xxxxx
    publicSubnet1Id: subnet-xxxxx
    publicSubnet2Id: subnet-xxxxx
    privateSubnet1Id: subnet-xxxxx
    privateSubnet2Id: subnet-xxxxx
```

---

## Create in Different AWS Account

```yaml
spec:
  accountId: "123456789012"  # Target account
  region: us-west-2
```

---

## Monitor Cluster Creation

```bash
# Watch ArgoCD application
argocd app get YOUR-TEAM-dev --watch

# Check KRO instance
kubectl get eksclusterwithvpcs -A

# Check EKS cluster
kubectl get clusters.eks.services.k8s.aws -A

# Check AWS
aws eks describe-cluster --name YOUR-TEAM-dev
```

---

## Delete Cluster

```bash
git rm tenants/YOUR-TEAM/clusters/dev.yaml
git commit -m "Delete dev cluster"
git push
```

ArgoCD automatically cleans up all resources.

---

## Deploy Application

```bash
# Connect to cluster
aws eks update-kubeconfig --name YOUR-TEAM-dev

# Deploy
kubectl apply -f your-app.yaml

# Or use ArgoCD
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR-ORG/YOUR-APP
    path: k8s/
    targetRevision: main
  destination:
    name: YOUR-TEAM-dev
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

---

## Troubleshooting

### Cluster Creation Stuck

```bash
# Check KRO logs
kubectl logs -n kro-system deployment/kro -f

# Check ACK logs
kubectl logs -n ack-system deployment/ack-eks-controller -f

# Check resource status
kubectl describe eksclusterwithvpc YOUR-TEAM-dev -n argocd
```

### Can't Access Cluster

```bash
# Verify cluster exists
aws eks describe-cluster --name YOUR-TEAM-dev

# Check IAM permissions
aws sts get-caller-identity

# Update kubeconfig
aws eks update-kubeconfig --name YOUR-TEAM-dev --region us-east-1 --profile YOUR-PROFILE
```

### Pods Not Scheduling

**Don't worry!** EKS Auto Mode automatically provisions nodes when pods are pending. Wait 2-3 minutes.

```bash
# Check pod status
kubectl get pods -A

# Check events
kubectl get events -A --sort-by='.lastTimestamp'

# Nodes will appear automatically
kubectl get nodes -w
```

---

## Supported Kubernetes Versions

- 1.30
- 1.31
- 1.32 (latest)

---

## Supported Regions

All AWS regions that support EKS Auto Mode:
- us-east-1
- us-east-2
- us-west-1
- us-west-2
- eu-west-1
- eu-central-1
- ap-southeast-1
- ap-northeast-1
- And more...

---

## Cost Estimate

**Per Cluster:**
- EKS Control Plane: ~$73/month
- NAT Gateway: ~$32/month (if VPC created)
- Compute: Variable (Auto Mode pricing)
  - Only pay for nodes that are running
  - Automatic optimization

**Example Dev Cluster:**
- Small workload: ~$150/month
- Medium workload: ~$300/month
- Large workload: ~$500+/month

---

## Default Node Pools

EKS Auto Mode provides two default node pools:

**general-purpose:**
- For application workloads
- Multiple instance types
- Automatic scaling

**system:**
- For cluster components
- Optimized for system pods
- Separate from apps

**You don't configure these** - AWS manages them automatically!

---

## Tags

Always include these tags:
```yaml
tags:
  owner: "team-name"
  environment: "dev|staging|prod"
  cost-center: "CC-12345"
  project: "project-name"
```

Used for:
- Cost allocation
- Resource organization
- Compliance tracking

---

## Best Practices

✅ Use descriptive cluster names: `team-alpha-dev`
✅ Tag all resources properly
✅ Use existing VPCs when possible
✅ Start with dev, then staging, then prod
✅ Deploy via GitOps (ArgoCD)
✅ Monitor costs regularly
✅ Delete unused clusters

❌ Don't create clusters manually
❌ Don't modify clusters outside Git
❌ Don't share clusters between teams
❌ Don't skip tagging

---

## Getting Help

- **Docs:** [README.md](README.md), [ARCHITECTURE.md](ARCHITECTURE.md)
- **Onboarding:** [docs/tenant-onboarding.md](docs/tenant-onboarding.md)
- **Slack:** #platform-engineering
- **Email:** platform-team@company.com

---

## Cheat Sheet

| Task | Command |
|------|---------|
| Create cluster | `git add/commit/push cluster YAML` |
| Access cluster | `aws eks update-kubeconfig --name NAME` |
| Check nodes | `kubectl get nodes` |
| Deploy app | `kubectl apply -f app.yaml` |
| Delete cluster | `git rm cluster YAML; git push` |
| Watch creation | `argocd app get NAME --watch` |
| Check logs | `kubectl logs -n kro-system deployment/kro` |

---

**Remember:** It's just Git commits. That's the whole platform! 🚀
