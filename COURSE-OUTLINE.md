# ZenPharma DevOps Course — Complete Outline

> End-to-end DevOps pipeline: from raw code to production-grade Kubernetes
> deployments with Terraform, GitHub Actions, ArgoCD, and GitOps.

---

## Repositories

| Repo | GitHub URL | Purpose |
|------|-----------|---------|
| infra | `zenpharma/infra` | Terraform modules + Python bootstrap scripts |
| frontend | `zenpharma/frontend` | Pharma-UI (React + Nginx) |
| backend | `zenpharma/backend` | 8 microservices (Java Spring Boot + Node.js) |
| gitops | `zenpharma/gitops` | Helm charts, ArgoCD apps, env-specific values |

---

## Module 0 — Prerequisites

### 0.1 Accounts Required
- AWS account with admin access (or an IAM user with sufficient permissions)
- GitHub account (free tier is fine)
- A terminal (macOS Terminal / iTerm2 / Windows WSL2)

### 0.2 Tools to Install on Your Local Machine

| Tool | Purpose | Install |
|------|---------|---------|
| **AWS CLI v2** | Interact with AWS from terminal | `brew install awscli` |
| **Terraform** (>= 1.11) | Infrastructure as Code | `brew install terraform` |
| **kubectl** | Kubernetes CLI | `brew install kubectl` |
| **Helm** (v3) | Kubernetes package manager | `brew install helm` |
| **Git** | Version control | `brew install git` |
| **Python 3** | Run bootstrap scripts | `brew install python3` |
| **Node.js 20** | Build frontend locally (optional) | `brew install node@20` |
| **Docker Desktop** | Build and test containers locally | https://docs.docker.com/desktop/ |
| **VS Code** | Code editor | https://code.visualstudio.com/ |
| **GitHub CLI (`gh`)** | Interact with GitHub from terminal | `brew install gh` |
| **yq** | YAML processor (used in CI workflows) | `brew install yq` |

### 0.3 VS Code Extensions (Recommended)
- HashiCorp Terraform
- Kubernetes
- YAML
- Docker
- GitLens

### 0.4 Verify Installations
```bash
aws --version
terraform --version
kubectl version --client
helm version
git --version
python3 --version
docker --version
gh --version
```

---

## Module 1 — AWS & GitHub Foundation

### 1.1 Create and Configure AWS Credentials
- Create an IAM user (or use IAM Identity Center)
- Generate Access Key and Secret Key
- Run `aws configure` to set up the default profile
- Verify with `aws sts get-caller-identity`

> **Tag: `module-1.1-aws-credentials`** (no code change — document/verify only)

### 1.2 Create GitHub Organization and Repositories
- Create GitHub organization `zenpharma`
- Create 4 repos: `infra`, `frontend`, `backend`, `gitops`
- Add application source code to `frontend` and `backend` (no Dockerfiles, no workflows yet)

> **Tag all repos: `module-1.2-initial-code`**

### 1.3 Configure Passwordless GitHub Authentication (SSH)
- Generate SSH key: `ssh-keygen -t ed25519`
- Add public key to GitHub account (Settings → SSH Keys)
- Test: `ssh -T git@github.com`
- Update repo remotes to use SSH: `git remote set-url origin git@github.com:zenpharma/infra.git`

> **Tag: `module-1.3-ssh-auth`** (no code change — config only)

### 1.4 Create S3 Bucket for Terraform State
- Create S3 bucket: `zen-pharma-terraform-state-<your-name>`
- Enable versioning on the bucket
- Explain why remote state matters (team collaboration, locking, disaster recovery)

> **Tag `infra` repo: `module-1.4-s3-state`** (no code change — AWS resource only)

### 1.5 Add VPC Terraform Module
- Create folder structure: `modules/vpc/`, `envs/dev/`, `envs/qa/`, `envs/prod/`
- Add `backend.tf` — configure S3 backend with state file path
- Add `providers.tf` — AWS provider with required version constraints
- Add `variables.tf` and `outputs.tf`
- Add `modules/vpc/` — VPC, subnets (public, private, database), IGW, NAT, route tables
- Add `envs/dev/main.tf` — call the VPC module with dev-specific CIDR values
- `envs/qa/` and `envs/prod/` remain empty at this point
- Run `terraform init` and `terraform plan`
- **Explain:** Terraform module structure, state file, provider versions

