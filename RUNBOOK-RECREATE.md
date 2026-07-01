# ZenPharma — Cluster Recreation Runbook

Step-by-step instructions to bring the DEV environment back up from zero after a destroy.

> **Estimated time:** 25–35 minutes total  
> Terraform + EKS node provisioning: ~15 min  
> Bootstrap scripts: ~8 min  
> Database init: ~2 min

---

## Prerequisites

- `aws` CLI authenticated to the correct AWS account
- `terraform` >= 1.11 installed
- `kubectl`, `helm`, `python3` installed
- You are in the `infra` repo root directory

```bash
cd /path/to/zenpharma/infra

export ENV=dev
export CLUSTER_NAME=pharma-${ENV}-cluster
export AWS_REGION=us-east-1
export TF_STATE_BUCKET=zen-pharma-terraform-state-ravdy
```

---

## Phase 1 — Terraform: Provision All AWS Infrastructure

This creates: VPC, EKS cluster + node group, RDS PostgreSQL, ECR repositories, IAM roles, and Secrets Manager entries.

### Option A — GitHub Actions (recommended for course demo)

1. Go to the infra repo → **Actions** → **Terraform Infrastructure**
2. Click **Run workflow**
3. Set `action = apply`
4. The plan job runs first → approve at the `dev` environment gate → apply runs

Monitor the workflow. When the apply job shows green, all AWS infrastructure is ready.

### Option B — Local

```bash
cd envs/${ENV}
terraform init
terraform plan \
  -var="db_password=<your-db-password>" \
  -var="jwt_secret=<your-jwt-secret>" \
  -var="github_org=<your-github-org>" \
  -out=tfplan

terraform apply tfplan
```

### Verify Infrastructure

```bash
# EKS cluster is ACTIVE
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query 'cluster.status' --output text
# Expected: ACTIVE

# RDS is available
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-${ENV}')].DBInstanceStatus" \
  --output text
# Expected: available
```

---

## Phase 2 — Bootstrap the Cluster

The bootstrap scripts install and configure everything the cluster needs to run workloads. They must be run **in order** after every fresh terraform apply — Kubernetes resources do not survive a cluster destroy.

```bash
cd infra   # run all scripts from the infra/ directory
```

### Script 01 — Install Prerequisites (AWS LB Controller + kubeconfig)

```bash
python3 scripts/01_install_prerequisites.py
```

This script will prompt you for:
- `CLUSTER_NAME` → `pharma-dev-cluster`
- `AWS_REGION` → `us-east-1`
- `ALB_CONTROLLER_ROLE_ARN` → from Terraform outputs or IAM console

It will:
1. Update your local kubeconfig (`aws eks update-kubeconfig`)
2. Install the AWS Load Balancer Controller (Helm chart into `kube-system`)

Verify:
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
# Should show READY 2/2
```

### Script 02 — Bootstrap ArgoCD

```bash
python3 scripts/02_bootstrap_argocd.py
```

This script will prompt you for:
- GitOps repo URL (SSH or HTTPS)
- GitHub deploy key or PAT

It will:
1. Install ArgoCD via Helm into the `argocd` namespace
2. Register your GitOps repo in ArgoCD
3. Create the `pharma` AppProject

Verify:
```bash
kubectl get pods -n argocd
# All pods should be Running/Ready

# Get ArgoCD initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### Script 03 — Setup External Secrets Operator

```bash
python3 scripts/03_setup_external_secrets.py
```

This script will:
1. Install External Secrets Operator (ESO) via Helm into `external-secrets` namespace
2. Create a `ClusterSecretStore` pointing at AWS Secrets Manager
3. Create the `ExternalSecret` resources in the `dev` namespace

Verify:
```bash
kubectl get pods -n external-secrets
# All pods should be Running

kubectl get externalsecret -n dev
# Should list: pharma-dev-db-secret, pharma-dev-jwt-secret (or similar)
# STATUS column should show: SecretSynced
```

If `STATUS` shows `SecretSyncedError`, check that Secrets Manager entries exist:
```bash
aws secretsmanager list-secrets \
  --query 'SecretList[*].Name' --output text | tr '\t' '\n' | grep pharma
```

---

## Phase 3 — Create Database Schemas

The RDS instance is empty after creation. Run the schema init script to create per-service PostgreSQL schemas.

First, get the RDS endpoint:
```bash
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-${ENV}')].Endpoint.Address" \
  --output text
# Copy this value — you will paste it into the script prompt
```

Then run the script:
```bash
./scripts/init-database.sh
```

The script prompts for:
- **RDS endpoint** → paste the hostname from the command above
- **Database password** → your DB password (same value as `DEV_DB_PASSWORD` secret)

The script will:
1. Create a temporary `pg-client` pod inside the `dev` namespace (so it can reach the private RDS)
2. Copy and run `db-init/01-schemas.sql` against `pharmadb`
3. Print the list of created schemas
4. Delete the temporary pod

Expected output at the end:
```
[OK]  Schema SQL executed.
      Verifying schemas...
 Schema Name |  Owner
 ...
[OK]  Database initialization complete.
```

---

## Phase 4 — Verify the Full Stack

```bash
# All nodes ready
kubectl get nodes

# ArgoCD running
kubectl get pods -n argocd

# ESO running
kubectl get pods -n external-secrets

# ALB Controller running
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Secrets synced
kubectl get externalsecret -n dev

# ECR repos exist (confirm Terraform created them)
aws ecr describe-repositories \
  --query 'repositories[*].repositoryName' --output text
```

---

## Hola — Infrastructure is Ready to Serve Traffic!

At this point you have:

| Component | Status |
|---|---|
| VPC + subnets | Provisioned by Terraform |
| EKS cluster (4 nodes) | Active |
| RDS PostgreSQL | Available, schemas created |
| ECR repositories (9 repos) | Ready for image pushes |
| IAM roles (OIDC) | Ready for GitHub Actions CI |
| Secrets Manager | DB credentials + JWT stored |
| AWS Load Balancer Controller | Running in kube-system |
| ArgoCD | Running, GitOps repo registered |
| External Secrets Operator | Running, syncing secrets from AWS |

### Next: Deploy pharma-ui

To deploy the frontend application:
1. Push a commit to the `pharma-ui` app repo → CI builds and pushes the image to ECR
2. The CI workflow opens a PR to update `envs/dev/values-pharma-ui.yaml` with the new image tag
3. Merge the PR → ArgoCD detects the change and deploys the pod
4. An ALB is created when ArgoCD applies the `Ingress` manifest

Or refer to **Module 6** for the full step-by-step deployment guide.
