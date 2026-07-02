# ZenPharma — Complete Environment Setup Guide

> **Goal:** Fork the four ZenPharma repositories to your own GitHub account, configure all secrets and variables, and get the full platform running in your AWS account — infrastructure, pipelines, and applications.
>
> **Time to complete:** 2–3 hours  
> **Prerequisite level:** Comfortable with AWS, GitHub, and the command line

---

## Before You Start — Collect These Values

You will substitute these throughout the guide. Fill them in now so you have them handy.

| Placeholder | Your Value | Description |
|---|---|---|
| `<YOUR-GITHUB-USERNAME>` | | Your GitHub username or org name |
| `<YOUR-AWS-ACCOUNT-ID>` | | 12-digit AWS account ID |
| `<YOUR-TF-STATE-BUCKET>` | | S3 bucket name you will create (must be globally unique) |
| `<YOUR-DB-PASSWORD>` | | Strong password for RDS PostgreSQL |
| `<YOUR-JWT-SECRET>` | | Long random string for JWT signing |
| `<YOUR-SONAR-ORG>` | | Your SonarCloud organisation key |
| `<YOUR-SONAR-PROJECT-FRONTEND>` | | SonarCloud project key for pharma-ui |
| `<YOUR-SONAR-PROJECT-BACKEND>` | | SonarCloud project key for backend services |
| `<YOUR-SONAR-TOKEN>` | | SonarCloud API token |
| `<YOUR-GITOPS-TOKEN>` | | GitHub PAT with repo write access |

To get your AWS account ID:
```bash
aws sts get-caller-identity --query Account --output text
```

---

## Table of Contents