> **Tag `infra` repo: `module-1.5-vpc-module`**

### 1.6 Run Terraform Apply — VPC
- Review the plan output (24 resources to add)
- Run `terraform apply`
- Verify resources in AWS Console (VPC, subnets, route tables)
- Inspect the state file in S3

> **Tag `infra` repo: `module-1.6-vpc-applied`**

### 1.7 Add IAM Terraform Module
- Create `modules/iam/` with IRSA roles:
  - **External Secrets Operator role** — read secrets from AWS Secrets Manager
  - **ArgoCD role** — application controller service account
  - **AWS Load Balancer Controller role** — manage ALBs from Kubernetes
  - **GitHub Actions OIDC role** — passwordless CI authentication (no static AWS keys in GitHub!)
- Add `github-actions-oidc.tf` — OIDC provider + trust policy scoped to your repos/branches
- Update `envs/dev/main.tf` to call the IAM module
- Add `github_org` variable
- Run `terraform plan` and `terraform apply`
- **Explain:** IRSA (IAM Roles for Service Accounts), OIDC federation, least-privilege

> **Tag `infra` repo: `module-1.7-iam-module`**

### 1.8 Add Remaining Terraform Modules
- **EKS module** (`modules/eks/`):
  - EKS cluster, managed node group, OIDC provider for IRSA
  - Kubernetes provider config using `exec` auth plugin
  - **Explain:** managed node groups, cluster autoscaling, OIDC provider
- **ECR module** (`modules/ecr/`):
  - Create one repository per microservice (9 repos: api-gateway, auth-service, drug-catalog-service, inventory-service, manufacturing-service, notification-service, pharma-ui, supplier-service, qc-service)
  - **Explain:** ECR lifecycle policies, image tagging strategy
- **RDS module** (`modules/rds/`):
  - PostgreSQL instance in database subnets
  - Security group allowing access only from EKS nodes
  - **Explain:** subnet groups, security group chaining, sensitive variables
- **Secrets Manager module** (`modules/secrets-manager/`):
  - Store DB credentials, JWT secret
  - **Explain:** how External Secrets Operator will sync these into Kubernetes
- Update `envs/dev/main.tf` to wire all modules together
- Add `db_password` and `jwt_secret` sensitive variables
- Run `terraform plan` and `terraform apply`
- **Explain:** module dependencies (EKS depends on VPC, IAM depends on EKS OIDC, etc.)

> **Tag `infra` repo: `module-1.8-all-modules`**

### 1.9 Verify and Explore Infrastructure
- Verify in AWS Console: VPC, EKS, RDS, ECR, IAM roles
- Configure kubectl: `aws eks update-kubeconfig --name pharma-dev --region us-east-1`
- Run `kubectl get nodes` to verify cluster access
- **Explain:** how `aws eks get-token` works under the hood

> **Tag `infra` repo: `module-1.9-infra-verified`**

### 1.10 Destroy Infrastructure
- Run `terraform destroy` to clean up (save costs)
- Verify all resources are gone
- **Explain:** destroy order, state cleanup

> *End of Module 1 — Destroy everything. We'll recreate via GitHub Actions in Module 2.*

---

## Module 2 — GitHub Actions for Terraform

### 2.1 Introduction to GitHub Actions
- What are GitHub Actions? (events, workflows, jobs, steps, runners)
- Workflow YAML syntax walkthrough
- **Explain:** triggers (`push`, `pull_request`, `workflow_dispatch`), `on.paths` filtering

### 2.2 Create Terraform GitHub Actions Workflow
- Add `.github/workflows/terraform.yml` with 3 jobs:
  - **Plan job:** `terraform fmt -check`, `init`, `validate`, `plan`, upload plan artifact
  - **Apply job:** downloads plan artifact, runs `terraform apply` (only on push to `main`)
  - **Destroy job:** manual trigger with confirmation gate (`workflow_dispatch`)
- **Explain:** concurrency groups (prevent parallel state modifications), environment approval gates, artifact passing between jobs, S3 native locking (`use_lockfile = true`)

