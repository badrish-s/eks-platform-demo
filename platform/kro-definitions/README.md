# KRO Resource Definitions

This directory contains KRO (Kubernetes Resource Orchestrator) ResourceGraphDefinitions for:
- **Applications:** Web apps with S3, IAM, and ALB

## What is KRO?

KRO is a Kubernetes controller that orchestrates the creation of complex resources with dependencies. It allows you to define a "graph" of resources that need to be created in a specific order.

## Files

### `eks-auto-mode-cluster.yaml`

This is the main ResourceGraphDefinition that creates a complete EKS Auto Mode cluster.

**What it creates:**

1. **VPC Resources** (if `vpc.create: true`):
   - VPC with DNS support
   - Internet Gateway
   - 2 Public Subnets (across 2 AZs)
   - 2 Private Subnets (across 2 AZs)
   - NAT Gateway with Elastic IP
   - Route Tables (public and private)

2. **IAM Roles**:
   - EKS Cluster Role (with required policies)
   - EKS Node Role (for Auto Mode nodes)

3. **EKS Cluster**:
   - EKS cluster with Auto Mode enabled
   - Default node pools: `general-purpose`, `system`
   - Kubernetes version as specified
   - Public and private API endpoints

4. **ArgoCD Integration**:
   - Secret in ArgoCD namespace
   - Registers cluster with ArgoCD
   - Enables GitOps deployments

## How It Works

### Input (ClusterRequest)

Teams create a simple YAML file:

```yaml
apiVersion: platform.marina.com/v1
kind: ClusterRequest
metadata:
  name: team-alpha-dev
  labels:
    tenant: team-alpha
    environment: dev
spec:
  kubernetesVersion: "1.31"
  region: us-east-1
  vpc:
    create: true
  computeConfig:
    autoMode: true
  tags:
    owner: "team-alpha"
```

### Processing

1. ArgoCD detects the file in Git
2. Creates a KRO instance from this ResourceGraphDefinition
3. KRO creates resources in dependency order:
   - VPC → Subnets → NAT Gateway → Route Tables
   - IAM Roles
   - EKS Cluster (waits for VPC and IAM)
   - ArgoCD Secret (waits for cluster to be ACTIVE)

### Output

- Fully functional EKS Auto Mode cluster
- Registered with ArgoCD
- Ready for workload deployment

## Resource Dependencies

```
VPC
 ├─→ Internet Gateway
 ├─→ Public Subnets
 ├─→ Private Subnets
 ├─→ Elastic IP
 │    └─→ NAT Gateway
 │         └─→ Private Route Table
 └─→ Public Route Table

IAM Roles
 ├─→ Cluster Role
 └─→ Node Role

EKS Cluster (depends on VPC + IAM)
 └─→ ArgoCD Secret (depends on cluster ACTIVE)
```

## Conditional Resources

The ResourceGraphDefinition uses `includeWhen` to conditionally create resources:

### New VPC
If `vpc.create: true`:
- Creates all VPC resources
- Uses VPC references in EKS cluster

### Existing VPC
If `vpc.create: false`:
- Skips VPC creation
- Uses provided VPC IDs in EKS cluster

## EKS Auto Mode Configuration

The cluster is created with Auto Mode enabled:

```yaml
computeConfig:
  enabled: true
  nodePools:
    - general-purpose
    - system
```

**What this means:**
- AWS automatically provisions nodes based on pod requirements
- No manual node group configuration needed
- Automatic scaling and lifecycle management
- Two default node pools for different workload types

## Status Fields

The ResourceGraphDefinition exposes status fields that can be used by other resources:

```yaml
status:
  clusterARN: ${ekscluster.status.ackResourceMetadata.arn}
  certificateAuthority: ${ekscluster.status.certificateAuthority.data}
  endpoint: ${ekscluster.status.endpoint}
  clusterStatus: ${ekscluster.status.status}
  vpcId: ${vpc.status.vpcID}
```

These are used by:
- ArgoCD secret (needs endpoint and CA data)
- Monitoring systems
- Other dependent resources

## Customization

### Using Existing VPC

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

### Custom VPC CIDR

```yaml
spec:
  vpc:
    create: true
    vpcCidr: "10.1.0.0/16"
    publicSubnet1Cidr: "10.1.1.0/24"
    publicSubnet2Cidr: "10.1.2.0/24"
    privateSubnet1Cidr: "10.1.11.0/24"
    privateSubnet2Cidr: "10.1.12.0/24"
```

### Different Region

```yaml
spec:
  region: us-west-2
```

### Different Kubernetes Version

```yaml
spec:
  kubernetesVersion: "1.32"
```

## Deployment

This ResourceGraphDefinition is deployed to the management cluster:

```bash
kubectl apply -f eks-auto-mode-cluster.yaml
```

Once deployed, KRO watches for ClusterRequest instances and creates the resources defined in this graph.

## Monitoring

### Check ResourceGraphDefinition Status

```bash
kubectl get resourcegraphdefinitions.kro.run
```

Expected output:
```
NAME                        APIVERSION                    KIND             STATE
eksautocluster.kro.run      platform.marina.com/v1        ClusterRequest   Active
```

### Check Cluster Creation

```bash
# List all web app instances
kubectl get webappwiths3 -A

# Check specific cluster
kubectl describe clusterrequest team-alpha-dev -n argocd

# Check underlying resources
kubectl get clusters.eks.services.k8s.aws -A
kubectl get vpcs.ec2.services.k8s.aws -A
kubectl get roles.iam.services.k8s.aws -A
```

## Troubleshooting

### ResourceGraphDefinition Not Active

```bash
kubectl describe resourcegraphdefinition eksautocluster.kro.run
```

Check for:
- Syntax errors in the YAML
- Missing CRDs (ACK controllers not installed)
- KRO controller not running

### Cluster Creation Stuck

```bash
# Check KRO logs
kubectl logs -n kro-system deployment/kro -f

# Check ACK controller logs
kubectl logs -n ack-system deployment/ack-eks-controller -f
kubectl logs -n ack-system deployment/ack-ec2-controller -f
kubectl logs -n ack-system deployment/ack-iam-controller -f
```

Common issues:
- ACK controllers not installed
- IAM permissions missing
- S3 bucket name conflicts

### Cluster Created But Not in ArgoCD

```bash
# Check if secret was created
kubectl get secret team-alpha-dev -n argocd

# Check secret contents
kubectl get secret team-alpha-dev -n argocd -o yaml
```

## Best Practices

1. **Test with one cluster first** - Validate the ResourceGraphDefinition works before scaling
2. **Monitor resource creation** - Watch KRO and ACK logs during first deployments
3. **Use existing VPCs when possible** - Reduces costs and creation time
4. **Tag resources properly** - Makes cost tracking and organization easier
5. **Version control** - Keep this file in Git with your platform code

## References

- [KRO Documentation](https://github.com/kubernetes-sigs/kro)
- [ACK EKS Controller](https://github.com/aws-controllers-k8s/eks-controller)
- [ACK EC2 Controller](https://github.com/aws-controllers-k8s/ec2-controller)
- [ACK IAM Controller](https://github.com/aws-controllers-k8s/iam-controller)
- [EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
