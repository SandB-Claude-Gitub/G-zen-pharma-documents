# ZenPharma DevOps Platform — New Joinee Onboarding Runbook

> **Purpose:** This document helps a new engineer understand how the ZenPharma platform is built, how everything connects, and how to navigate the system on day one.
>
> **Assumed starting point:** The `zeninfra` environment is already up and running.  
> **Time to read:** ~45 minutes

---

## Table of Contents

1. [What is ZenPharma?](#1-what-is-zenpharma)
2. [Big Picture Architecture](#2-big-picture-architecture)
3. [The Four Repositories](#3-the-four-repositories)
4. [Technology Stack](#4-technology-stack)
5. [AWS Infrastructure — What Exists and Why](#5-aws-infrastructure--what-exists-and-why)
6. [How the Infrastructure Was Built](#6-how-the-infrastructure-was-built)
7. [How a Code Change Flows End to End](#7-how-a-code-change-flows-end-to-end)
8. [How Deployments Work — GitOps and ArgoCD](#8-how-deployments-work--gitops-and-argocd)
9. [How Environments Are Managed](#9-how-environments-are-managed)
10. [How Secrets Are Managed](#10-how-secrets-are-managed)
11. [Day One — Get Access and Connect](#11-day-one--get-access-and-connect)
12. [Key Files to Know](#12-key-files-to-know)
13. [Common Day-to-Day Commands](#13-common-day-to-day-commands)
14. [Glossary](#14-glossary)

---

## 1. What is ZenPharma?

ZenPharma is a pharmaceutical microservices platform. It consists of a React frontend and 8 Spring Boot backend services, deployed on AWS EKS (Kubernetes) across three environments: **dev**, **qa**, and **prod**.

### Services at a Glance

| Service | Technology | Role |
|---|---|---|
| `pharma-ui` | React + Nginx | Web frontend — the UI users see |
| `api-gateway` | Spring Boot | Single entry point for all `/api/*` traffic |
| `auth-service` | Spring Boot | Login, JWT token issuance and validation |
| `drug-catalog-service` | Spring Boot | Drug product catalogue |
| `inventory-service` | Spring Boot | Stock and inventory management |
| `manufacturing-service` | Spring Boot | Manufacturing orders and tracking |
| `notification-service` | Spring Boot | Email and SMS notifications |
| `supplier-service` | Spring Boot | Supplier management |
| `qc-service` | Spring Boot | Quality control and compliance |

> **Default login for dev:** `admin` / `changeme`

---

## 2. Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Developer Laptop                               │
│                                                                             │
│   git push (feature branch)                                                 │
└───────────────┬─────────────────────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GitHub (4 Repositories)                           │
│                                                                             │
│   zenpharma/infra     zenpharma/frontend    zenpharma/backend               │
│   (Terraform)         (React pharma-ui)     (8 Java services)               │
│                                                                             │
│   zenpharma/gitops    ← source of truth for what runs in Kubernetes         │
└───────┬───────────────────────┬──────────────────────────────────────────────┘
        │ GitHub Actions CI/CD  │ ArgoCD polls gitops
        ▼                       ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                            AWS — us-east-1                                    │
│                                                                               │
│  ┌─────────────────────────────────────────────────────┐                      │
│  │  VPC  10.0.0.0/16                                   │                      │
│  │                                                     │                      │
│  │  Public Subnets (10.0.1-2.0/24)                    │                      │
│  │    └─ ALB (internet-facing)                         │                      │
│  │         ├─ /         → pharma-ui pod                │                      │
│  │         └─ /api/*    → api-gateway pod              │                      │
│  │                                                     │                      │
│  │  Private Subnets (10.0.3-4.0/24)                   │                      │
│  │    └─ EKS Cluster  pharma-dev-cluster               │                      │
│  │         ├─ Namespace: dev    (9 services)           │                      │
│  │         ├─ Namespace: argocd (GitOps controller)    │                      │
│  │         ├─ Namespace: external-secrets (ESO)        │                      │
│  │         └─ Namespace: kube-system (ALB Controller)  │                      │
│  │                                                     │                      │
│  │  Database Subnets (10.0.5-6.0/24)                  │                      │
│  │    └─ RDS PostgreSQL  pharmadb                      │                      │
│  └─────────────────────────────────────────────────────┘                      │
│                                                                               │
│  ECR (9 image repos)   Secrets Manager   S3 (Terraform state)  IAM/OIDC      │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Traffic Flow (User to Service)

```
Browser
  │
  ▼
AWS ALB (internet-facing)
  │
  ├── path /         → pharma-ui (Nginx, port 80)  →  serves React app
  │
  └── path /api/*    → api-gateway (port 8080)
                           │
                           ├── /api/auth/*       → auth-service:8081
                           ├── /api/drugs/*      → drug-catalog-service:8082
                           ├── /api/inventory/*  → inventory-service:8083
                           ├── /api/manufacture/ → manufacturing-service:8084
                           ├── /api/suppliers/*  → supplier-service:8085
                           ├── /api/qc/*         → qc-service:8086
                           └── /api/notify/*     → notification-service:3000
```

---

## 3. The Four Repositories

The platform is split across four GitHub repositories. Each has a distinct responsibility.

```
GitHub Organisation: zenpharma
│
├── zenpharma/infra       ← Terraform — creates ALL AWS infrastructure
├── zenpharma/frontend    ← React app (pharma-ui) + its CI/CD pipeline
├── zenpharma/backend     ← All 8 Java microservices + their CI/CD pipelines
└── zenpharma/gitops      ← Kubernetes config — source of truth for what is deployed
```

### zenpharma/infra

Provisions all AWS infrastructure using Terraform. No manual AWS console clicks — everything is code.

```
infra/
├── envs/
│   ├── dev/
│   │   ├── backend.tf        ← S3 state config (key: envs/dev/terraform.tfstate)
│   │   ├── main.tf           ← calls all modules for dev
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── qa/                   ← same structure, different values
│   └── prod/
├── modules/
│   ├── vpc/                  ← VPC, subnets, route tables, NAT gateway
│   ├── eks/                  ← EKS cluster + managed node group
│   ├── rds/                  ← PostgreSQL RDS instance
│   ├── ecr/                  ← ECR repositories (one per service)
│   ├── iam/                  ← IAM roles for GitHub Actions OIDC + IRSA
│   └── secrets-manager/      ← stores DB credentials and JWT secret
└── scripts/
    ├── 01_install_prerequisites.py  ← installs ALB Controller on EKS
    ├── 02_bootstrap_argocd.py       ← installs and configures ArgoCD
    ├── 03_setup_external_secrets.py ← installs ESO, wires up Secrets Manager
    ├── init-database.sh             ← creates PostgreSQL schemas
    └── ...
```

### zenpharma/frontend

The React application and its complete CI/CD pipeline.

```
frontend/
├── src/                      ← React source code
├── public/
├── Dockerfile                ← multi-stage: node:22-alpine builder → nginx:1.25-alpine
├── nginx.conf                ← Nginx config (proxies /api to backend)
└── .github/workflows/
    ├── ci-pharma-ui.yml          ← main CI/CD (push to develop)
    ├── promote-qa-pharma-ui.yml  ← manual: promote dev image to QA
    └── promote-prod-pharma-ui.yml ← manual: promote QA image to prod
```

### zenpharma/backend

All 8 Java Spring Boot microservices in a monorepo, with a reusable CI pipeline.

```
backend/
├── api-gateway/              ← service directory (each has src/, pom.xml, Dockerfile)
├── auth-service/
├── drug-catalog-service/
├── inventory-service/
├── manufacturing-service/
├── notification-service/
├── qc-service/
├── supplier-service/
└── .github/workflows/
    ├── _java-build.yml       ← REUSABLE: Maven + SonarCloud + Docker + Trivy + ECR + Cosign
    ├── _java-pr-check.yml    ← REUSABLE: lightweight PR check (no Docker build)
    ├── ci-api-gateway.yml    ← calls _java-build.yml for api-gateway
    ├── ci-auth-service.yml   ← calls _java-build.yml for auth-service
    ├── ci-pr-api-gateway.yml ← calls _java-pr-check.yml on PRs
    ├── ...                   ← one ci-*.yml and one ci-pr-*.yml per service
    ├── promote-qa.yml        ← manual: promote any service from dev to QA
    └── promote-prod.yml      ← manual: promote any service from QA to prod
```

### zenpharma/gitops

This is not an application repo — it contains only Kubernetes configuration. ArgoCD watches this repo and deploys whatever is in it.

```
gitops/
├── helm-charts/              ← ONE shared Helm chart used by all 9 services
│   ├── Chart.yaml
│   ├── values.yaml           ← default values
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── serviceaccount.yaml
│       ├── configmap.yaml
│       ├── hpa.yaml
│       └── _helpers.tpl      ← reusable template functions
├── envs/
│   ├── dev/
│   │   ├── values-pharma-ui.yaml       ← image tag + config for pharma-ui in dev
│   │   ├── values-api-gateway.yaml
│   │   └── values-*.yaml               ← one per service
│   ├── qa/
│   │   └── values-*.yaml
│   └── prod/
│       └── values-*.yaml
├── argocd/
│   ├── apps/
│   │   ├── dev/
│   │   │   ├── pharma-ui-app.yaml      ← ArgoCD Application for pharma-ui in dev
│   │   │   └── *.yaml                  ← one per service
│   │   └── qa/
│   │       └── pharma-ui-app.yaml
│   ├── projects/
│   │   └── pharma-project.yaml         ← ArgoCD AppProject (governance policy)
│   └── install/
│       └── argocd-ingress.yaml         ← makes ArgoCD UI accessible via ALB
├── k8s/
│   └── namespaces.yaml                 ← dev, qa, prod namespace definitions
└── db-init/
    └── 01-schemas.sql                  ← creates one schema per service in PostgreSQL
```

---

## 4. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Cloud | AWS (us-east-1) | All infrastructure |
| Infrastructure as Code | Terraform ≥ 1.11 | Provision AWS resources |
| Container Orchestration | EKS (Kubernetes 1.33) | Run all services |
| Container Registry | Amazon ECR | Store Docker images |
| Database | RDS PostgreSQL | Persistent data (one DB, 8 schemas) |
| Secret Store | AWS Secrets Manager | DB credentials, JWT secret |
| CI/CD | GitHub Actions | Build, test, push images |
| GitOps | ArgoCD | Sync gitops repo → Kubernetes |
| Package Manager | Helm | Template Kubernetes manifests |
| Secret Sync | External Secrets Operator | Bridge Secrets Manager → K8s Secrets |
| Load Balancer | AWS Load Balancer Controller | Provisions ALB from Ingress resources |
| SAST | SonarCloud | Static code analysis |
| Image Scanning | Trivy | Container vulnerability scanning |
| Image Signing | Cosign (keyless) | Supply chain security |
| State Locking | S3 native locking | Terraform concurrent run protection |
| Frontend | React + Nginx | User interface |
| Backend | Java 17 + Spring Boot | Business logic |

---

## 5. AWS Infrastructure — What Exists and Why

When the environment is running, here is what AWS has provisioned (via Terraform):

### VPC

```
VPC: pharma-dev-vpc  (10.0.0.0/16)
├── Public Subnets:    10.0.1.0/24, 10.0.2.0/24   ← ALB lives here
├── Private Subnets:   10.0.3.0/24, 10.0.4.0/24   ← EKS nodes live here
└── Database Subnets:  10.0.5.0/24, 10.0.6.0/24   ← RDS lives here
```

**Why three subnet tiers?**  
Public subnets have a route to the internet gateway — only the ALB sits here. Private subnets have NAT gateway access for outbound calls (pulling images, calling AWS APIs) but cannot be reached from the internet. Database subnets have no internet route at all — only traffic from the private subnets can reach RDS.

### EKS Cluster

```
Cluster:      pharma-dev-cluster
Version:      Kubernetes 1.33
Nodes:        4 × t3.small  (min 1, max 5 — auto-scaled)
Node subnet:  private subnets
IRSA:         enabled (pods can assume IAM roles securely)
Add-ons:      VPC CNI, CoreDNS, kube-proxy, EKS Pod Identity Agent
```

### Kubernetes Namespaces

| Namespace | What runs here |
|---|---|
| `dev` | All 9 application services (dev environment) |
| `qa` | QA environment services |
| `prod` | Production services |
| `argocd` | ArgoCD controller — the GitOps engine |
| `external-secrets` | External Secrets Operator — syncs secrets from AWS |
| `kube-system` | AWS Load Balancer Controller, CoreDNS, kube-proxy |

### RDS PostgreSQL

```
Instance:      pharma-dev-postgres
Database:      pharmadb
Username:      pharmaadmin
Location:      database subnets (private — no public endpoint)
Access:        only EKS node security group allowed on port 5432
Schemas:       auth, drug_catalog, inventory, manufacturing,
               quality_control, supplier, distribution, reporting
```

Each microservice gets its own PostgreSQL schema within the single `pharmadb` database. This keeps them logically isolated while sharing one RDS instance for cost efficiency in dev.

### ECR Repositories

Nine repositories — one per service:
`api-gateway`, `auth-service`, `drug-catalog-service`, `inventory-service`, `manufacturing-service`, `notification-service`, `pharma-ui`, `supplier-service`, `qc-service`

Each repository keeps the **last 10 images** (lifecycle policy). Older images are deleted automatically.

### Secrets Manager

```
pharma-dev-db-secret  → { username, password, host, port, dbname }
pharma-dev-jwt-secret → { jwt_secret }
```

These are created by Terraform and populated with values from GitHub Secrets at apply time.

### IAM Roles

```
pharma-dev-github-actions-role  ← assumed by GitHub Actions via OIDC (no static keys)
                                   Permissions: ECR push, read Secrets Manager
pharma-dev-eks-role             ← assumed by EKS pods via IRSA
                                   Permissions: read Secrets Manager, specific ECR pulls
pharma-dev-alb-controller-role  ← assumed by AWS Load Balancer Controller pod
                                   Permissions: create/manage ALBs and target groups
```

### S3 — Terraform State

```
Bucket:  zen-pharma-terraform-state-ravdy
Keys:
  envs/dev/terraform.tfstate
  envs/dev/terraform.tfstate.tflock   ← S3 native lock (created during apply)
  envs/qa/terraform.tfstate
  envs/prod/terraform.tfstate
```

---

## 6. How the Infrastructure Was Built

The environment was not created by hand. It was built in two phases:

### Phase 1 — Terraform (GitHub Actions)

A merge to `main` in the infra repo triggers the terraform pipeline:

```
Push to main in zenpharma/infra
  │
  ▼
GitHub Actions: terraform.yml
  ├── Job: plan  → terraform init + plan (shows what will be created)
  │              → pauses: human reviews plan
  │
  └── Job: apply (after approval)
        → terraform apply
        → Creates: VPC, EKS, RDS, ECR, IAM, Secrets Manager
        → Duration: ~15 minutes
```

Everything Terraform creates is reproducible — if you delete the entire environment and run `terraform apply` again, you get the same infrastructure back.

### Phase 2 — Bootstrap Scripts (run manually once after terraform apply)

Terraform provisions AWS resources but does not install Kubernetes tools. After every new cluster creation, three Python scripts are run in order:

```bash
# From the infra/ directory:

# Script 01 — Connects kubectl to the cluster,
#             installs AWS Load Balancer Controller via Helm
python3 scripts/01_install_prerequisites.py

# Script 02 — Installs ArgoCD, registers the gitops repo,
#             creates the pharma AppProject
python3 scripts/02_bootstrap_argocd.py

# Script 03 — Installs External Secrets Operator,
#             creates ClusterSecretStore → Secrets Manager,
#             creates ExternalSecret resources in the dev namespace
python3 scripts/03_setup_external_secrets.py
```

### Phase 3 — Database Initialisation (run once)

```bash
# Creates the 8 PostgreSQL schemas inside pharmadb
./scripts/init-database.sh
# Prompts for RDS endpoint and DB password
# Spins up a temporary pod inside the cluster to reach the private RDS endpoint
# Runs db-init/01-schemas.sql
# Deletes the temporary pod
```

After these three phases, the cluster is ready to receive application deployments.

---

## 7. How a Code Change Flows End to End

Understanding this flow is the most important thing for a new joinee. This is how a developer's code change gets to the running application.

### Backend Service Example (auth-service)

```
1. Developer writes code on feature/fix-login branch
   │
   ▼
2. Opens PR to develop branch
   └── ci-pr-auth-service.yml triggers automatically
       ├── Maven verify (compile + unit tests + JaCoCo coverage ≥80%)
       └── SonarCloud SAST (no Docker build on PRs — fast feedback ~5 min)

3. PR approved + merged to develop
   │
   ▼
4. ci-auth-service.yml triggers (push to develop, path: auth-service/**)
   └── Calls _java-build.yml (reusable workflow):
       ├── Maven verify (full test suite)
       ├── SonarCloud analysis
       ├── AWS credentials via OIDC (no static keys)
       ├── Docker build → image tagged: sha-a3f9b21
       ├── Trivy scan (HIGH/CRITICAL CVEs reported)
       ├── Push to ECR: 873135413040.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-a3f9b21
       ├── Cosign sign (keyless — GitHub OIDC → Fulcio → Rekor)
       └── Update gitops: envs/dev/values-auth-service.yaml
              image.tag: sha-a3f9b21
              (commit: "ci(dev): update auth-service → sha-a3f9b21")
   │
   ▼
5. ArgoCD detects gitops commit (polls every ~3 minutes)
   └── Computes diff: running image sha-old vs new sha-a3f9b21
   └── Applies: rolling update → new pod starts → health checks pass → old pod removed
   │
   ▼
6. auth-service:sha-a3f9b21 is now running in dev namespace
```

### Frontend Example (pharma-ui)

Same flow, but the CI pipeline is `ci-pharma-ui.yml`:
```
lint → unit tests → SonarCloud → npm build → Docker build → Trivy → ECR push → gitops update
```

### Key Insight — The Pipeline Never Touches Kubernetes Directly

The pipeline does **not** run `kubectl apply`. It only:
1. Builds an image and pushes it to ECR
2. Commits one line to the gitops repo (the new image tag)

ArgoCD does the actual deployment. This separation means:
- Rollback = revert a gitops commit (not a pipeline run)
- Audit trail = git history of the gitops repo
- Drift detection = ArgoCD alerts if someone manually edits a pod

---

## 8. How Deployments Work — GitOps and ArgoCD

ArgoCD is the deployment engine. It runs inside the cluster and continuously reconciles what is in the gitops repo against what is running in Kubernetes.

### The Deployment Model

```
gitops repo (source of truth)
├── envs/dev/values-pharma-ui.yaml   image.tag: sha-a3f9b21
│
▼  (ArgoCD polls every 3 min)
│
ArgoCD Application: pharma-ui-dev
  source:  gitops/helm-charts + envs/dev/values-pharma-ui.yaml
  destination: namespace dev
  syncPolicy: automated (prune + selfHeal)
│
▼  (renders Helm chart with values, applies to cluster)
│
Kubernetes: Deployment/pharma-ui in namespace dev
  image: 873135413040.dkr.ecr.us-east-1.amazonaws.com/pharma-ui:sha-a3f9b21
```

### ArgoCD Application File

Every service has an Application manifest in the gitops repo:

```yaml
# argocd/apps/dev/pharma-ui-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pharma-ui-dev
  namespace: argocd
spec:
  project: pharma

  source:
    repoURL: https://github.com/zenpharma/gitops.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/dev/values-pharma-ui.yaml  ← environment-specific config

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true      # removes K8s resources deleted from gitops
      selfHeal: true   # reverts manual kubectl changes — git is always the truth
```

### Checking ArgoCD

**ArgoCD UI:** accessible via the ALB on HTTPS port 443.  
**Default password:** stored in the cluster:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Via CLI:**
```bash
kubectl get applications -n argocd
# Shows: NAME, SYNC STATUS, HEALTH STATUS
```

### Environment Promotion — Dev → QA → Prod

Promotion is **manual and deliberate**. It never happens automatically for QA or prod.

```
DEV (automatic — every push to develop deploys here)
  │
  │  Developer or QA Lead triggers:
  │  GitHub Actions → promote-qa-pharma-ui.yml → Run workflow
  │
  │  What it does:
  │  1. Reads image tag from envs/dev/values-pharma-ui.yaml
  │  2. Creates branch promote/qa/pharma-ui/sha-a3f9b21 in gitops
  │  3. Opens a PR to update envs/qa/values-pharma-ui.yaml
  │  4. QA team reviews and merges PR
  │  5. ArgoCD auto-syncs QA environment
  │
  ▼
QA
  │
  │  Same pattern: promote-prod-pharma-ui.yml
  │
  ▼
PROD
```

**The same image that passed testing in dev goes to QA and prod — it is never rebuilt.**

---

## 9. How Environments Are Managed

### Environment Differences

| | Dev | QA | Prod |
|---|---|---|---|
| AWS Account | shared (or separate) | separate | separate |
| EKS Cluster | pharma-dev-cluster | pharma-qa-cluster | pharma-prod-cluster |
| K8s Namespace | dev | qa | prod |
| Terraform State | envs/dev/terraform.tfstate | envs/qa/terraform.tfstate | envs/prod/terraform.tfstate |
| GitHub Environment | dev | qa | prod |
| Deployment | Automatic (CI pushes) | Manual (PR merge) | Manual (PR merge) |
| Node Size | t3.small | t3.medium | c5.large |
| ArgoCD Sync | Automated | Automated (after PR merge) | Manual trigger |

### GitHub Environments — Approval Gates

GitHub Environments add an approval step before sensitive jobs run. The `apply` and `destroy` jobs in the Terraform pipeline reference `environment: dev`. Before the job executes, the configured reviewers must click Approve in GitHub.

```
Settings → Environments → dev → Required reviewers: [your-team]
```

This prevents anyone from accidentally running `terraform destroy` in production.

### Branching Strategy

**Application repos (frontend + backend):**
```
feature/new-feature  ──┐
fix/bug-fix           ──┤  PR review required
                        ▼
                     develop  ← CI runs here → builds image → deploys to dev
                        │
                        │  PR review required
                        ▼
                       main   ← protected; triggers nothing on its own
                        │
                   release/2.1  ← cut for stable releases (CI also runs here)
```

**Infra repo:**
```
feature/add-karpenter  ──┐
                         │  PR → shows terraform plan as comment
                         ▼
                        main  ← protected; merge triggers terraform plan → approval → apply
```

There is no `develop` branch in the infra repo. Infrastructure changes go directly to `main` via PR because `terraform plan` in the PR itself serves as the review mechanism.

---

## 10. How Secrets Are Managed

Understanding the secrets architecture is critical. There are three layers:

### Layer 1 — Build Secrets (GitHub Repository/Environment Secrets)

Stored in GitHub. Used only during CI pipeline runs. Never available to running pods.

| Secret | Where | Used by |
|---|---|---|
| `AWS_ACCOUNT_ID` | Repo secret | All pipelines — construct OIDC role ARN |
| `SONAR_TOKEN` | Repo secret | SonarCloud analysis |
| `GITOPS_TOKEN` | Repo secret | Bot that commits image tag to gitops repo |
| `DEV_DB_PASSWORD` | `dev` Environment secret | Terraform apply only |
| `DEV_JWT_SECRET` | `dev` Environment secret | Terraform apply only |

### Layer 2 — Infrastructure Secrets (AWS Secrets Manager)

Created by Terraform during apply. Contain the actual runtime values.

```
pharma-dev-db-secret:
  username: pharmaadmin
  password: <from DEV_DB_PASSWORD GitHub secret>
  host: pharma-dev-postgres.xxx.us-east-1.rds.amazonaws.com
  port: 5432
  dbname: pharmadb

pharma-dev-jwt-secret:
  jwt_secret: <from DEV_JWT_SECRET GitHub secret>
```

### Layer 3 — Runtime Secrets (Kubernetes Secrets, created by ESO)

External Secrets Operator (running in `external-secrets` namespace) reads from Secrets Manager and creates native Kubernetes Secrets in the `dev` namespace. These are what pods actually consume.

```
ExternalSecret (K8s resource, defines the mapping)
      │
      ▼  ESO syncs every hour
      │
Kubernetes Secret: db-credentials  (in namespace: dev)
  SPRING_DATASOURCE_URL: jdbc:postgresql://pharma-dev-postgres.../pharmadb
  SPRING_DATASOURCE_USERNAME: pharmaadmin
  SPRING_DATASOURCE_PASSWORD: ***
      │
      ▼  envFrom: secretRef
      │
Running pod — DB password injected as env variable at startup
```

**Rule to remember:** Secrets never appear in git, never in Docker images, never in CI logs. They flow: GitHub (build only) → Secrets Manager (storage) → ESO → K8s Secret → Pod.

---

## 11. Day One — Get Access and Connect

### Tools to Install on Your Laptop

```bash
# AWS CLI
brew install awscli

# kubectl
brew install kubectl

# Helm
brew install helm

# kubectx + kubens (fast context/namespace switching)
brew install kubectx

# ArgoCD CLI (optional but useful)
brew install argocd

# Terraform
brew install terraform

# yq (YAML processor — used in pipelines)
brew install yq
```

### Access You Need to Request

| Access | Who to ask | Why |
|---|---|---|
| AWS IAM user / SSO access | DevOps/Cloud team | To run kubectl, view logs, check AWS resources |
| GitHub organisation invite | Team lead | Access to all 4 repos |
| SonarCloud access | DevOps team | View code quality reports |
| ArgoCD access | DevOps team | View/manage deployments |
| AWS Secrets Manager read | DevOps team | To read dev secrets if debugging |

### Connect kubectl to the Dev Cluster

```bash
# 1. Configure your AWS credentials
aws configure
# OR if using SSO:
aws sso login --profile zenpharma-dev

# 2. Pull the kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name pharma-dev-cluster \
  --alias dev

# 3. Verify
kubectl get nodes
# Should show 4 nodes in Ready state

kubectl get pods -n dev
# Should show all 9 services running
```

### Get the Application URL

```bash
kubectl get ingress -n dev
# The ADDRESS column shows the ALB DNS name
# Example: pharma-dev-xxx.us-east-1.elb.amazonaws.com
```

Open that URL in a browser — you should see the pharma-ui login page.  
Login: `admin` / `changeme`

### Access ArgoCD UI

```bash
# Get the ArgoCD URL
kubectl get ingress -n argocd

# Get the initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## 12. Key Files to Know

These are the files you will touch most often as you work on this project.

| File | Repo | What it controls |
|---|---|---|
| `envs/dev/main.tf` | infra | All dev infrastructure (EKS size, RDS, ECR repos) |
| `envs/dev/backend.tf` | infra | S3 state location for dev |
| `.github/workflows/terraform.yml` | infra | Terraform CI/CD pipeline |
| `envs/dev/values-<svc>.yaml` | gitops | **Image tag** and config for each service in dev |
| `envs/qa/values-<svc>.yaml` | gitops | Image tag and config for QA |
| `argocd/apps/dev/<svc>-app.yaml` | gitops | ArgoCD Application for each dev service |
| `helm-charts/values.yaml` | gitops | Default Helm values (probes, resources, security) |
| `helm-charts/templates/deployment.yaml` | gitops | Kubernetes Deployment template |
| `.github/workflows/_java-build.yml` | backend | Reusable CI pipeline for all Java services |
| `.github/workflows/ci-<svc>.yml` | backend | Triggers the build for a specific service |
| `.github/workflows/promote-qa.yml` | backend | Manual promotion: dev → QA |
| `.github/workflows/ci-pharma-ui.yml` | frontend | Full CI/CD for the React frontend |
| `db-init/01-schemas.sql` | gitops | PostgreSQL schema definitions |
| `scripts/01_install_prerequisites.py` | infra | Installs ALB Controller on cluster |
| `scripts/02_bootstrap_argocd.py` | infra | Installs and configures ArgoCD |
| `scripts/03_setup_external_secrets.py` | infra | Sets up ESO → Secrets Manager |

---

## 13. Common Day-to-Day Commands

### Checking Application Health

```bash
# Are all pods running?
kubectl get pods -n dev

# Check a specific service's logs
kubectl logs -l app.kubernetes.io/name=auth-service -n dev --tail=100

# Check pod resource usage
kubectl top pods -n dev

# Check all ArgoCD apps
kubectl get applications -n argocd

# Check if secrets are synced from Secrets Manager
kubectl get externalsecret -n dev
# STATUS column should show: SecretSynced
```

### Accessing the Application

```bash
# Get ALB URL for the application
kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'

# Test the API directly
curl http://<ALB-URL>/api/actuator/health
```

### Debugging a Pod

```bash
# Pod not starting? Check events
kubectl describe pod <pod-name> -n dev

# Crashed? Check previous container logs
kubectl logs <pod-name> -n dev --previous

# Test DB connectivity from inside the cluster
kubectl run debug --rm -it --image=postgres:15-alpine -n dev -- \
  psql -h <RDS-ENDPOINT> -U pharmaadmin -d pharmadb
```

### Terraform Operations

```bash
# Check what's in the current state
cd infra/envs/dev
terraform init
terraform show

# If the S3 state lock is stuck (from a cancelled pipeline)
aws s3 rm s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock
```

### Force-Refresh a Secret

```bash
# If a secret was updated in Secrets Manager, force ESO to re-sync immediately
kubectl annotate externalsecret <name> -n dev \
  force-sync=$(date +%s) --overwrite

# Then restart pods to pick up the new Kubernetes Secret
kubectl rollout restart deployment/<service-name> -n dev
```

### Rolling Back a Deployment

```bash
# Option 1 (preferred): Revert the gitops commit
cd gitops
git log --oneline envs/dev/values-auth-service.yaml  # find the bad commit
git revert <commit-sha>
git push
# ArgoCD auto-syncs and deploys the previous image

# Option 2: Emergency rollback via kubectl
kubectl rollout undo deployment/auth-service -n dev
# Then fix the gitops file to match, or ArgoCD will revert your rollback
```

---

## 14. Glossary

| Term | What it means in this project |
|---|---|
| **GitOps** | The gitops repo is the only source of truth for what runs in Kubernetes. You change what's deployed by changing a file in git, not by running kubectl. |
| **ArgoCD** | The GitOps engine. It watches the gitops repo and applies changes to the cluster automatically. |
| **OIDC** | How GitHub Actions authenticates to AWS without any stored access keys. GitHub issues a short-lived token; AWS trusts it. |
| **IRSA** | IAM Roles for Service Accounts. A pod can assume an AWS IAM role without any credentials stored in the pod. |
| **ESO** | External Secrets Operator. Reads from AWS Secrets Manager and creates Kubernetes Secrets. |
| **ECR** | Amazon Elastic Container Registry. Where Docker images are stored. One repo per service. |
| **Helm chart** | A package of Kubernetes YAML templates. We have one shared chart and one values file per service per environment. |
| **ArgoCD Application** | A Kubernetes resource that tells ArgoCD where to get config (gitops repo) and where to deploy it (cluster namespace). |
| **ArgoCD AppProject** | The governance policy. Defines which repos and namespaces Applications are allowed to use. |
| **Cosign** | Signs Docker images after push so you can verify they came from your pipeline and weren't tampered with. |
| **Trivy** | Scans Docker images for known CVEs before they are pushed to ECR. |
| **SonarCloud** | Analyses source code for bugs, security vulnerabilities, and code quality issues on every push. |
| **IRSA** | IAM Roles for Service Accounts — pods authenticate to AWS using their K8s service account identity. |
| **S3 native locking** | Terraform's state locking mechanism using an `.tflock` file in S3. Prevents two pipeline runs from modifying the same state simultaneously. |
| **sha-<7chars>** | Our image tag format. `sha-a3f9b21` is the first 7 characters of the git commit SHA. Immutable and traceable. |
| **selfHeal** | ArgoCD feature. If someone manually edits a K8s resource with kubectl, ArgoCD reverts it within 3 minutes. Git is always the truth. |
| **Prune** | ArgoCD feature. If a K8s resource is removed from the gitops repo, ArgoCD deletes it from the cluster. |

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                 ZenPharma Quick Reference                   │
├─────────────────────────────────────────────────────────────┤
│ Cluster        pharma-dev-cluster  (us-east-1)              │
│ State bucket   zen-pharma-terraform-state-ravdy             │
│ DB             pharmadb / pharmaadmin  (private RDS)        │
│ Namespaces     dev, qa, prod, argocd, external-secrets      │
│ Image format   sha-<7chars>  e.g. sha-a3f9b21              │
│ ECR base       873135413040.dkr.ecr.us-east-1.amazonaws.com│
├─────────────────────────────────────────────────────────────┤
│ Check pods       kubectl get pods -n dev                    │
│ Check ArgoCD     kubectl get applications -n argocd         │
│ Check secrets    kubectl get externalsecret -n dev          │
│ Get app URL      kubectl get ingress -n dev                 │
│ Get ArgoCD pwd   kubectl get secret argocd-initial-admin-   │
│                  secret -n argocd -o jsonpath=              │
│                  '{.data.password}' | base64 -d             │
│ Release TF lock  aws s3 rm s3://zen-pharma-terraform-       │
│                  state-ravdy/envs/dev/terraform.tfstate.    │
│                  tflock                                     │
├─────────────────────────────────────────────────────────────┤
│ Repos                                                       │
│   zenpharma/infra     → Terraform (AWS infra)              │
│   zenpharma/frontend  → pharma-ui (React)                  │
│   zenpharma/backend   → 8 Java microservices               │
│   zenpharma/gitops    → K8s config (source of truth)       │
└─────────────────────────────────────────────────────────────┘
```