> **Tag `infra` repo: `module-2.2-terraform-workflow`**

### 2.3 Adding Secrets and Variables to GitHub
- The workflow references `secrets.AWS_ACCESS_KEY_ID` and `vars.GH_ORG` — configure these before the workflow can run
- Add repository secrets:
  - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (for Terraform — the infra repo uses static keys, not OIDC)
  - `DEV_DB_PASSWORD`, `DEV_JWT_SECRET`
- Add repository variables:
  - `GH_ORG` (your GitHub username/org)
  - `TF_STATE_BUCKET` (S3 bucket name)
- **Explain:** secrets vs. variables (secrets are encrypted and masked in logs, variables are plaintext), environment-scoped secrets

> **Tag `infra` repo: `module-2.3-github-secrets`** (no code change — GitHub config only)

### 2.4 Enable Branch Protection and Approval Process
- Protect `main` branch: require PR, require approvals, require status checks
- Create a GitHub Environment called `dev` with required reviewers
- Walk through the flow: PR → plan runs → review plan output → approve → merge → apply runs → approve in environment → apply executes
- **Explain:** environment protection rules, why apply needs a separate gate

> **Tag `infra` repo: `module-2.4-branch-protection`**

### 2.5 Run Terraform Through GitHub Actions
- Push code to trigger the workflow
- Review the plan in GitHub Actions
- Approve and apply
- Verify infrastructure is created
- **Explain:** reading plan output in CI, debugging failed runs

> **Tag `infra` repo: `module-2.5-infra-via-ci`**

### 2.6 Destroy Infrastructure via GitHub Actions
- **Step 1 — Pre-destroy cleanup (run locally):** Delete ArgoCD apps, ingresses, and Helm charts before triggering destroy — ALBs created by the ALB Controller will block VPC deletion if left behind
  ```bash
  kubectl delete applications --all -n argocd
  kubectl delete ingress --all -n dev
  helm uninstall external-secrets -n external-secrets
  helm uninstall argocd -n argocd
  helm uninstall aws-load-balancer-controller -n kube-system
  kubectl delete namespace argocd external-secrets dev qa prod --ignore-not-found
  # Wait 2-3 minutes for ALBs to fully delete before proceeding
  ```
- **Step 2 — Trigger destroy:** Use `workflow_dispatch` with `action: destroy` and `confirm_destroy: "destroy"`
- **Step 3 — Approve** in the `dev` environment gate
- **Step 4 — Verify** all resources are cleaned up in AWS Console
- **Explain:** destroy safety gates, why pre-cleanup is required (ALBs are not managed by Terraform), why we require confirmation

> *End of Module 2 — Infrastructure is destroyed. Recreate it before Module 3.*

---

## Module 3 — Kubernetes Cluster Bootstrap

### 3.1 Recreate Infrastructure
- Re-run Terraform apply (via GitHub Actions or locally)
- Configure kubectl: `aws eks update-kubeconfig --name pharma-dev --region us-east-1`
- Verify: `kubectl get nodes`

### 3.2 Understanding the Bootstrap Scripts
- Walk through the 6 Python scripts and what each does:
  1. `01_install_prerequisites.py` — installs AWS LB Controller, ArgoCD, External Secrets Operator
  2. `02_bootstrap_argocd.py` — configures ArgoCD with gitops repo access
  3. `03_setup_external_secrets.py` — creates SecretStore + ExternalSecret resources
  4. `04_run_pipeline.py` — triggers CI pipelines to build images
  5. `05_deploy_services.py` — creates ArgoCD Application resources
  6. `06_verify_deployment.py` — health checks all services

### 3.3 Install AWS Load Balancer Controller, ArgoCD, and External Secrets Operator
- Run `01_install_prerequisites.py`
- **Explain:** what each tool does:
  - **AWS LB Controller:** watches Ingress resources → creates ALBs in AWS
  - **ArgoCD:** watches gitops repo → syncs Kubernetes state to match
  - **External Secrets Operator:** watches ExternalSecret CRDs → syncs AWS Secrets Manager into K8s Secrets
