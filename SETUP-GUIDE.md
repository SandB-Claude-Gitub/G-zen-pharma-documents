# ZenPharma — Complete Environment Setup Guide

> **Goal:** Get the full ZenPharma platform running in your AWS account — infrastructure, pipelines, and applications.
>
> **Strategy:** Terraform takes 20-30 minutes to run. We start it as fast as possible (Phase 1 ≈ 15 min of your time), then collect all remaining inputs while it runs (Phase 2). You approve the deploy and wait — everything else is ready when Terraform finishes.
>
> **Total hands-on time:** ~45 minutes across 3 phases + 20-30 minutes of waiting
>
> **Prerequisite level:** Comfortable with AWS, GitHub, and the command line

---

## Prerequisites

Install these tools on your laptop before starting:

```bash
# macOS (Homebrew)
brew install awscli git

# Verify
aws --version        # >= 2.x
git --version        # any recent version
```

Configure your AWS CLI with an account that has admin access:
```bash
aws configure
# AWS Access Key ID:     <your personal admin key>
# AWS Secret Access Key: <your personal admin secret>
# Default region:        us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity
```

> `kubectl`, `helm`, and `python3` are needed later (Phase 3). You can install them while Terraform runs.

---

## Values Reference

Two tables — fill in Phase 1 values before starting, Phase 2 values while Terraform runs.

### Phase 1 — Required Before Terraform Can Run

| Placeholder | Your Value | How to Get It |
|---|---|---|
| `<YOUR-GITHUB-USERNAME>` | | Your GitHub username or org name |
| `<YOUR-AWS-ACCOUNT-ID>` | | `aws sts get-caller-identity --query Account --output text` |
| `<YOUR-TF-STATE-BUCKET>` | | Name you pick (must be globally unique, e.g. `zen-pharma-tf-YOURNAME`) |
| `<TF-IAM-ACCESS-KEY-ID>` | | From Step 1a — `aws iam create-access-key` output |
| `<TF-IAM-SECRET-ACCESS-KEY>` | | From Step 1a — `aws iam create-access-key` output |
| `<YOUR-DB-PASSWORD>` | | Strong password you choose for RDS |
| `<YOUR-JWT-SECRET>` | | Long random string (run: `openssl rand -hex 32`) |

### Phase 2 — Collect While Terraform Runs

| Placeholder | Your Value | How to Get It |
|---|---|---|
| `<YOUR-GITOPS-TOKEN>` | | GitHub PAT — Step 6 |
| `<YOUR-SONAR-ORG>` | | SonarCloud org key — Step 7 |
| `<YOUR-SONAR-PROJECT-FRONTEND>` | | SonarCloud project key for pharma-ui |
| `<YOUR-SONAR-PROJECT-BACKEND>` | | SonarCloud project key for backend |
| `<YOUR-SONAR-TOKEN>` | | SonarCloud API token |

---

## Table of Contents

### Phase 1 — Launch Terraform *(~15 min of your time)*