1. [Install Required Tools](#step-1-install-required-tools)
2. [Fork All Four Repositories](#step-2-fork-all-four-repositories)
3. [Create the S3 Terraform State Bucket](#step-3-create-the-s3-terraform-state-bucket)
4. [Update Code Files — Replace Hardcoded Values](#step-4-update-code-files--replace-hardcoded-values)
5. [Create a GitHub Personal Access Token (GITOPS_TOKEN)](#step-5-create-a-github-personal-access-token-gitops_token)
6. [Set Up SonarCloud](#step-6-set-up-sonarcloud)
7. [Configure the Infra Repository](#step-7-configure-the-infra-repository)
8. [Configure the Frontend Repository](#step-8-configure-the-frontend-repository)
9. [Configure the Backend Repository](#step-9-configure-the-backend-repository)
10. [Create Branches and Enable Branch Protection](#step-10-create-branches-and-enable-branch-protection)
11. [Run the Terraform Pipeline — Provision AWS Infrastructure](#step-11-run-the-terraform-pipeline--provision-aws-infrastructure)
12. [Bootstrap the Kubernetes Cluster](#step-12-bootstrap-the-kubernetes-cluster)
13. [Initialise the Database](#step-13-initialise-the-database)
14. [Trigger the First Application Builds](#step-14-trigger-the-first-application-builds)
15. [Verify Everything is Running](#step-15-verify-everything-is-running)
16. [Troubleshooting](#step-16-troubleshooting)

---

## Step 1: Install Required Tools

Install these on your laptop before doing anything else.

```bash
# macOS (Homebrew)
brew install awscli terraform kubectl helm python3 git yq kubectx

# Verify versions
aws --version          # >= 2.x
terraform --version    # >= 1.11
kubectl version        # >= 1.28
helm version           # >= 3.x
python3 --version      # >= 3.9
```

Configure your AWS credentials:
```bash
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region: us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity
# Should return your account ID, user ARN, and UserId
```

---

## Step 2: Fork All Four Repositories

Fork each repository to your personal GitHub account or organisation.

Go to each URL and click **Fork → Create fork**:

| Repository | URL to Fork |
|---|---|
| Infra | `https://github.com/zenpharma/infra` |
| Frontend | `https://github.com/zenpharma/frontend` |
| Backend | `https://github.com/zenpharma/backend` |
| GitOps | `https://github.com/zenpharma/gitops` |

After forking, clone all four to your laptop:

```bash
mkdir zenpharma && cd zenpharma

git clone https://github.com/<YOUR-GITHUB-USERNAME>/infra
git clone https://github.com/<YOUR-GITHUB-USERNAME>/frontend
git clone https://github.com/<YOUR-GITHUB-USERNAME>/backend
git clone https://github.com/<YOUR-GITHUB-USERNAME>/gitops
```

---

## Step 3: Create the S3 Terraform State Bucket

Terraform needs an S3 bucket to store state files before it can run. This is the only resource you create manually — everything else is managed by Terraform.

```bash
# Choose a unique bucket name — S3 names are global across all AWS accounts
export TF_STATE_BUCKET=<YOUR-TF-STATE-BUCKET>
export AWS_REGION=us-east-1

# Create the bucket
aws s3api create-bucket \
  --bucket $TF_STATE_BUCKET \
  --region $AWS_REGION

# Enable versioning (lets you recover from accidental state corruption)
aws s3api put-bucket-versioning \
  --bucket $TF_STATE_BUCKET \
  --versioning-configuration Status=Enabled

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket $TF_STATE_BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Block all public access
aws s3api put-public-access-block \
  --bucket $TF_STATE_BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

echo "State bucket created: $TF_STATE_BUCKET"
```

---

## Step 4: Update Code Files — Replace Hardcoded Values

The forked code contains hardcoded values from the original environment (AWS account ID `873135413040` and GitHub org `zenpharma`). You must replace these with your own values before pushing.

### 4a. Update Infra Repository

```bash
cd infra

# Update the S3 backend bucket name in backend.tf
sed -i '' "s/zen-pharma-terraform-state-ravdy/<YOUR-TF-STATE-BUCKET>/g" \
  envs/dev/backend.tf

# Update the GitHub org in terraform.tfvars
sed -i '' "s/github_org = \"zenpharma\"/github_org = \"<YOUR-GITHUB-USERNAME>\"/g" \
  envs/dev/terraform.tfvars

# Verify
grep "bucket" envs/dev/backend.tf
grep "github_org" envs/dev/terraform.tfvars
```

**Manual check** — open `envs/dev/terraform.tfvars` and confirm it looks like this:
```hcl
db_password = "<YOUR-DB-PASSWORD>"
jwt_secret  = "<YOUR-JWT-SECRET>"
github_org  = "<YOUR-GITHUB-USERNAME>"
```

> ⚠️ **Do NOT commit terraform.tfvars to git** — it contains your DB password and JWT secret. These values will be passed via GitHub Secrets instead. The file is only for local use.

Commit the backend.tf change only:
```bash
git add envs/dev/backend.tf
git commit -m "chore: update S3 state bucket to my account"
git push origin main
```

---

### 4b. Update GitOps Repository

All `values-*.yaml` files in the gitops repo have the original AWS account ID hardcoded in the ECR image repository URL. Replace every occurrence with your account ID.

```bash
cd ../gitops

export OLD_ACCOUNT=873135413040
export NEW_ACCOUNT=<YOUR-AWS-ACCOUNT-ID>
export OLD_ORG=zenpharma
export NEW_ORG=<YOUR-GITHUB-USERNAME>

# Replace AWS account ID in all values files
find envs/ -name "values-*.yaml" | xargs sed -i '' \
  "s/${OLD_ACCOUNT}/${NEW_ACCOUNT}/g"

# Replace GitHub org in all ArgoCD Application files
find argocd/apps/ -name "*.yaml" | xargs sed -i '' \
  "s|https://github.com/${OLD_ORG}/gitops.git|https://github.com/${NEW_ORG}/gitops.git|g"

# Verify replacements
echo "--- ECR URLs ---"
grep -r "repository:" envs/dev/ | head -5

echo "--- ArgoCD repoURLs ---"
grep -r "repoURL" argocd/apps/dev/ | head -3
```

Expected output after replacement:
```
envs/dev/values-pharma-ui.yaml:  repository: <YOUR-AWS-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/pharma-ui
argocd/apps/dev/pharma-ui-app.yaml:    repoURL: https://github.com/<YOUR-GITHUB-USERNAME>/gitops.git
```

Commit and push:
```bash
git add .
git commit -m "chore: update AWS account ID and GitHub org to my values"
git push origin main
```

---

## Step 5: Create a GitHub Personal Access Token (GITOPS_TOKEN)

The CI pipelines need a token to commit image tags to the gitops repository. This is the `GITOPS_TOKEN` secret used across all three application/infra repos.

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token (classic)**
3. Set:
   - **Note:** `zenpharma-gitops-bot`
   - **Expiration:** 90 days (or no expiration for a long-lived setup)
   - **Scopes:** Check `repo` (full control of private repositories)
4. Click **Generate token**
5. **Copy the token immediately** — you cannot see it again

Save this as `<YOUR-GITOPS-TOKEN>`.

---

## Step 6: Set Up SonarCloud

SonarCloud provides SAST (static application security testing). Both the frontend and backend pipelines require it.

### 6a. Create a SonarCloud Account

1. Go to [sonarcloud.io](https://sonarcloud.io)
2. Click **Log in with GitHub**
3. Authorise SonarCloud to access your GitHub account

### 6b. Create a SonarCloud Organisation

1. Click **+** (top right) → **Create new organisation**
2. Choose **Import from GitHub**
3. Select your GitHub account
4. Install the SonarCloud GitHub App when prompted
5. Select which repos to import: at minimum select `frontend` and `backend`
6. Choose the **Free plan**
7. Your organisation key will be your GitHub username (e.g., `your-github-username`)

Save this as `<YOUR-SONAR-ORG>`.

### 6c. Create the Frontend Project

1. In SonarCloud, click **+** → **Analyze new project**
2. Select your `frontend` repository
3. Click **Set Up**
4. Choose **With GitHub Actions** as the CI method
5. **IMPORTANT:** On the setup page, click **Administration → Analysis Method → Disable Automatic Analysis**
   > This is required. If Automatic Analysis is ON, SonarCloud will conflict with the CI-based analysis in the pipeline and fail.
6. Note the `projectKey` shown on screen — it typically looks like `<YOUR-SONAR-ORG>_frontend`

Save this as `<YOUR-SONAR-PROJECT-FRONTEND>`.

### 6d. Create the Backend Project

Repeat the same steps for the `backend` repository:

1. Click **+** → **Analyze new project** → select `backend`
2. Choose **With GitHub Actions**
3. **Disable Automatic Analysis** (Administration → Analysis Method)
4. Note the `projectKey` (e.g., `<YOUR-SONAR-ORG>_backend`)

Save this as `<YOUR-SONAR-PROJECT-BACKEND>`.

### 6e. Generate a SonarCloud Token

1. Go to **My Account → Security** (top-right menu → My Account → Security tab)
2. In **Generate Tokens**, type `zenpharma-ci` and click **Generate**
3. Copy the token

Save this as `<YOUR-SONAR-TOKEN>`.

---

## Step 7: Configure the Infra Repository

### 7a. Add Repository Secrets

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/infra` → **Settings → Secrets and variables → Actions → Secrets**

Click **New repository secret** and add each of these:

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your AWS IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | Your AWS IAM secret access key |

> These are the credentials Terraform uses to create AWS infrastructure. Use an IAM user with `AdministratorAccess` policy for initial setup. You can restrict permissions later.

### 7b. Add Repository Variables

Go to: **Settings → Secrets and variables → Actions → Variables** tab

Click **New repository variable** and add:

| Variable Name | Value |
|---|---|
| `GH_ORG` | `<YOUR-GITHUB-USERNAME>` |
| `TF_STATE_BUCKET` | `<YOUR-TF-STATE-BUCKET>` |

### 7c. Create the `dev` GitHub Environment

GitHub Environments add a manual approval gate before the Terraform apply job runs.

Go to: **Settings → Environments → New environment**

1. Name: `dev`
2. Click **Configure environment**
3. Under **Deployment protection rules**, enable **Required reviewers**
4. Add yourself as a reviewer
5. Click **Save protection rules**

### 7d. Add Environment-Scoped Secrets

Still on the `dev` environment page, scroll to **Environment secrets** and add:

| Secret Name | Value |
|---|---|
| `DEV_DB_PASSWORD` | `<YOUR-DB-PASSWORD>` |
| `DEV_JWT_SECRET` | `<YOUR-JWT-SECRET>` |

> These secrets are only visible to pipeline jobs that target the `dev` environment — they cannot be read by other workflows.

---

## Step 8: Configure the Frontend Repository

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/frontend`

### 8a. Add Repository Secrets

**Settings → Secrets and variables → Actions → Secrets:**

| Secret Name | Value | Purpose |
|---|---|---|
| `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` | Constructs the OIDC IAM role ARN |
| `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` | SonarCloud authentication |
| `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` | Allows CI to commit image tags to gitops repo |

### 8b. Add Repository Variables

**Settings → Secrets and variables → Actions → Variables:**

| Variable Name | Value | Purpose |
|---|---|---|
| `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` | The gitops repo the pipeline commits to |
| `SONAR_ORG` | `<YOUR-SONAR-ORG>` | SonarCloud organisation key |
| `SONAR_PROJECT_KEY_FRONTEND` | `<YOUR-SONAR-PROJECT-FRONTEND>` | SonarCloud project key for frontend |

### 8c. Create the `dev` GitHub Environment

**Settings → Environments → New environment:**

1. Name: `dev`
2. No required reviewers needed here (frontend deploy to dev is automatic)
3. Click **Save**

---

## Step 9: Configure the Backend Repository

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/backend`

### 9a. Add Repository Secrets

**Settings → Secrets and variables → Actions → Secrets:**

| Secret Name | Value | Purpose |
|---|---|---|
| `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` | Constructs the OIDC IAM role ARN |
| `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` | SonarCloud authentication |
| `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` | Allows CI to commit image tags to gitops repo |

### 9b. Add Repository Variables

**Settings → Secrets and variables → Actions → Variables:**

| Variable Name | Value | Purpose |
|---|---|---|
| `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` | The gitops repo the pipeline commits to |
| `SONAR_ORG` | `<YOUR-SONAR-ORG>` | SonarCloud organisation key |
| `SONAR_PROJECT_KEY_BACKEND` | `<YOUR-SONAR-PROJECT-BACKEND>` | SonarCloud project key for backend |

### 9c. Create the `dev` GitHub Environment

**Settings → Environments → New environment:**

1. Name: `dev`
2. No required reviewers for auto-deploy to dev
3. Click **Save**

---

## Step 10: Create Branches and Enable Branch Protection

### 10a. Create `develop` Branch in Frontend and Backend

The CI pipelines trigger on pushes to the `develop` branch. Create it from `main`:

```bash
# Frontend
cd frontend
git checkout -b develop
git push origin develop

# Backend
cd ../backend
git checkout -b develop
git push origin develop
```

### 10b. Enable Branch Protection on `main`

Do this for each of the four repositories:

1. Go to **Settings → Branches → Add branch protection rule**
2. Branch name pattern: `main`
3. Enable:
   - ✅ **Require a pull request before merging**
   - ✅ **Require approvals** (1 reviewer)
   - ✅ **Require status checks to pass before merging**
4. Click **Create**

This prevents anyone from pushing directly to `main` — all changes go through a PR.

---

## Step 11: Run the Terraform Pipeline — Provision AWS Infrastructure

This is the most important step. The Terraform pipeline creates all AWS resources: VPC, EKS cluster, RDS, ECR repos, IAM roles, and Secrets Manager entries.

### 11a. Trigger the Pipeline

The infra pipeline triggers on push to `main`. Make a small commit to trigger it:

```bash
cd infra
echo "# ZenPharma Infrastructure" >> README.md
git add README.md
git commit -m "chore: trigger initial terraform pipeline"
git push origin main
```

### 11b. Watch the Pipeline

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/infra` → **Actions** tab

You should see the **Terraform Infrastructure** workflow running with two jobs:
1. **Terraform Plan** — runs automatically, shows what will be created
2. **Terraform Apply** — waits for your approval

### 11c. Review and Approve the Plan

1. Click the running workflow
2. Click the **Terraform Apply** job — it shows **Waiting for approval**
3. Click **Review deployments**
4. Select the `dev` environment checkbox
5. Click **Approve and deploy**

The apply job now runs. It will take approximately **15–20 minutes** to complete because EKS cluster creation takes the longest.

**Expected output at the end:**
```
Apply complete! Resources: 45 added, 0 changed, 0 destroyed.
```

### 11d. Confirm Key AWS Resources Were Created

```bash
# EKS cluster is ACTIVE
aws eks describe-cluster \
  --name pharma-dev-cluster \
  --region us-east-1 \
  --query 'cluster.status' \
  --output text
# Expected: ACTIVE

# RDS is available
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-dev')].DBInstanceStatus" \
  --output text
# Expected: available

# ECR repositories created
aws ecr describe-repositories \
  --query 'repositories[*].repositoryName' \
  --output table
# Expected: 9 repos (api-gateway, auth-service, drug-catalog-service, ...)
```

---

## Step 12: Bootstrap the Kubernetes Cluster

Terraform created the AWS infrastructure but did not install anything inside Kubernetes. Run three Python scripts in order.

### 12a. Connect kubectl to the Cluster

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pharma-dev-cluster \
  --alias dev

kubectl get nodes
# Expected: 4 nodes in Ready state (may take 2–3 minutes after cluster is active)
```

### 12b. Script 01 — Install AWS Load Balancer Controller

```bash
cd infra
python3 scripts/01_install_prerequisites.py
```

The script will prompt you for:
- **CLUSTER_NAME** → `pharma-dev-cluster` (press Enter for default)
- **AWS_REGION** → `us-east-1` (press Enter for default)
- **ALB_CONTROLLER_ROLE_ARN** → get this from AWS:
  ```bash
  aws iam list-roles --query "Roles[?contains(RoleName,'alb-controller')].Arn" --output text
  ```

What it does:
- Updates your kubeconfig
- Adds the `eks-charts` Helm repo
- Installs the AWS Load Balancer Controller into the `kube-system` namespace

**Verify:**
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
# Expected: READY 2/2
```

### 12c. Script 02 — Bootstrap ArgoCD

```bash
python3 scripts/02_bootstrap_argocd.py
```

The script will prompt you for:
- **GITOPS_REPO_URL** → `https://github.com/<YOUR-GITHUB-USERNAME>/gitops.git`
- **GITHUB_TOKEN** → `<YOUR-GITOPS-TOKEN>` (PAT from Step 5)

What it does:
- Installs ArgoCD via Helm into the `argocd` namespace
- Registers your gitops repo in ArgoCD
- Creates the `pharma` AppProject

**Verify:**
```bash
kubectl get pods -n argocd
# All pods should show Running status

# Get initial ArgoCD admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
# Save this password — you'll use it to log into ArgoCD UI
```

**Access ArgoCD UI:**
```bash
# Port-forward the ArgoCD server to your laptop
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open: https://localhost:8080
# Username: admin
# Password: (from the command above)
```

### 12d. Script 03 — Setup External Secrets Operator

```bash
python3 scripts/03_setup_external_secrets.py
```

What it does:
- Installs External Secrets Operator via Helm into `external-secrets` namespace
- Creates a `ClusterSecretStore` pointing at AWS Secrets Manager
- Creates `ExternalSecret` resources in the `dev` namespace

**Verify:**
```bash
kubectl get pods -n external-secrets
# All pods should show Running

kubectl get externalsecret -n dev
# STATUS column should show: SecretSynced
# If SecretSyncedError: see Troubleshooting section
```

---

## Step 13: Initialise the Database

The RDS instance was created by Terraform but the database schemas are empty. Run the init script to create the per-service schemas.

### 13a. Get the RDS Endpoint

```bash
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-dev')].Endpoint.Address" \
  --output text
# Example output: pharma-dev-postgres.abc123.us-east-1.rds.amazonaws.com
```

### 13b. Run the Schema Init Script

```bash
./scripts/init-database.sh
```

Prompts:
- **RDS endpoint** → paste the hostname from above
- **Database password** → `<YOUR-DB-PASSWORD>`

The script will:
1. Create a temporary PostgreSQL pod inside the `dev` namespace
2. Copy `db-init/01-schemas.sql` into the pod
3. Connect to RDS and create these schemas: `auth`, `drug_catalog`, `inventory`, `manufacturing`, `quality_control`, `supplier`, `distribution`, `reporting`
4. Print the schema list as verification
5. Delete the temporary pod

Expected output:
```
[OK]  Schema SQL executed.
      Verifying schemas...
  Schema Name   | Owner
 auth           | pharmaadmin
 drug_catalog   | pharmaadmin
 ...
[OK]  Database initialization complete.
```

---

## Step 14: Trigger the First Application Builds

Now push code to the `develop` branch to trigger the CI pipelines. They will build Docker images and push them to ECR.

### 14a. Frontend — Push to Develop

```bash
cd ../frontend
git checkout develop

# Make a trivial change to trigger CI
echo "# pharma-ui" >> README.md
git add README.md
git commit -m "chore: trigger initial CI build"
git push origin develop
```

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/frontend` → **Actions**

Watch the **CI/CD — pharma-ui** workflow. It runs these stages:
```
Lint → Unit Tests → SonarCloud → Build → Docker Build + Trivy → Push to ECR → Deploy to DEV
```

The final **Deploy — DEV** job commits a line like:
```
ci(dev): update pharma-ui → sha-abc1234
```
to the gitops repo's `envs/dev/values-pharma-ui.yaml`.

### 14b. Backend — Push to Develop

```bash
cd ../backend
git checkout develop

# Make a trivial change
echo "# ZenPharma Backend" >> README.md
git add README.md
git commit -m "chore: trigger initial CI builds"
git push origin develop
```

This triggers **all 8 service CI pipelines** simultaneously (each filtered by its own path, but since no service files changed, only pipelines matching `README.md` or unfiltered triggers will run — push a change inside each service directory to trigger its specific pipeline if needed).

To trigger a specific service build:
```bash
# Trigger api-gateway pipeline
touch api-gateway/src/main/resources/application.yml
git add api-gateway/
git commit -m "chore: trigger api-gateway build"
git push origin develop
```

### 14c. Watch ArgoCD Deploy the Images

Once the CI pipeline commits the new image tag to the gitops repo, ArgoCD detects the change within ~3 minutes and deploys.

```bash
# Watch ArgoCD sync status
kubectl get applications -n argocd -w
# STATUS column should change from OutOfSync → Synced
# HEALTH column should show: Healthy
```

In the ArgoCD UI (port-forward from Step 12c), you should see all application cards turn green.

---

## Step 15: Verify Everything is Running

### 15a. Check All Pods

```bash
kubectl get pods -n dev
```

Expected output — all pods should be `Running` with `1/1` Ready:
```
NAME                                    READY   STATUS    RESTARTS   AGE
api-gateway-xxx                         1/1     Running   0          5m
auth-service-xxx                        1/1     Running   0          5m
drug-catalog-service-xxx               1/1     Running   0          5m
inventory-service-xxx                   1/1     Running   0          5m
manufacturing-service-xxx               1/1     Running   0          5m
notification-service-xxx                1/1     Running   0          5m
pharma-ui-xxx                           1/1     Running   0          5m
qc-service-xxx                          1/1     Running   0          5m
supplier-service-xxx                    1/1     Running   0          5m
```

### 15b. Get the Application URL

```bash
kubectl get ingress -n dev
```

The `ADDRESS` column contains the ALB DNS name. It may take **2–5 minutes** after the first pod starts for the ALB to be provisioned and health checks to pass.

```bash
# Test the application responds
ALB_URL=$(kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
curl -s -o /dev/null -w "%{http_code}" http://$ALB_URL/
# Expected: 200
```

Open `http://$ALB_URL` in your browser. You should see the ZenPharma login page.

**Login:** `admin` / `changeme`

### 15c. Check Secrets Are Synced

```bash
kubectl get externalsecret -n dev
# All should show STATUS: SecretSynced
```

### 15d. Check SonarCloud Reports

Go to [sonarcloud.io](https://sonarcloud.io) → your organisation. You should see:
- A quality report for `frontend` with code coverage and analysis results
- A quality report for `backend` with test coverage and SAST results

### 15e. Final Checklist

```
✅ kubectl get nodes          → 4 nodes Ready
✅ kubectl get pods -n dev    → 9 pods Running
✅ kubectl get applications -n argocd → all Synced + Healthy
✅ kubectl get externalsecret -n dev  → all SecretSynced
✅ Browser: http://<ALB-URL>  → login page loads
✅ SonarCloud                 → analysis reports visible
✅ ECR                        → images present in all 9 repos
```

---

## Step 16: Troubleshooting

### Pod is in `ImagePullBackOff`

The ECR URL in the values file does not match the actual ECR URL.

```bash
kubectl describe pod <pod-name> -n dev | grep -A5 "Events"
# Look for: "failed to pull image" — check the image URL

# Verify ECR repo exists with your account ID
aws ecr describe-repositories --repository-names pharma-ui
# The URI should be: <YOUR-AWS-ACCOUNT-ID>.dkr.ecr.us-east-1.amazonaws.com/pharma-ui

# Fix: ensure you replaced 873135413040 with your account ID in gitops/envs/dev/values-*.yaml
grep "repository:" gitops/envs/dev/values-pharma-ui.yaml
```

### Pod is in `CrashLoopBackOff`

```bash
kubectl logs <pod-name> -n dev --previous
# Most common cause: missing environment variable (secret not synced yet)

kubectl get externalsecret -n dev
# If STATUS is SecretSyncedError:
kubectl describe externalsecret <name> -n dev | grep -A10 "Status"
```

### ExternalSecret shows `SecretSyncedError`

The ESO pod cannot read from Secrets Manager. Check the IAM role:

```bash
# Verify the IAM role exists
aws iam get-role --role-name pharma-dev-eks-role

# Verify secrets exist in Secrets Manager
aws secretsmanager list-secrets \
  --query 'SecretList[*].Name' --output text
# Should show: pharma-dev-db-secret, pharma-dev-jwt-secret
```

### Terraform Apply Fails — `Error creating S3 bucket`

The bucket name already exists in another AWS account. S3 names are globally unique. Choose a different name:

```bash
# Try appending your name or a random suffix
export TF_STATE_BUCKET=zen-pharma-tf-state-<yourname>-2026

# Update backend.tf
sed -i '' "s/zen-pharma-terraform-state-ravdy/$TF_STATE_BUCKET/g" infra/envs/dev/backend.tf
```

### Terraform Apply Fails — `Error: Provider produced inconsistent final plan`

Retry the apply — this is a transient AWS API issue:

```bash
# In the GitHub Actions UI, click "Re-run failed jobs"
# Or trigger manually via workflow_dispatch with action=apply
```

### ArgoCD Shows `OutOfSync` After Image Push

Check the gitops repo was updated by the CI pipeline:

```bash
cd gitops
git log --oneline envs/dev/values-pharma-ui.yaml
# Should show a recent commit like "ci(dev): update pharma-ui → sha-abc1234"
```

If the commit is there but ArgoCD is still out of sync, trigger a manual sync:

```bash
# Via ArgoCD CLI
argocd app sync pharma-ui-dev

# Or via kubectl
kubectl patch application pharma-ui-dev -n argocd \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{}}}'
```

### SonarCloud Analysis Fails — `Automatic Analysis not disabled`

Go to SonarCloud → project → **Administration → Analysis Method** → **switch off Automatic Analysis**.  
Then re-run the CI pipeline.

### S3 State Lock Stuck

If a pipeline was cancelled mid-run:

```bash
aws s3 rm s3://<YOUR-TF-STATE-BUCKET>/envs/dev/terraform.tfstate.tflock
```

---

## Complete Secrets and Variables Reference

Use this as a final checklist to verify all values are configured correctly.

### zenpharma/infra

| Type | Name | Value |
|---|---|---|
| Secret | `AWS_ACCESS_KEY_ID` | IAM access key |
| Secret | `AWS_SECRET_ACCESS_KEY` | IAM secret key |
| Variable | `GH_ORG` | `<YOUR-GITHUB-USERNAME>` |
| Variable | `TF_STATE_BUCKET` | `<YOUR-TF-STATE-BUCKET>` |
| Env Secret (`dev`) | `DEV_DB_PASSWORD` | `<YOUR-DB-PASSWORD>` |
| Env Secret (`dev`) | `DEV_JWT_SECRET` | `<YOUR-JWT-SECRET>` |

### zenpharma/frontend

| Type | Name | Value |
|---|---|---|
| Secret | `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` |
| Secret | `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` |
| Secret | `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` |
| Variable | `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` |
| Variable | `SONAR_ORG` | `<YOUR-SONAR-ORG>` |
| Variable | `SONAR_PROJECT_KEY_FRONTEND` | `<YOUR-SONAR-PROJECT-FRONTEND>` |

### zenpharma/backend

| Type | Name | Value |
|---|---|---|
| Secret | `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` |
| Secret | `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` |
| Secret | `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` |
| Variable | `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` |
| Variable | `SONAR_ORG` | `<YOUR-SONAR-ORG>` |
| Variable | `SONAR_PROJECT_KEY_BACKEND` | `<YOUR-SONAR-PROJECT-BACKEND>` |

### zenpharma/gitops (code changes, not secrets)

| File | Change |
|---|---|
| `envs/dev/values-*.yaml` (all 9) | Replace `873135413040` with `<YOUR-AWS-ACCOUNT-ID>` |
| `envs/dev/values-api-gateway.yaml` | Replace IAM role ARN account ID |
| `argocd/apps/dev/*.yaml` (all 9) | Replace `zenpharma` with `<YOUR-GITHUB-USERNAME>` in `repoURL` |

---

## What You Have After This Setup

```
GitHub
├── <YOUR-GITHUB-USERNAME>/infra      → Terraform pipeline (auto plan, manual apply)
├── <YOUR-GITHUB-USERNAME>/frontend   → React app CI/CD (auto deploy to dev on push)
├── <YOUR-GITHUB-USERNAME>/backend    → 8 Java services CI/CD (auto deploy to dev on push)
└── <YOUR-GITHUB-USERNAME>/gitops     → Source of truth for what runs in Kubernetes

AWS (us-east-1)
├── VPC with public/private/database subnets
├── EKS cluster: pharma-dev-cluster (4 × t3.small nodes)
├── RDS PostgreSQL: pharmadb (8 schemas initialised)
├── ECR: 9 image repositories
├── IAM: OIDC roles for GitHub Actions and pod identity
└── Secrets Manager: DB credentials + JWT secret

Kubernetes
├── Namespace dev:      9 microservices running
├── Namespace argocd:   ArgoCD (GitOps controller)
├── Namespace external-secrets: ESO (secret syncing)
└── Namespace kube-system: AWS Load Balancer Controller

Application
└── pharma-ui accessible at: http://<ALB-DNS-NAME>
    Login: admin / changeme
```