- **Explain:** Helm charts, Helm values, IRSA annotation on service accounts
- Verify: `kubectl get pods -n kube-system`, `kubectl get pods -n argocd`, `kubectl get pods -n external-secrets`

> **Tag `infra` repo: `module-3.3-cluster-bootstrap`**

---

## Module 4 — Frontend: Docker & CI/CD Pipeline

### 4.1 Create Dockerfile for Pharma-UI
- Write multi-stage Dockerfile:
  - Stage 1: `node:20-alpine` — `npm ci`, `npm run build`
  - Stage 2: `nginx:1.25-alpine` — copy build output, add nginx config
- Add `nginx.conf` for SPA routing
- Build and test locally: `docker build -t pharma-ui .` and `docker run -p 80:80 pharma-ui`
- **Explain:** multi-stage builds, non-root users, image size optimization

> **Tag `frontend` repo: `module-4.1-dockerfile`**

### 4.2 Protect Main Branch and Create Develop Branch
- Enable branch protection on `main`: require PR, require status checks
- Create `develop` branch from `main`
- **Explain:** branching strategy — `develop` for active development, `main` for stable releases, `release/*` for release candidates

> **Tag `frontend` repo: `module-4.2-branching`**

### 4.3 Add GitOps Token to GitHub Secrets
- Create a GitHub Personal Access Token (PAT) with repo scope on the gitops repo
- Add as `GITOPS_TOKEN` secret in frontend repo
- Add `AWS_ACCOUNT_ID` secret
- Add `GITOPS_REPO` variable (e.g., `zenpharma/gitops`)
- **Explain:** why the CI pipeline needs write access to a separate repo (GitOps pattern)

### 4.4 Create GitHub Workflows for Pharma-UI
- **PR workflow** (runs on pull_request to `main`/`develop`):
  - Lint (ESLint), unit tests (Jest with coverage), security scans (SonarCloud, npm audit)
  - No Docker build on PRs — fast feedback loop
- **CI/CD workflow** (`ci-pharma-ui.yml`, runs on push to `develop`/`release/*`):
  - Lint → Test → Security → Build React app → Docker build → Trivy scan → Push to ECR → Update gitops (dev) → Open QA promotion PR
- **Reusable workflow** (`_reusable-update-gitops.yml`):
  - Shared logic for updating image tags in the gitops repo
- **Explain:**
  - Image tag strategy: `sha-<7-char-git-sha>` (immutable, traceable to exact commit)
  - ECR authentication via OIDC (no static AWS keys!) — references the IAM role created in Module 1.7
  - Trivy container vulnerability scanning
  - Why two workflows (PR vs. merge): PR = quality gate, merge = deploy

> **Tag `frontend` repo: `module-4.4-ci-workflows`**

### 4.5 Create Production Promotion Workflow
- **Promote workflow** (`promote-prod-pharma-ui.yml`, manual `workflow_dispatch`):
  - Reads current image tag from QA values file
  - Opens a PR to update prod values file in gitops repo
  - Includes pre-merge checklist (QA sign-off, change ticket, runbook)
- **Explain:** manual promotion as a safety gate, PROD ArgoCD requires manual sync after merge

> **Tag `frontend` repo: `module-4.5-promote-workflow`**

---

## Module 5 — GitOps Repository

### 5.1 Create GitOps Repository Structure
- Create the gitops repo with this folder structure:
  ```
  gitops/
  ├── argocd/
  │   ├── apps/dev/          # ArgoCD Application manifests per env
  │   ├── apps/qa/
  │   ├── apps/prod/
  │   ├── install/           # ArgoCD ingress + namespace
  │   └── projects/          # ArgoCD AppProject
  ├── db-init/               # Database schema initialization SQL
  ├── envs/
  │   ├── dev/               # Per-service values files (dev)
  │   ├── qa/                # Per-service values files (qa)
  │   └── prod/              # Per-service values files (prod)
  ├── helm-charts/           # Shared Helm chart (1 chart, many values files)
  │   ├── Chart.yaml
  │   ├── values.yaml
  │   └── templates/
  │       ├── deployment.yaml
  │       ├── service.yaml
  │       ├── ingress.yaml
  │       ├── configmap.yaml
  │       ├── hpa.yaml
  │       ├── serviceaccount.yaml
  │       └── _helpers.tpl
  └── k8s/
      └── namespaces.yaml    # Namespace definitions
  ```
