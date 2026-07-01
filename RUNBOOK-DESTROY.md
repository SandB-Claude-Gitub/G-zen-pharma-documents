# ZenPharma — Cluster Destroy Runbook

Step-by-step instructions to cleanly destroy the DEV environment without leaving orphaned AWS resources.

> **Why this order matters**  
> Terraform does not know about Kubernetes resources created by Helm (ArgoCD, ESO, ALB Controller).  
> If the ALB Load Balancer still exists when `terraform destroy` runs, the VPC will fail to delete because AWS won't remove a VPC that still has an active Load Balancer. Similarly, ECR repositories with images must be force-deleted manually or via the AWS CLI before Terraform can remove them.

---

## Prerequisites

- `kubectl` pointing at the target cluster
- `aws` CLI authenticated to the correct account
- `helm` CLI installed
- The `ENV` variable below must match your environment (`dev` or `qa`)

```bash
export ENV=dev
export CLUSTER_NAME=pharma-${ENV}-cluster
export AWS_REGION=us-east-1
```

---

## Step 1 — Verify Cluster Connectivity

```bash
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
kubectl get nodes
```

All nodes should show `Ready`. If the cluster is already gone, skip to [Step 6](#step-6--force-delete-ecr-images).

---

## Step 2 — Delete ArgoCD Applications

ArgoCD manages Helm releases. Deleting applications through ArgoCD (not kubectl) ensures proper cleanup order.

```bash
# List all ArgoCD apps in all namespaces
kubectl get applications -n argocd

# Delete apps for this environment (adjust names to match your actual apps)
kubectl delete application pharma-ui-dev      -n argocd --ignore-not-found
kubectl delete application pharma-ui-qa       -n argocd --ignore-not-found
kubectl delete application api-gateway-dev    -n argocd --ignore-not-found
kubectl delete application auth-service-dev   -n argocd --ignore-not-found
# ... repeat for all registered apps
```

Wait until all application pods are gone:

```bash
kubectl get pods -n dev
kubectl get pods -n qa
```

---

## Step 3 — Delete Ingress Objects (Release ALB)

The ALB is provisioned by the AWS Load Balancer Controller in response to `Ingress` resources. Deleting the Ingress tells the controller to de-provision the ALB.

```bash
# Delete all ingresses in the app namespaces
kubectl delete ingress --all -n dev  --ignore-not-found
kubectl delete ingress --all -n qa   --ignore-not-found
kubectl delete ingress --all -n prod --ignore-not-found
```

Wait for the ALB to disappear in the AWS Console (EC2 → Load Balancers) or use:

```bash
# Poll until no ALBs with the cluster tag remain
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(LoadBalancerName,'pharma-${ENV}')].LoadBalancerArn" \
  --output text
```

This may take **2–5 minutes**. Do not proceed until the output is empty.

---

## Step 4 — Uninstall Helm Charts

Uninstall in reverse dependency order: application charts first, then platform charts.

```bash
# Platform charts
helm uninstall aws-load-balancer-controller -n kube-system  --ignore-not-found
helm uninstall external-secrets             -n external-secrets --ignore-not-found
helm uninstall argocd                       -n argocd        --ignore-not-found
```

---

## Step 5 — Delete Namespaces

```bash
kubectl delete namespace dev            --ignore-not-found
kubectl delete namespace qa             --ignore-not-found
kubectl delete namespace prod           --ignore-not-found
kubectl delete namespace argocd         --ignore-not-found
kubectl delete namespace external-secrets --ignore-not-found
```

Wait for all namespaces to be fully terminated:

```bash
kubectl get namespaces
# Should show only: default, kube-system, kube-public, kube-node-lease
```

---

## Step 6 — Force-Delete ECR Images

Terraform cannot delete an ECR repository that still has images. You must delete all images first, or use `force_delete = true` in the ECR module (not set by default).

Run this for every repository in your environment:

```bash
# List all repos
aws ecr describe-repositories \
  --query 'repositories[*].repositoryName' --output text

# Delete all images in a repo (repeat for each)
REPOS=(
  api-gateway
  auth-service
  drug-catalog-service
  inventory-service
  manufacturing-service
  notification-service
  pharma-ui
  supplier-service
  qc-service
)

for REPO in "${REPOS[@]}"; do
  echo "Clearing images from: $REPO"
  IMAGE_IDS=$(aws ecr list-images \
    --repository-name "$REPO" \
    --query 'imageIds[*]' \
    --output json)

  if [ "$IMAGE_IDS" != "[]" ]; then
    aws ecr batch-delete-image \
      --repository-name "$REPO" \
      --image-ids "$IMAGE_IDS"
    echo "  Cleared: $REPO"
  else
    echo "  Empty already: $REPO"
  fi
done
```

---

## Step 7 — Release Any Stuck Terraform State Lock

If a previous terraform run was interrupted, the S3 lock file may still exist:

```bash
# Replace with your actual bucket name
export TF_STATE_BUCKET=zen-pharma-terraform-state-ravdy

aws s3 rm s3://${TF_STATE_BUCKET}/envs/${ENV}/terraform.tfstate.tflock
```

---

## Step 8 — Run Terraform Destroy

You can either trigger the GitHub Actions destroy workflow (recommended) or run locally.

### Option A — GitHub Actions (recommended)

1. Go to the infra repo → **Actions** → **Terraform Infrastructure**
2. Click **Run workflow**
3. Set `action = destroy`, type `destroy` in the confirm field
4. Approve at the `dev` environment gate

### Option B — Local

```bash
cd infra/envs/${ENV}
terraform init
terraform destroy \
  -var="db_password=<your-db-password>" \
  -var="jwt_secret=<your-jwt-secret>" \
  -var="github_org=<your-github-org>" \
  -auto-approve
```

Expected output at the end:

```
Destroy complete! Resources: XX destroyed.
```

---

## Step 9 — Verify Cleanup

```bash
# Confirm EKS cluster is gone
aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION 2>&1 | grep "No cluster"

# Confirm RDS is gone
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-${ENV}')].DBInstanceIdentifier" \
  --output text

# Confirm VPC is gone
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=pharma-${ENV}-vpc" \
  --query 'Vpcs[*].VpcId' --output text
# Should return empty
```

Environment is fully destroyed. Proceed to the recreation runbook when ready.
