# Deploying Applications

This guide shows how to deploy applications to your tenant namespace using KRO ResourceGraphs.

## Prerequisites

- You have Git access to the repository
- Your tenant folder exists (e.g., `tenants/team-alpha/`)

## Application Types

Currently supported:
- **WebAppWithS3** - Web application with S3 bucket, IAM role, and ALB

## Deploy a Web App with S3

### Step 1: Copy the Template

```bash
cp tenants/_templates/app-request.yaml \
   tenants/YOUR-TEAM/applications/APP-NAME.yaml
```

Example:
```bash
cp tenants/_templates/app-request.yaml \
   tenants/team-alpha/applications/web-app.yaml
```

### Step 2: Edit the File

```yaml
apiVersion: v1alpha1
kind: WebAppWithS3
metadata:
  name: web-app
spec:
  tenantName: team-alpha
  bucketSuffix: "20241028"  # Use date, account ID, or any unique identifier
  clusterName: eks-integ-test-eks-cluster  # Your EKS cluster name
  replicas: 2
```

**Important:**
- `tenantName` should match your tenant folder name (e.g., `team-alpha`)
- This will be used for naming all resources

### Step 3: Commit and Push

```bash
git add tenants/YOUR-TEAM/applications/APP-NAME.yaml
git commit -m "Deploy APP-NAME"
git push
```

### Step 4: Monitor Deployment

ArgoCD will automatically detect the file and start deployment.

**Check ArgoCD:**
```bash
# List applications
kubectl get applications -n argocd | grep YOUR-TEAM

# Get detailed status
kubectl describe application YOUR-TEAM-APP-NAME -n argocd
```

**Check your namespace:**
```bash
# Check resources in your tenant namespace
kubectl get all -n YOUR-TEAM

# Check S3 bucket
kubectl get bucket -n YOUR-TEAM

# Check IAM role
kubectl get role.iam -n YOUR-TEAM

# Check Pod Identity Association
kubectl get podidentityassociation -n YOUR-TEAM

# Get ALB URL
kubectl get ingress -n YOUR-TEAM
```

## What Gets Created

When you deploy a WebAppWithS3, KRO creates:

1. **S3 Bucket** (via ACK)
   - Unique bucket name: `{tenantName}-landing-{bucketSuffix}`
   - Example: `team-alpha-landing-20241028`
   - Public read access for static content
   - Stores your landing page HTML

2. **IAM Role** (via ACK)
   - Trust policy for EKS Pod Identity
   - Permissions to read/write S3 bucket
   - Used by the upload job

3. **Pod Identity Association** (via ACK)
   - Links ServiceAccount to IAM Role
   - Enables pods to assume the role

4. **Kubernetes Job**
   - Uploads index.html to S3 bucket
   - Runs once on deployment

5. **Kubernetes Deployment**
   - Nginx pods serving content
   - Proxies requests to S3
   - 2 replicas by default

6. **Kubernetes Service**
   - NodePort service
   - Routes traffic to nginx pods

7. **Ingress with ALB**
   - Application Load Balancer
   - Internet-facing
   - Provides public URL

## Accessing Your Application

After deployment completes (~5-10 minutes), get the ALB URL:

```bash
kubectl get ingress -n YOUR-TEAM
```

Look for the `ADDRESS` column - this is your application URL.

Example output:
```
NAME            CLASS   HOSTS   ADDRESS                                                                  PORTS   AGE
team-alpha-web  alb     *       k8s-teamalph-teamalph-abc123-1234567890.us-west-2.elb.amazonaws.com     80      5m
```

Visit the URL in your browser to see your landing page.

## Deployment Timeline

- **Initial deployment**: ~5-10 minutes
  - S3 bucket creation: ~1 minute
  - IAM role creation: ~1 minute
  - ALB provisioning: ~3-5 minutes
  - Pod startup: ~1 minute

- **Updates**: ~2-3 minutes (just app deployment)

## Updating Your Application

To update the number of replicas:

```bash
# Edit the file
vim tenants/YOUR-TEAM/applications/APP-NAME.yaml

# Change replicas
# replicas: 3

# Commit and push
git add tenants/YOUR-TEAM/applications/APP-NAME.yaml
git commit -m "Scale APP-NAME to 3 replicas"
git push
```

ArgoCD will automatically sync the changes.

## Deleting an Application

To delete an application:

```bash
git rm tenants/YOUR-TEAM/applications/APP-NAME.yaml
git commit -m "Remove APP-NAME"
git push
```

**Warning:** This will delete:
- The Kubernetes resources (Deployment, Service, Ingress)
- The S3 bucket and its contents
- The IAM role
- The ALB

## Troubleshooting

### Application not deploying

Check ArgoCD application status:
```bash
kubectl get application YOUR-TEAM-APP-NAME -n argocd -o yaml
```

Look for sync errors or health status.

### S3 bucket not created

Check ACK S3 controller:
```bash
kubectl get bucket -n YOUR-TEAM
kubectl describe bucket -n YOUR-TEAM
```

Verify ACK S3 controller is running:
```bash
kubectl get pods -n ack-system | grep s3
```

### ALB not provisioning

Check Ingress status:
```bash
kubectl describe ingress YOUR-TEAM-web -n YOUR-TEAM
```

Verify AWS Load Balancer Controller is running:
```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```

### Pod not starting

Check pod logs:
```bash
kubectl get pods -n YOUR-TEAM
kubectl logs POD-NAME -n YOUR-TEAM
kubectl describe pod POD-NAME -n YOUR-TEAM
```

Common issues:
- Image pull errors
- IAM role not ready
- Resource limits too low

### Upload job failed

Check job status:
```bash
kubectl get jobs -n YOUR-TEAM
kubectl logs job/YOUR-TEAM-s3-upload -n YOUR-TEAM
```

Common issues:
- IAM permissions not ready
- S3 bucket not created yet
- Pod Identity not configured

## What You'll See

When you visit your application URL, you'll see a landing page with:
- Welcome message
- Your tenant name
- "Powered by KRO + ACK + ArgoCD"
- "Served from S3"

The page is styled with a gradient background and centered content.

## Support

- **Slack**: #platform-engineering
- **Email**: platform-team@company.com
- **Issues**: https://github.com/badrish-s/eks-platform-demo/issues