- **Explain:** single-chart-multiple-values pattern (one generic Helm chart, each service gets its own `values-<service>.yaml`)

> **Tag `gitops` repo: `module-5.1-repo-structure`**

### 5.2 Add ArgoCD Configuration
- Add `argocd/projects/pharma-project.yaml` — ArgoCD AppProject scoping allowed sources and destinations
- Add `argocd/install/argocd-namespace.yaml` and `argocd-ingress.yaml`
- **Explain:** AppProject as a security boundary, source restrictions, destination restrictions

> **Tag `gitops` repo: `module-5.2-argocd-config`**

### 5.3 Create Kubernetes Namespace Definitions
- Add `k8s/namespaces.yaml` — create namespaces for dev, qa, prod
- **Explain:** namespace isolation, resource quotas (optional)

> **Tag `gitops` repo: `module-5.3-namespaces`**

### 5.4 Add Database Initialization Script
- Add `db-init/01-schemas.sql` — creates per-service schemas in PostgreSQL (auth, drug_catalog, inventory, manufacturing, quality_control, supplier, etc.)
- **Explain:** schema-per-service pattern, how to apply this to RDS (connect via kubectl port-forward or bastion)

> **Tag `gitops` repo: `module-5.4-db-init`**

### 5.5 Write Raw Kubernetes Manifests for Pharma-UI (Conceptual)
- Show what raw manifests look like: Deployment, Service, Ingress, ConfigMap
- **Explain:** the problem — copy-paste across 9 services and 3 environments = 27+ files to maintain

### 5.6 Convert to a Shared Helm Chart
- Create `helm-charts/` — the generic chart with templates
- Walk through each template:
  - `deployment.yaml` — image, probes, resources, security context
  - `service.yaml` — ClusterIP
  - `ingress.yaml` — AWS ALB via annotation (conditional on `ingress.enabled`)
  - `configmap.yaml` — environment variables
  - `hpa.yaml` — horizontal pod autoscaler (conditional on `autoscaling.enabled`)
  - `serviceaccount.yaml` — with IRSA annotation support
  - `_helpers.tpl` — naming conventions
- **Explain:** Helm templating, values override hierarchy, `fullnameOverride`

> **Tag `gitops` repo: `module-5.6-helm-chart`**

### 5.7 Add Pharma-UI Values File for Dev Environment
- Create `envs/dev/values-pharma-ui.yaml` with:
  - ECR image repository and tag
  - Ingress enabled with ALB annotations (`alb.ingress.kubernetes.io/scheme: internet-facing`)
  - Resource requests/limits
  - Liveness and readiness probes
  - ConfigMap: `API_BASE_URL`, `AUTH_BASE_URL`, `ENV`
  - Volume mounts for Nginx (tmp, cache, run — required for `readOnlyRootFilesystem: true`)
- **Explain:** how the values file overrides the generic chart, ALB ingress annotations

> **Tag `gitops` repo: `module-5.7-pharma-ui-values`**

### 5.8 Add ArgoCD Application Manifest for Pharma-UI (Dev)
- Create `argocd/apps/dev/pharma-ui-app.yaml`
- **Explain:** ArgoCD Application CRD — source (gitops repo + path + values file), destination (cluster + namespace), sync policy (auto vs. manual)

> **Tag `gitops` repo: `module-5.8-pharma-ui-argocd-app`**

---

## Module 6 — First Deployment (Pharma-UI on Dev)

### 6.1 Bootstrap ArgoCD
- Run `02_bootstrap_argocd.py`
- This configures ArgoCD with:
  - Access to the gitops repo (using deploy key or token)
  - ArgoCD project registration
- Access ArgoCD UI:
  - Get initial admin password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
  - Port forward: `kubectl port-forward svc/argocd-server -n argocd 8080:443`
  - Or access via ALB ingress (if configured)

> **Tag `infra` repo: `module-6.1-argocd-bootstrap`**