- Step 1: [Create Manual AWS Resources — S3 + IAM](#step-1-create-manual-aws-resources--s3--iam)
- Step 2: [Fork All Four Repositories](#step-2-fork-all-four-repositories)
- Step 3: [Update Infra Code — S3 Bucket Name](#step-3-update-infra-code--s3-bucket-name)
- Step 4: [Configure the Infra Repository in GitHub](#step-4-configure-the-infra-repository-in-github)
- Step 5: [Trigger Terraform — Approve and Start](#step-5-trigger-terraform--approve-and-start)

> ⏱️ **Terraform is now running — 20-30 minutes → go to Phase 2**

### Phase 2 — While Terraform Runs

- Step 6: [Create GitHub PAT — GITOPS_TOKEN](#step-6-create-github-pat--gitops_token)
- Step 7: [Set Up SonarCloud](#step-7-set-up-sonarcloud)
- Step 8: [Update GitOps Repository Code](#step-8-update-gitops-repository-code)
- Step 9: [Configure Frontend Repository](#step-9-configure-frontend-repository)
- Step 10: [Configure Backend Repository](#step-10-configure-backend-repository)
- Step 11: [Create Branches and Enable Branch Protection](#step-11-create-branches-and-enable-branch-protection)

### Phase 3 — After Terraform Completes

- Step 12: [Verify Terraform Outputs](#step-12-verify-terraform-outputs)
- Step 13: [Bootstrap the Kubernetes Cluster](#step-13-bootstrap-the-kubernetes-cluster)
- Step 14: [Initialise the Database](#step-14-initialise-the-database)
- Step 15: [Trigger the First Application Builds](#step-15-trigger-the-first-application-builds)
- Step 16: [Verify Everything is Running](#step-16-verify-everything-is-running)
- Step 17: [Troubleshooting](#step-17-troubleshooting)

---

# Phase 1 — Launch Terraform

> **Goal of Phase 1:** Get Terraform running on GitHub Actions as fast as possible.
> You need just 2 manual AWS resources and 6 GitHub secrets — then you approve the pipeline and move on.

---

## Step 1: Create Manual AWS Resources — IAM + S3

Terraform manages all AWS infrastructure except these two, which must exist before Terraform can run:

- **IAM user `terraform-user`** — GitHub Actions uses its access keys to run Terraform
- **S3 bucket** — stores Terraform state files

We create IAM first, then configure AWS CLI with those credentials, then create the S3 bucket — so everything from this point runs under `terraform-user`.

### 1a. Create IAM User `terraform-user`

Run this with your personal AWS admin credentials (whatever is currently in `aws configure`):

```bash
# Create the user
aws iam create-user --user-name terraform-user

# Attach AdministratorAccess (needed to create EKS, RDS, IAM roles, VPC, etc.)
aws iam attach-user-policy \
  --user-name terraform-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create access keys — COPY THE OUTPUT NOW, you cannot retrieve it again
aws iam create-access-key --user-name terraform-user
```

Output:
```json
{
  "AccessKey": {
    "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  }
}
```

Save both values — you need them in two places:
- **Step 4a** → GitHub Secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- **Right now** → configure AWS CLI (next step)

### 1b. Configure AWS CLI with `terraform-user`

Switch your AWS CLI to use `terraform-user` credentials so the S3 bucket is created under the same identity that Terraform will use:

```bash
aws configure
# AWS Access Key ID:     <AccessKeyId from above>
# AWS Secret Access Key: <SecretAccessKey from above>
# Default region:        us-east-1
# Default output format: json

# Verify you are now acting as terraform-user
aws sts get-caller-identity
# "Arn" should show: arn:aws:iam::<account-id>:user/terraform-user
```

### 1c. Create the S3 State Bucket

Now running as `terraform-user`:

```bash
export TF_STATE_BUCKET=<YOUR-TF-STATE-BUCKET>
export AWS_REGION=us-east-1

# Create bucket
aws s3api create-bucket \
  --bucket $TF_STATE_BUCKET \
  --region $AWS_REGION

# Enable versioning (allows recovery from accidental state corruption)
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

echo "State bucket ready: $TF_STATE_BUCKET"
```

---

## Step 2: Fork All Four Repositories

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

## Step 3: Update Infra Code — S3 Bucket Name

The only code change needed before Terraform can run is updating the S3 backend configuration with your bucket name.

```bash
cd infra

# Replace the original bucket name with yours
sed -i '' "s/zen-pharma-terraform-state-ravdy/<YOUR-TF-STATE-BUCKET>/g" \
  envs/dev/backend.tf

# Verify
grep "bucket" envs/dev/backend.tf
# Expected: bucket = "<YOUR-TF-STATE-BUCKET>"

# Commit and push
git add envs/dev/backend.tf
git commit -m "chore: update S3 state bucket to my account"
git push origin main
```

> That is the only file that needs to change before Terraform runs. The GitHub org and account ID are passed via GitHub Secrets and Variables — no hardcoded values needed.

---

## Step 4: Configure the Infra Repository in GitHub

This is where you add everything GitHub Actions needs to run Terraform. Do all of this at:
`https://github.com/<YOUR-GITHUB-USERNAME>/infra → Settings`

### 4a. Add Repository Secrets

**Settings → Secrets and variables → Actions → Secrets → New repository secret**

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | `<TF-IAM-ACCESS-KEY-ID>` (from Step 1b) |
| `AWS_SECRET_ACCESS_KEY` | `<TF-IAM-SECRET-ACCESS-KEY>` (from Step 1b) |

### 4b. Add Repository Variables

**Settings → Secrets and variables → Actions → Variables → New repository variable**

| Variable Name | Value |
|---|---|
| `GH_ORG` | `<YOUR-GITHUB-USERNAME>` |
| `TF_STATE_BUCKET` | `<YOUR-TF-STATE-BUCKET>` |

### 4c. Create the `dev` GitHub Environment

GitHub Environments add a manual approval gate before the Terraform apply job runs, so you review the plan before anything is created.

**Settings → Environments → New environment**

1. Name: `dev`
2. Click **Configure environment**
3. Under **Deployment protection rules**, enable **Required reviewers**
4. Add yourself as a reviewer
5. Click **Save protection rules**

### 4d. Add Environment-Scoped Secrets

Still on the `dev` environment page → **Environment secrets → Add secret**

| Secret Name | Value |
|---|---|
| `DEV_DB_PASSWORD` | `<YOUR-DB-PASSWORD>` |
| `DEV_JWT_SECRET` | `<YOUR-JWT-SECRET>` |

> These secrets are only accessible to pipeline jobs targeting the `dev` environment — they cannot leak into other workflows.

---

## Step 5: Trigger Terraform — Approve and Start

### 5a. Trigger the Pipeline

The infra pipeline triggers on push to `main`. The commit you made in Step 3 already triggered it — go check:

`https://github.com/<YOUR-GITHUB-USERNAME>/infra → Actions`

If it didn't trigger (GitHub sometimes delays), make a trivial commit:

```bash
cd infra
echo "" >> README.md
git add README.md
git commit -m "chore: trigger terraform pipeline"
git push origin main
```

### 5b. Review and Approve

1. Click the running **Terraform Infrastructure** workflow
2. Click the **Terraform Apply** job — it shows **Waiting for approval**
3. Click **Review deployments**
4. Select the `dev` environment checkbox
5. Click **Approve and deploy**

### 5c. Confirm It's Running

After approval, the apply job starts. You should see:
```
Terraform will perform the following actions:
  # module.vpc.aws_vpc.this will be created
  # module.eks.aws_eks_cluster.this will be created
  ...
Plan: 45 to add, 0 to change, 0 to destroy.
```

The job will now run for **20-30 minutes**. EKS cluster creation is the slow step.

---

# ⏱️ Terraform Is Running

> The apply job is running. You have 20-30 minutes.
> **Go to Phase 2 now** — everything in Phase 2 is independent of Terraform and can be done in parallel.
>
> Keep the Actions tab open in a browser tab. You need to check back when it's done.

---

# Phase 2 — While Terraform Runs

---

## Step 6: Create GitHub PAT — GITOPS_TOKEN

The CI pipelines (frontend and backend) need a token to commit image tags to the gitops repository after each build. This token is stored as `GITOPS_TOKEN` in both repos.

### Option A: Fine-Grained Token (Recommended — Least Privilege)

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Click **Generate new token**
3. Set:
   - **Token name:** `zenpharma-gitops-bot`
   - **Expiration:** 90 days
   - **Resource owner:** your GitHub account
   - **Repository access:** Select only `gitops` (the one repo that CI writes to)
4. Under **Repository permissions**, set:
   - **Contents:** Read and write
   - (Metadata is auto-included — read-only, no action needed)
   - All other permissions: No access
5. Click **Generate token** and copy it immediately

### Option B: Classic Token (Simpler)

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token (classic)**
3. Set:
   - **Note:** `zenpharma-gitops-bot`
   - **Expiration:** 90 days
   - **Scopes:** Check `repo` only (full control of private repositories)
4. Click **Generate token** and copy it immediately

> The fine-grained token is more secure — it limits the token to a single repo and a single permission. The classic `repo` scope gives full access to all your repositories.

Save as `<YOUR-GITOPS-TOKEN>`.

---

## Step 7: Set Up SonarCloud

SonarCloud provides static code analysis (SAST). Both the frontend and backend pipelines require it.

### 7a. Create a SonarCloud Account

1. Go to [sonarcloud.io](https://sonarcloud.io)
2. Click **Log in with GitHub** and authorise

### 7b. Create a SonarCloud Organisation

1. Click **+** (top right) → **Create new organisation**
2. Choose **Import from GitHub**
3. Select your GitHub account
4. Install the SonarCloud GitHub App when prompted
5. Select `frontend` and `backend` repos to import
6. Choose the **Free plan**

Your organisation key is typically your GitHub username. Save as `<YOUR-SONAR-ORG>`.

### 7c. Create the Frontend Project

1. Click **+** → **Analyze new project** → select your `frontend` repository
2. Click **Set Up** → choose **With GitHub Actions**
3. **IMPORTANT:** Go to **Administration → Analysis Method → Disable Automatic Analysis**
   > If Automatic Analysis is ON, SonarCloud will conflict with the CI pipeline and both will fail.
4. Note the `projectKey` shown (typically `<YOUR-SONAR-ORG>_frontend`)

Save as `<YOUR-SONAR-PROJECT-FRONTEND>`.

### 7d. Create the Backend Project

Repeat for `backend`:
1. Click **+** → **Analyze new project** → select `backend`
2. **With GitHub Actions** → **Disable Automatic Analysis**
3. Note the `projectKey` (typically `<YOUR-SONAR-ORG>_backend`)

Save as `<YOUR-SONAR-PROJECT-BACKEND>`.

### 7e. Generate a SonarCloud Token

1. Click your avatar (top right) → **My Account → Security**
2. Under **Generate Tokens**, type `zenpharma-ci` → **Generate**
3. Copy the token

Save as `<YOUR-SONAR-TOKEN>`.

---

## Step 8: Update GitOps Repository Code

The gitops repo has the original AWS account ID (`873135413040`) and GitHub org (`zenpharma`) hardcoded in image URLs and ArgoCD application files. Replace them with yours.

```bash
cd gitops

export OLD_ACCOUNT=873135413040
export NEW_ACCOUNT=<YOUR-AWS-ACCOUNT-ID>
export OLD_ORG=zenpharma
export NEW_ORG=<YOUR-GITHUB-USERNAME>

# Replace AWS account ID in all Helm values files (ECR image URLs)
find envs/ -name "values-*.yaml" | xargs sed -i '' \
  "s/${OLD_ACCOUNT}/${NEW_ACCOUNT}/g"

# Replace GitHub org in all ArgoCD Application files (gitops repo URL)
find argocd/apps/ -name "*.yaml" | xargs sed -i '' \
  "s|https://github.com/${OLD_ORG}/gitops.git|https://github.com/${NEW_ORG}/gitops.git|g"

# Verify
echo "--- ECR URLs (should show your account ID) ---"
grep -r "repository:" envs/dev/ | head -3

echo "--- ArgoCD repoURLs (should show your GitHub username) ---"
grep -r "repoURL" argocd/apps/dev/ | head -3

# Commit and push
git add .
git commit -m "chore: update AWS account ID and GitHub org to my values"
git push origin main
```

---

## Step 9: Configure Frontend Repository

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/frontend → Settings`

### 9a. Add Repository Secrets

**Settings → Secrets and variables → Actions → Secrets**

| Secret Name | Value | Purpose |
|---|---|---|
| `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` | Constructs the OIDC IAM role ARN for ECR push |
| `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` | SonarCloud authentication |
| `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` | Commits image tags to gitops repo |

### 9b. Add Repository Variables

**Settings → Secrets and variables → Actions → Variables**

| Variable Name | Value | Purpose |
|---|---|---|
| `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` | Target repo for image tag commits |
| `SONAR_ORG` | `<YOUR-SONAR-ORG>` | SonarCloud organisation key |
| `SONAR_PROJECT_KEY_FRONTEND` | `<YOUR-SONAR-PROJECT-FRONTEND>` | SonarCloud project key |

### 9c. Create the `dev` GitHub Environment

**Settings → Environments → New environment**

1. Name: `dev`
2. No required reviewers (frontend deploy to dev is automatic)
3. Click **Save**

---

## Step 10: Configure Backend Repository

Go to: `https://github.com/<YOUR-GITHUB-USERNAME>/backend → Settings`

### 10a. Add Repository Secrets

**Settings → Secrets and variables → Actions → Secrets**

| Secret Name | Value | Purpose |
|---|---|---|
| `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` | Constructs the OIDC IAM role ARN for ECR push |
| `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` | SonarCloud authentication |
| `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` | Commits image tags to gitops repo |

### 10b. Add Repository Variables

**Settings → Secrets and variables → Actions → Variables**

| Variable Name | Value | Purpose |
|---|---|---|
| `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` | Target repo for image tag commits |
| `SONAR_ORG` | `<YOUR-SONAR-ORG>` | SonarCloud organisation key |
| `SONAR_PROJECT_KEY_BACKEND` | `<YOUR-SONAR-PROJECT-BACKEND>` | SonarCloud project key |

### 10c. Create the `dev` GitHub Environment

**Settings → Environments → New environment**

1. Name: `dev`
2. No required reviewers for auto-deploy to dev
3. Click **Save**

---

## Step 11: Create Branches and Enable Branch Protection

### 11a. Create `develop` Branch in Frontend and Backend

The CI pipelines trigger on pushes to `develop`. Create it from `main`:

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

### 11b. Enable Branch Protection on `main`

Do this for **each of the four repositories**:

1. Go to **Settings → Branches → Add branch protection rule**
2. Branch name pattern: `main`
3. Enable:
   - ✅ **Require a pull request before merging**
   - ✅ **Require approvals** (1 reviewer)
   - ✅ **Require status checks to pass before merging**
4. Click **Create**

### 11c. Install Phase 3 Tools (While You Wait)

If Terraform is still running, install the tools needed for Phase 3:

```bash
brew install kubectl helm python3 kubectx

# Verify
kubectl version --client   # >= 1.28
helm version               # >= 3.x
python3 --version          # >= 3.9
```

---

# Phase 3 — After Terraform Completes

> Check the Actions tab. When you see:
> ```
> Apply complete! Resources: 45 added, 0 changed, 0 destroyed.
> ```
> Phase 3 begins.

---

## Step 12: Verify Terraform Outputs

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

# 9 ECR repositories created
aws ecr describe-repositories \
  --query 'repositories[*].repositoryName' \
  --output table
# Expected: api-gateway, auth-service, drug-catalog-service, inventory-service,
#           manufacturing-service, notification-service, pharma-ui, qc-service, supplier-service
```

---

## Step 13: Bootstrap the Kubernetes Cluster

Terraform created the AWS infrastructure but did not install anything inside Kubernetes. Run three scripts in order.

### 13a. Connect kubectl to the Cluster

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pharma-dev-cluster \
  --alias dev

kubectl get nodes
# Expected: 4 nodes in Ready state (wait 2-3 minutes if not ready yet)
```

### 13b. Script 01 — Install AWS Load Balancer Controller

```bash
cd infra
python3 scripts/01_install_prerequisites.py
```

Prompts:
- **CLUSTER_NAME** → `pharma-dev-cluster` (Enter for default)
- **AWS_REGION** → `us-east-1` (Enter for default)
- **ALB_CONTROLLER_ROLE_ARN** → get from AWS:
  ```bash
  aws iam list-roles --query "Roles[?contains(RoleName,'alb-controller')].Arn" --output text
  ```

What it does: installs the AWS Load Balancer Controller into `kube-system` via Helm.

```bash
# Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
# Expected: READY 2/2
```

### 13c. Script 02 — Bootstrap ArgoCD

```bash
python3 scripts/02_bootstrap_argocd.py
```

Prompts:
- **GITOPS_REPO_URL** → `https://github.com/<YOUR-GITHUB-USERNAME>/gitops.git`
- **GITHUB_TOKEN** → `<YOUR-GITOPS-TOKEN>`

What it does: installs ArgoCD, registers your gitops repo, creates the `pharma` AppProject.

```bash
# Verify all ArgoCD pods are Running
kubectl get pods -n argocd

# Get the initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
# Save this password

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080 — admin / <password above>
```

### 13d. Script 03 — Setup External Secrets Operator

```bash
python3 scripts/03_setup_external_secrets.py
```

What it does: installs ESO, creates a `ClusterSecretStore` pointing at AWS Secrets Manager, creates `ExternalSecret` resources in the `dev` namespace.

```bash
# Verify ESO pods
kubectl get pods -n external-secrets

# Verify secrets synced from Secrets Manager
kubectl get externalsecret -n dev
# STATUS column should show: SecretSynced
```

---

## Step 14: Initialise the Database

RDS was created by Terraform but the schemas are empty.

### 14a. Get the RDS Endpoint

```bash
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-dev')].Endpoint.Address" \
  --output text
# Example: pharma-dev-postgres.abc123.us-east-1.rds.amazonaws.com
```

### 14b. Run the Schema Init Script

```bash
cd infra
./scripts/init-database.sh
```

Prompts:
- **RDS endpoint** → hostname from above
- **Database password** → `<YOUR-DB-PASSWORD>`

The script creates a temporary pod, connects to RDS, and creates all 8 schemas (`auth`, `drug_catalog`, `inventory`, `manufacturing`, `quality_control`, `supplier`, `distribution`, `reporting`).

Expected output:
```
[OK]  Schema SQL executed.
      Verifying schemas...
  Schema Name         | Owner
 auth                 | pharmaadmin
 drug_catalog         | pharmaadmin
 inventory            | pharmaadmin
 ...
[OK]  Database initialization complete.
```

---

## Step 15: Trigger the First Application Builds

Push to `develop` to trigger the CI pipelines. They build Docker images, push to ECR, and commit image tags to the gitops repo.

### 15a. Frontend

```bash
cd frontend
git checkout develop
echo "# pharma-ui" >> README.md
git add README.md
git commit -m "chore: trigger initial CI build"
git push origin develop
```

Go to Actions and watch the **CI/CD — pharma-ui** workflow:
```
Lint → Unit Tests → SonarCloud → Build → Docker Build + Trivy Scan → Push to ECR → Deploy to DEV
```

The Deploy step commits `ci(dev): update pharma-ui → sha-abc1234` to the gitops repo.

### 15b. Backend

```bash
cd ../backend
git checkout develop
echo "# ZenPharma Backend" >> README.md
git add README.md
git commit -m "chore: trigger initial CI builds"
git push origin develop
```

This triggers all 8 backend service pipelines. To trigger a specific service:
```bash
# Example: trigger api-gateway only
touch api-gateway/src/main/resources/application.yml
git add api-gateway/
git commit -m "chore: trigger api-gateway build"
git push origin develop
```

### 15c. Watch ArgoCD Deploy

Once CI commits the new image tag to gitops, ArgoCD detects it within ~3 minutes and deploys.

```bash
# Watch sync status
kubectl get applications -n argocd -w
# STATUS: OutOfSync → Synced
# HEALTH: Missing → Healthy
```

---

## Step 16: Verify Everything is Running

### All Pods Running

```bash
kubectl get pods -n dev
```

Expected — all 9 pods `Running` with `1/1` Ready:
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

### Get the Application URL

```bash
kubectl get ingress -n dev
# Copy the ADDRESS column — this is the ALB DNS name (may take 2-5 min to provision)

ALB_URL=$(kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
curl -s -o /dev/null -w "%{http_code}" http://$ALB_URL/
# Expected: 200
```

Open `http://$ALB_URL` in your browser.
**Login:** `admin` / `changeme`

### Final Checklist

```
✅ kubectl get nodes                          → 4 nodes Ready
✅ kubectl get pods -n dev                   → 9 pods Running 1/1
✅ kubectl get applications -n argocd        → all Synced + Healthy
✅ kubectl get externalsecret -n dev         → all SecretSynced
✅ Browser: http://<ALB-URL>                 → login page loads
✅ SonarCloud → sonarcloud.io                → analysis reports visible
✅ ECR → aws ecr describe-repositories      → 9 repos with recent images
```

---

## Step 17: Troubleshooting

### Terraform Apply Fails — `Error creating S3 bucket`

The bucket name is taken (S3 names are globally unique). Choose a different name:
```bash
export TF_STATE_BUCKET=zen-pharma-tf-<yourname>-2026
sed -i '' "s/zen-pharma-terraform-state-ravdy/$TF_STATE_BUCKET/g" infra/envs/dev/backend.tf
git add infra/envs/dev/backend.tf && git commit -m "fix: use unique bucket name" && git push
```

### Terraform Apply Fails — `Provider produced inconsistent final plan`

Transient AWS API issue. In GitHub Actions → click **Re-run failed jobs**.

### S3 State Lock Stuck

If a pipeline was cancelled mid-run, the lock file remains:
```bash
aws s3 rm s3://<YOUR-TF-STATE-BUCKET>/envs/dev/terraform.tfstate.tflock
```

### Pod in `ImagePullBackOff`

ECR URL in values file doesn't match your account ID:
```bash
kubectl describe pod <pod-name> -n dev | grep "Events:" -A10

# Fix: verify you replaced 873135413040 in gitops/envs/dev/values-*.yaml
grep "repository:" gitops/envs/dev/values-pharma-ui.yaml
# Should show your account ID
```

### Pod in `CrashLoopBackOff`

Usually a missing secret:
```bash
kubectl logs <pod-name> -n dev --previous

# Check if ExternalSecrets are synced
kubectl get externalsecret -n dev
kubectl describe externalsecret <name> -n dev | grep -A10 "Status"
```

### ExternalSecret shows `SecretSyncedError`

ESO cannot read from Secrets Manager:
```bash
# Verify IAM role exists
aws iam get-role --role-name pharma-dev-eks-role

# Verify secrets exist in Secrets Manager
aws secretsmanager list-secrets --query 'SecretList[*].Name' --output text
# Should show: pharma-dev-db-secret, pharma-dev-jwt-secret
```

### SonarCloud Analysis Fails — `Automatic Analysis not disabled`

SonarCloud → your project → **Administration → Analysis Method → switch off Automatic Analysis**.
Then re-run the CI pipeline.

### ArgoCD Shows `OutOfSync` After Image Push

Verify the gitops commit happened:
```bash
cd gitops && git log --oneline envs/dev/values-pharma-ui.yaml
# Should show a recent "ci(dev): update pharma-ui → sha-xxxxx" commit
```

If the commit is there but ArgoCD is still out of sync:
```bash
kubectl patch application pharma-ui-dev -n argocd \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{}}}'
```

---

## Complete Secrets and Variables Reference

Use this as a final checklist before triggering Terraform.

### `<YOUR-GITHUB-USERNAME>/infra`

| Type | Name | Value |
|---|---|---|
| Secret | `AWS_ACCESS_KEY_ID` | `terraform-user` access key (from Step 1a) |
| Secret | `AWS_SECRET_ACCESS_KEY` | `terraform-user` secret key (from Step 1a) |
| Variable | `GH_ORG` | `<YOUR-GITHUB-USERNAME>` |
| Variable | `TF_STATE_BUCKET` | `<YOUR-TF-STATE-BUCKET>` |
| Env Secret (`dev`) | `DEV_DB_PASSWORD` | `<YOUR-DB-PASSWORD>` |
| Env Secret (`dev`) | `DEV_JWT_SECRET` | `<YOUR-JWT-SECRET>` |

### `<YOUR-GITHUB-USERNAME>/frontend`

| Type | Name | Value |
|---|---|---|
| Secret | `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` |
| Secret | `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` |
| Secret | `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` |
| Variable | `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` |
| Variable | `SONAR_ORG` | `<YOUR-SONAR-ORG>` |
| Variable | `SONAR_PROJECT_KEY_FRONTEND` | `<YOUR-SONAR-PROJECT-FRONTEND>` |

### `<YOUR-GITHUB-USERNAME>/backend`

| Type | Name | Value |
|---|---|---|
| Secret | `AWS_ACCOUNT_ID` | `<YOUR-AWS-ACCOUNT-ID>` |
| Secret | `SONAR_TOKEN` | `<YOUR-SONAR-TOKEN>` |
| Secret | `GITOPS_TOKEN` | `<YOUR-GITOPS-TOKEN>` |
| Variable | `GITOPS_REPO` | `<YOUR-GITHUB-USERNAME>/gitops` |
| Variable | `SONAR_ORG` | `<YOUR-SONAR-ORG>` |
| Variable | `SONAR_PROJECT_KEY_BACKEND` | `<YOUR-SONAR-PROJECT-BACKEND>` |

### `<YOUR-GITHUB-USERNAME>/gitops` (code changes, no secrets)

| File | Change |
|---|---|
| `envs/dev/values-*.yaml` (all 9) | Replace `873135413040` with `<YOUR-AWS-ACCOUNT-ID>` |
| `argocd/apps/dev/*.yaml` (all apps) | Replace `zenpharma/gitops.git` with `<YOUR-GITHUB-USERNAME>/gitops.git` |