### 6.2 Setup External Secrets
- Run `03_setup_external_secrets.py`
- This creates:
  - `SecretStore` — points to AWS Secrets Manager using the ESO IRSA role
  - `ExternalSecret` — maps specific secrets (DB credentials, JWT secret) to Kubernetes Secrets
- Verify: `kubectl get secretstore`, `kubectl get externalsecret`
- **Explain:** how ExternalSecret controller polls Secrets Manager and creates/updates K8s Secrets

> **Tag `infra` repo: `module-6.2-external-secrets`**

### 6.3 Initialize Database Schemas
- Connect to RDS via kubectl port-forward or bastion
- Run `db-init/01-schemas.sql` to create per-service schemas
- Verify schemas are created
- **Explain:** schema-per-service isolation

### 6.4 Run Pharma-UI CI Pipeline
- Push code to `develop` branch in frontend repo
- Or run `04_run_pipeline.py` to trigger the pipeline via GitHub API
- Pipeline builds Docker image, pushes to ECR, updates gitops dev values
- Verify: image appears in ECR, gitops repo has updated image tag

### 6.5 Deploy Pharma-UI via ArgoCD
- Run `05_deploy_services.py` to create the ArgoCD Application for pharma-ui
- Or apply manually: `kubectl apply -f argocd/apps/dev/pharma-ui-app.yaml`
- ArgoCD detects the values file and syncs the deployment

> **Tag `gitops` repo: `module-6.5-pharma-ui-deployed`**

### 6.6 Verify Deployment
- Run `06_verify_deployment.py` for automated health checks
- Check in ArgoCD UI: application should be "Synced" and "Healthy"
- Get ALB URL: `kubectl get ingress -n dev`
- Access pharma-ui from browser via the ALB URL
- **Explain:** ArgoCD sync status, health checks, how ALB ingress works end-to-end

> **Tag all repos: `module-6.6-first-deployment-verified`**

---

## Module 7 — Backend Microservices

### 7.1 Overview of Backend Architecture
- Walk through the 8 microservices:

  | Service | Language | Framework | Port |
  |---------|----------|-----------|------|
  | api-gateway | Java | Spring Boot | 8080 |
  | auth-service | Java | Spring Boot | 8081 |
  | drug-catalog-service | Java | Spring Boot | 8082 |
  | inventory-service | Java | Spring Boot | 8083 |
  | supplier-service | Java | Spring Boot | 8084 |
  | manufacturing-service | Java | Spring Boot | 8085 |
  | qc-service | Java | Spring Boot | 8086 |
  | notification-service | Node.js | Express | 3000 |

- **Explain:** microservice architecture, API gateway pattern, service communication

### 7.2 Create Dockerfiles for Each Microservice
- **Java services** — multi-stage: `maven:3-eclipse-temurin-17` build → `eclipse-temurin:17-jre` runtime, non-root user
- **Node.js service** (notification-service) — multi-stage: `node:20-alpine` build → runtime, non-root user
- Build and test locally
- **Explain:** JRE vs. JDK in production, `java.security.egd` for faster startup, multi-stage for smaller images

> **Tag `backend` repo: `module-7.2-dockerfiles`**

### 7.3 Create Reusable GitHub Workflows for Backend
- **Reusable Java workflow** (`_java-build.yml`):
  - Maven verify + JaCoCo coverage (>= 80%) → SonarCloud → OWASP Dependency Check → Docker build → Trivy scan → ECR push → Cosign keyless signing
- **Reusable Node.js workflow** (`_node-build.yml`):
  - npm ci + ESLint → Jest + coverage (>= 80%) → SonarCloud → npm audit → Docker build → Trivy scan → ECR push → Cosign keyless signing
- **Explain:** reusable workflows (`workflow_call`), why one reusable workflow per language, Cosign keyless image signing (supply chain security)

> **Tag `backend` repo: `module-7.3-reusable-workflows`**

### 7.4 Create Per-Service CI Workflows
- **Per-service CI workflows** (e.g., `ci-api-gateway.yml`):
  - Triggered on push to `develop`/`release/*` with path filter to that service's directory
  - Calls the reusable Java or Node.js workflow
  - Then calls the reusable gitops update workflow
  - Then opens a QA promotion PR
- **Per-service PR check workflows** (e.g., `ci-pr-api-gateway.yml`):
  - Triggered on PR to `main`/`develop`
  - Runs lint + test + security only (no Docker build, no ECR push)
- **Explain:** path filtering (only build what changed), monorepo CI strategy, PR checks vs. merge CI

> **Tag `backend` repo: `module-7.4-ci-workflows`**

### 7.5 Create Backend Promotion Workflow
- **`promote-prod.yml`** — manual dispatch with service selector dropdown
  - Reads QA image tag → opens PROD PR in gitops repo
  - Single workflow handles all backend services via input parameter
- **Explain:** consolidated promotion workflow vs. per-service (compare with frontend approach)

> **Tag `backend` repo: `module-7.5-promote-workflow`**

### 7.6 Add Backend Values Files to GitOps Repo
- Create `envs/dev/values-<service>.yaml` for each backend service
- Create corresponding `argocd/apps/dev/<service>-app.yaml` for each service
- **Explain:** per-service ingress paths (api-gateway routes), service discovery, environment variables via ConfigMap and Secrets

> **Tag `gitops` repo: `module-7.6-backend-values`**

### 7.7 Build and Deploy Backend Services
- Push backend code to `develop` branch (or trigger pipelines)
- All 8 services build, push images to ECR, update gitops dev values
- Run `05_deploy_services.py` to register all ArgoCD Applications
- Verify all services in ArgoCD UI — all should be "Synced" and "Healthy"

### 7.8 End-to-End Validation
- Access pharma-ui from browser via ALB URL
- Verify frontend can talk to backend services through the API gateway
- Check logs: `kubectl logs -n dev deployment/<service-name>`
- **Explain:** debugging microservice issues, ArgoCD app-of-apps pattern

> **Tag all repos: `module-7.8-full-stack-dev`**

---

## Module 8 — Environment Promotion (Dev → QA → Prod)

### 8.1 Understanding the Promotion Flow
- **Explain the full lifecycle:**
  ```
  Developer pushes to develop
       ↓
  CI builds image, pushes to ECR
       ↓
  CI updates envs/dev/values-*.yaml (direct commit)
       ↓
  ArgoCD syncs dev deployment (auto-sync)
       ↓
  CI opens PR: envs/qa/values-*.yaml (automated)
       ↓
  QA team reviews + approves PR → merge
       ↓
  ArgoCD syncs QA deployment (auto-sync)
       ↓
  Manual trigger: promote-prod workflow
       ↓
  Opens PR: envs/prod/values-*.yaml (automated)
       ↓
  Change board reviews + approves → merge
       ↓
  ArgoCD PROD: manual sync required → deploy
  ```

### 8.2 Add QA Environment Values and ArgoCD Apps
- Create `envs/qa/values-<service>.yaml` for all services
- Create `argocd/apps/qa/<service>-app.yaml` for all services
- **Explain:** environment-specific configuration differences (replicas, resources, feature flags)

> **Tag `gitops` repo: `module-8.2-qa-environment`**

### 8.3 Add Prod Environment Values and ArgoCD Apps
- Create `envs/prod/values-<service>.yaml` for all services
- Create `argocd/apps/prod/<service>-app.yaml` for all services
- **Explain:** prod differences — higher replicas, stricter resource limits, manual sync policy, HPA enabled

> **Tag `gitops` repo: `module-8.3-prod-environment`**

### 8.4 Promote Pharma-UI from Dev → QA
- CI has already opened a QA PR (from Module 6.4 pipeline run)
- Review the PR in gitops repo — verify only the image tag changed
- Merge the PR
- ArgoCD detects the change and syncs QA
- Verify in ArgoCD UI and browser

### 8.5 Promote Pharma-UI from QA → Prod
- Run `promote-prod-pharma-ui.yml` via workflow_dispatch in frontend repo
- Review the PROD PR — includes checklist (QA sign-off, change ticket, runbook)
- Merge the PR
- Manually sync in ArgoCD (PROD requires manual sync for safety)
- Verify in ArgoCD UI and browser

### 8.6 Promote Backend Services
- Repeat the promotion flow for backend services
- Use the consolidated `promote-prod.yml` workflow with the service selector dropdown
- **Explain:** staggered rollouts, rollback via reverting the gitops PR

### 8.7 QA/Prod Terraform Infrastructure (If Multi-Cluster)
- If using separate EKS clusters for QA and Prod:
  - Populate `envs/qa/main.tf` and `envs/prod/main.tf` with appropriate values
  - Run Terraform apply for each environment
  - Configure kubectl contexts for each cluster
- If using single cluster with namespace isolation:
  - **Explain:** the trade-offs (cost vs. isolation)

> **Tag `infra` repo: `module-8.7-multi-env-infra`** (if applicable)
>
> **Tag all repos: `module-8-promotion-complete`**

---

## Tagging Checklist

Use this as a reference while working through the course. Tag all relevant repos at each checkpoint.

```bash
# Tagging command format
git tag -a <tag-name> -m "<description>"
git push origin <tag-name>
```

| Module | Tag Name | Repos to Tag |
|--------|----------|-------------|
| 1.2 | `module-1.2-initial-code` | infra, frontend, backend |
| 1.5 | `module-1.5-vpc-module` | infra |
| 1.6 | `module-1.6-vpc-applied` | infra |
| 1.7 | `module-1.7-iam-module` | infra |
| 1.8 | `module-1.8-all-modules` | infra |
| 1.9 | `module-1.9-infra-verified` | infra |
| 2.2 | `module-2.2-terraform-workflow` | infra |
| 2.3 | `module-2.3-github-secrets` | infra |
| 2.4 | `module-2.4-branch-protection` | infra |
| 2.5 | `module-2.5-infra-via-ci` | infra |
| 3.3 | `module-3.3-cluster-bootstrap` | infra |
| 4.1 | `module-4.1-dockerfile` | frontend |
| 4.2 | `module-4.2-branching` | frontend |
| 4.4 | `module-4.4-ci-workflows` | frontend |
| 4.5 | `module-4.5-promote-workflow` | frontend |
| 5.1 | `module-5.1-repo-structure` | gitops |
| 5.2 | `module-5.2-argocd-config` | gitops |
| 5.3 | `module-5.3-namespaces` | gitops |
| 5.4 | `module-5.4-db-init` | gitops |
| 5.6 | `module-5.6-helm-chart` | gitops |
| 5.7 | `module-5.7-pharma-ui-values` | gitops |
| 5.8 | `module-5.8-pharma-ui-argocd-app` | gitops |
| 6.1 | `module-6.1-argocd-bootstrap` | infra |
| 6.2 | `module-6.2-external-secrets` | infra |
| 6.5 | `module-6.5-pharma-ui-deployed` | gitops |
| 6.6 | `module-6.6-first-deployment-verified` | all repos |
| 7.2 | `module-7.2-dockerfiles` | backend |
| 7.3 | `module-7.3-reusable-workflows` | backend |
| 7.4 | `module-7.4-ci-workflows` | backend |
| 7.5 | `module-7.5-promote-workflow` | backend |
| 7.6 | `module-7.6-backend-values` | gitops |
| 7.8 | `module-7.8-full-stack-dev` | all repos |
| 8.2 | `module-8.2-qa-environment` | gitops |
| 8.3 | `module-8.3-prod-environment` | gitops |
| 8.7 | `module-8.7-multi-env-infra` | infra (if applicable) |
| 8 (end) | `module-8-promotion-complete` | all repos |

---

## Course Summary

By the end of this course, learners will have built:
- **Infrastructure as Code** — Terraform modules for VPC, EKS, RDS, ECR, IAM, Secrets Manager
- **CI/CD Pipelines** — GitHub Actions with lint, test, security scan, Docker build, ECR push, Cosign signing
- **GitOps** — ArgoCD syncing Kubernetes state from a gitops repo
- **Secrets Management** — AWS Secrets Manager → External Secrets Operator → Kubernetes Secrets
- **Environment Promotion** — Dev (auto) → QA (PR approval) → Prod (manual dispatch + manual sync)
- **Security** — OIDC federation (no static keys), IRSA, SonarCloud (SAST + code quality), Trivy, Cosign, non-root containers
