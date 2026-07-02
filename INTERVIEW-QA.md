# ZenPharma DevOps — Interview Q&A Guide

> Answers are grounded in the actual ZenPharma project:  
> 4 repositories (infra, frontend, backend, gitops) · 3 environments (dev/qa/prod) · 9 microservices · EKS on AWS

---

## Table of Contents

1. [Infrastructure & AWS](#1-infrastructure--aws)
2. [CI/CD Pipeline](#2-cicd-pipeline)
3. [Security](#3-security)
4. [Kubernetes & Helm](#4-kubernetes--helm)
5. [GitOps & ArgoCD](#5-gitops--argocd)
6. [Docker & Image Management](#6-docker--image-management)
7. [Environment & Secret Management](#7-environment--secret-management)
8. [Troubleshooting](#8-troubleshooting)
9. [Operational Topics](#9-operational-topics)
10. [Python & Automation](#10-python--automation)
11. [AI & Advanced Topics](#11-ai--advanced-topics)

---

## 1. Infrastructure & AWS

### Q1. Can you explain your environment? How many servers do you have, and how many microservices?

We have three environments: **dev**, **qa**, and **prod**, each provisioned independently through Terraform.

In dev, the EKS cluster runs **4 t3.small worker nodes** (min 1, max 5 — auto-scaled by the node group). On top of these nodes, we run **9 microservices**:

| Service | Language | Role |
|---|---|---|
| `pharma-ui` | React + Nginx | Frontend |
| `api-gateway` | Spring Boot | Routes all `/api/*` requests |
| `auth-service` | Spring Boot | JWT authentication |
| `drug-catalog-service` | Spring Boot | Drug catalog |
| `inventory-service` | Spring Boot | Stock management |
| `manufacturing-service` | Spring Boot | Manufacturing orders |
| `notification-service` | Spring Boot | Email/SMS notifications |
| `supplier-service` | Spring Boot | Supplier management |
| `qc-service` | Spring Boot | Quality control |

The supporting AWS infrastructure per environment:
- **VPC** with public, private, and database subnets
- **EKS** cluster (Kubernetes 1.33)
- **RDS PostgreSQL** in the database subnet (private, no public access)
- **9 ECR repositories** (one per service)
- **IAM roles** with OIDC federation for GitHub Actions and EKS node permissions
- **Secrets Manager** for DB credentials and JWT secret
- **S3** for Terraform state

---

### Q2. How do you manage different environments?

Each environment has its own Terraform workspace under `infra/envs/<env>/`:

```
infra/
  envs/
    dev/    ← main.tf, backend.tf, terraform.tfvars
    qa/     ← same structure, different values
    prod/   ← same structure, stricter sizing
  modules/ ← shared vpc, eks, rds, ecr, iam, secrets-manager modules
```

Each environment has:
- Its own **S3 state key** (`envs/dev/terraform.tfstate`, `envs/qa/terraform.tfstate`)
- Its own **GitHub Environment** (`dev`, `qa`, `prod`) for approval gates
- Its own **Kubernetes namespace** (`dev`, `qa`, `prod`)
- Its own **values files** in the gitops repo (`envs/dev/values-*.yaml`, `envs/qa/...`)
- Its own **ArgoCD Application** per service per environment

---

### Q3. How do you connect to different environments from your laptop?

We use `aws eks update-kubeconfig` to download cluster credentials locally:

```bash
# Connect to dev
aws eks update-kubeconfig --region us-east-1 --name pharma-dev-cluster

# Connect to qa
aws eks update-kubeconfig --region us-east-1 --name pharma-qa-cluster

# Switch between them
kubectl config get-contexts
kubectl config use-context <context-name>
```

Each `update-kubeconfig` call adds a new context to `~/.kube/config`. You switch with `kubectl config use-context`. For multiple AWS accounts (dev vs. qa/prod), you switch AWS profiles first:

```bash
export AWS_PROFILE=zenpharma-qa
aws eks update-kubeconfig --region us-east-1 --name pharma-qa-cluster
```

---

### Q4. How do pods communicate with the database?

The pods never have the DB password in their environment directly. The flow is:

1. **Terraform** creates the RDS instance in the **database subnet** (private, not publicly accessible) and stores credentials in **AWS Secrets Manager** (`pharma-dev-db-secret`).
2. **External Secrets Operator (ESO)** running in the cluster reads from Secrets Manager using an IAM role (via IRSA/OIDC) and creates a Kubernetes `Secret` in the `dev` namespace.
3. The **Helm chart** mounts that Kubernetes Secret as environment variables (`SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`) into the pod via `envFrom`.
4. The pod connects to the **RDS private endpoint** on port 5432 — the EKS node security group is whitelisted in the RDS security group, so only cluster traffic can reach it.

```
Pod → envFrom: [secretRef: pharma-dev-db-secret]
         ↑
External Secrets Operator ← AWS Secrets Manager
```

---

### Q5. How does ingress traffic flow?

```
Internet
   │
   ▼
AWS ALB (internet-facing, created by AWS Load Balancer Controller)
   │
   ├─ path: /      → pharma-ui (Nginx) pod, port 80
   └─ path: /api/* → api-gateway pod, port 8080
                          │
                          └─ routes to: auth-service, catalog-service,
                             inventory-service, manufacturing-service,
                             notification-service, supplier-service, qc-service
```

The ALB is **not created manually** — it is provisioned automatically by the **AWS Load Balancer Controller** when ArgoCD applies an `Ingress` manifest with `ingressClassName: alb`. The key annotations:

```yaml
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip        # routes directly to pod IP
alb.ingress.kubernetes.io/group.name: pharma-dev # share one ALB across services
```

---

### Q6. How do you connect this project to a domain name?

Currently the project is accessed via the ALB's auto-generated DNS name. To connect a custom domain:

1. Buy or manage the domain in **Route 53**.
2. Create a **CNAME** or **Alias record** in Route 53 pointing `app.zenpharma.com` → the ALB DNS name.
3. Request an **ACM certificate** for `app.zenpharma.com` and attach it to the ALB via the Ingress annotation:
   ```yaml
   alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:ACCOUNT:certificate/...
   alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
   alb.ingress.kubernetes.io/ssl-redirect: '443'
   ```
4. Set `host: app.zenpharma.com` in the Helm values file so the Ingress rule matches the hostname.
5. Re-deploy via ArgoCD and the ALB listener is updated automatically.

---

### Q7. EKS Upgrade — How would you upgrade the Kubernetes version?

EKS upgrades must go one minor version at a time (e.g., 1.33 → 1.34, not 1.33 → 1.35).

**Step 1 — Update Terraform config:**
```hcl
# envs/dev/main.tf
module "eks" {
  kubernetes_version = "1.34"   # bump from 1.33
}
```

**Step 2 — Apply via pipeline:**
- Push to a branch → open PR → Terraform plan shows the version change → merge → approval gate → apply.

**Step 3 — Upgrade managed node group:**
- After the control plane is upgraded, update node group `ami_type` / `version` in Terraform.
- EKS rolling-replaces nodes so pods are not all evicted at once (respects PodDisruptionBudgets).

**Step 4 — Upgrade add-ons:**
- Update the AWS Load Balancer Controller, CoreDNS, kube-proxy, and VPC CNI to versions compatible with the new control plane.

**Step 5 — Test:**
- Verify all pods are running post-upgrade: `kubectl get pods -A`
- Check ArgoCD is healthy and synced

---

### Q8. How do you rotate / update the database password?

**Option A — Update via GitHub Actions (recommended):**

1. Change the `DEV_DB_PASSWORD` secret in GitHub → the next Terraform `apply` run updates both RDS and Secrets Manager.
2. ESO re-syncs the Kubernetes Secret automatically.
3. Pods using `envFrom` pick up the new secret on their next restart: `kubectl rollout restart deployment -n dev`

**Option B — Manual rotation:**

```bash
# 1. Update Secrets Manager
aws secretsmanager put-secret-value \
  --secret-id pharma-dev-db-secret \
  --secret-string '{"username":"pharmaadmin","password":"new-password",...}'

# 2. Update RDS master password
aws rds modify-db-instance \
  --db-instance-identifier pharma-dev-postgres \
  --master-user-password new-password \
  --apply-immediately

# 3. Force ESO to re-sync
kubectl annotate externalsecret pharma-dev-db-secret -n dev \
  force-sync=$(date +%s) --overwrite

# 4. Restart pods to pick up the new Kubernetes Secret
kubectl rollout restart deployment -n dev
```

---

## 2. CI/CD Pipeline

### Q9. Can you explain your CI/CD pipeline?

We have **3 separate GitHub repositories**, each with its own pipeline:

**1. Infra repo** (`zenpharma/infra`) — `terraform.yml`:
- Trigger: PR to `main` → plan only; push to `main` → plan → approval gate → apply
- Provisions all AWS infrastructure (VPC, EKS, RDS, ECR, IAM, Secrets Manager)

**2. Frontend repo** (`zenpharma/frontend`) — `ci-pharma-ui.yml`:
- Trigger: push to `develop` or `release/**`
- Stages: Lint → Unit Tests + Coverage → SonarCloud SAST → Build React → Docker Build + Trivy Scan → Push to ECR → Update DEV gitops values → ArgoCD auto-deploys

**3. Backend repo** (`zenpharma/backend`) — per-service `ci-<service>.yml` calling reusable `_java-build.yml`:
- Trigger: push to `develop` (only files changed in that service's directory)
- Stages: Maven verify (tests + JaCoCo ≥80% coverage) → SonarCloud SAST → Docker Build → Trivy Scan → Push to ECR → Cosign sign → Deploy to DEV

**Promotion to QA/PROD** — separate `promote-qa.yml` / `promote-prod.yml` workflows triggered manually (`workflow_dispatch`).

---

### Q10. Can you walk me through your GitHub Actions pipeline in detail?

Using the backend Java pipeline (`_java-build.yml`) as the example:

```
1. Checkout code
2. Set image tag → sha-${GITHUB_SHA::7}   (e.g., sha-a3f9b21)
3. Setup Java 17 (Temurin) with Maven cache
4. (Optional) Start PostgreSQL container for integration tests
5. Maven verify → compiles, runs unit + integration tests, generates JaCoCo coverage
6. Upload JaCoCo report as artifact (retained 7 days)
7. SonarCloud scan → SAST + coverage analysis sent to sonarcloud.io
8. Configure AWS credentials via OIDC (no static keys in repo)
9. Login to Amazon ECR
10. Docker build → multi-stage, non-root user (UID/GID 1000)
11. Trivy image scan → reports HIGH/CRITICAL CVEs (non-blocking currently)
12. Push image to ECR tagged sha-<7chars>
13. Cosign sign (keyless) → GitHub OIDC → Fulcio CA → Rekor transparency log
14. Post build summary to GitHub Actions step summary
```

Key design decisions:
- **Reusable workflows** (`workflow_call`): all 8 Java services call `_java-build.yml` — one place to update security gates
- **Artifacts** are passed between jobs (build → deploy) using `upload-artifact`/`download-artifact`
- **`concurrency` group** prevents parallel runs on the same branch to avoid state conflicts

---

### Q11. How do you send secrets to your pipeline?

We use **three layers**:

**Layer 1 — GitHub Repository Secrets** (available to all workflows):
- `AWS_ACCOUNT_ID` — for constructing the OIDC role ARN
- `SONAR_TOKEN` — SonarCloud authentication
- `GITOPS_TOKEN` — PAT with write access to the gitops repo

**Layer 2 — GitHub Environment Secrets** (scoped to `dev`, `qa`, `prod` environments):
- `DEV_DB_PASSWORD`, `DEV_JWT_SECRET` — used by Terraform apply for the dev environment
- `QA_DB_PASSWORD`, `QA_JWT_SECRET` — available only to workflows targeting the `qa` environment

**Layer 3 — AWS Secrets Manager** (runtime secrets, not in GitHub at all):
- DB password, DB host, JWT secret are stored in Secrets Manager after Terraform creates them
- Pods access them via External Secrets Operator at runtime — never in environment variables at build time

**What we deliberately avoid:**
- Static `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` in GitHub for ECR push — we use **OIDC federation** instead (see Q15)

---

### Q12. What is your branching strategy?

```
feature/xyz  ──┐
               ▼
            develop  ──→  (CI runs here — build, test, push image, deploy to dev)
               │
               ▼
           Pull Request
               │
               ▼
             main  ──→  (Terraform CI, protected — requires PR review + status checks)
               │
           release/1.2.0  ──→  (optional release branch, triggers CI)
```

- **`main`** is protected — no direct pushes, requires PR approval
- **`develop`** is the integration branch — all feature PRs merge here first
- **CI triggers on `develop`** — lint, test, build, push image, auto-deploy to dev
- **`main` triggers Terraform** — infrastructure changes go through a plan → approval → apply gate
- **`release/*`** branches are cut for stable releases; CI also runs on these

---

### Q13. Can you migrate from GitHub-hosted runners to self-hosted runners?

Yes. The main change is replacing `runs-on: ubuntu-latest` with your self-hosted runner label:

```yaml
# Before
runs-on: ubuntu-latest

# After
runs-on: [self-hosted, linux, x64]
```

**Steps to migrate:**

1. Provision a VM or EC2 instance with the GitHub Actions runner agent installed.
2. Attach an IAM role to the EC2 (instead of OIDC from GitHub). Update `configure-aws-credentials` to use `role-to-assume` with EC2 instance metadata.
3. Pre-install Docker, Java 17, kubectl, Helm, Terraform on the runner.
4. Register the runner in GitHub → Settings → Actions → Runners → New self-hosted runner.
5. Replace `runs-on` in all workflow files.

**Why you might do this:**
- Faster builds (no cold-start, cached Maven/npm dependencies persist)
- Private network access (runner inside VPC can reach RDS, EKS API server directly)
- Cost savings for high-volume pipelines

**Tradeoff:** You own the runner maintenance, scaling, and security patching.

---

## 3. Security

### Q14. How is security enforced in this project?

Security is layered across build time, deploy time, and runtime:

**Build time:**
- **SonarCloud SAST** — static analysis on every push for frontend (JavaScript) and backend (Java). Fails if quality gate is not met.
- **Trivy image scanning** — scans Docker images for HIGH and CRITICAL CVEs before push to ECR.
- **OWASP Dependency Check** — scans Maven dependencies for known vulnerabilities (CVSS ≥ 7.0).
- **npm audit** — checks frontend packages for vulnerabilities.
- **Cosign keyless signing** — images are signed using GitHub OIDC → Fulcio CA → Rekor transparency log. Every image push has an immutable audit trail.

**Deploy time:**
- **OIDC federation** — GitHub Actions assumes an AWS IAM role using a short-lived token. No static `AWS_ACCESS_KEY_ID` anywhere.
- **GitHub Environments** — `dev`, `qa`, `prod` gates require manual approval before `terraform apply` or sensitive workflow steps run.
- **ArgoCD RBAC** — only ArgoCD's service account can deploy to the cluster.

**Runtime:**
- **Pod Security Context** (in `values.yaml`):
  ```yaml
  podSecurityContext:
    runAsNonRoot: true
    runAsUser: 1000
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop: [ALL]
  ```
- **External Secrets Operator** — DB passwords and JWT secrets are never baked into images or ConfigMaps. They are pulled from AWS Secrets Manager at runtime.
- **Private RDS** — database is in a private subnet with a security group allowing only EKS node traffic on port 5432.
- **ECR image scanning** on every push.

---

### Q15. What is OIDC and why don't you use static AWS keys?

**OIDC (OpenID Connect)** is an identity federation protocol. GitHub Actions supports it natively — when a workflow runs, GitHub issues a short-lived JWT token proving "this is workflow X in repo Y on branch Z."

AWS is configured to **trust GitHub's OIDC provider**. The workflow exchanges the GitHub token for temporary AWS credentials (valid for 1 hour) with specific IAM permissions.

```yaml
- name: Configure AWS credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v5
  with:
    role-to-assume: arn:aws:iam::ACCOUNT:role/pharma-dev-github-actions-role
    aws-region: us-east-1
```

**Why not static keys?**
- Static keys never expire — a leak gives permanent access
- Keys must be rotated manually; OIDC tokens expire in 1 hour automatically
- GitHub Secrets can be accidentally logged or leaked in PR comments
- OIDC credentials are scoped to a specific repo, branch, and environment — principle of least privilege

---

### Q16. What is your image cleanup process?

**Automated (ECR Lifecycle Policy):**  
Every ECR repository has a lifecycle policy (set by Terraform) that retains only the last 10 images. Older images are automatically expired by AWS.

```json
{
  "rulePriority": 1,
  "description": "Keep last 10 images",
  "selection": { "tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 10 },
  "action": { "type": "expire" }
}
```

**Manual (before terraform destroy):**  
Before running `terraform destroy`, ECR repos must be manually emptied because Terraform cannot delete a repository that contains images:

```bash
aws ecr batch-delete-image \
  --repository-name pharma-ui \
  --image-ids "$(aws ecr list-images --repository-name pharma-ui --query 'imageIds' --output json)"
```

See `RUNBOOK-DESTROY.md` for the full loop script.

---

## 4. Kubernetes & Helm

### Q17. Can you walk me through your Deployment file?

Our Deployment is a **Helm template** at `gitops/helm-charts/templates/deployment.yaml`. Key sections:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}       # from values.yaml
  template:
    spec:
      serviceAccountName: pharma-ui          # IRSA-annotated SA
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000                      # non-root user
      containers:
        - image: "<ECR_URL>:<sha-tag>"       # injected by CI
          envFrom:
            - configMapRef: ...              # app config (API_BASE_URL, ENV)
            - secretRef: ...                 # DB password, JWT (from ESO)
          livenessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 10          # wait 10s before first check
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 5           # wait 5s before marking ready
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits: { cpu: 200m, memory: 128Mi }
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: [ALL] }
```

**Liveness vs. Readiness Probe:**
- **Liveness** — if this fails 3 times, Kubernetes **restarts** the pod (the pod is broken).
- **Readiness** — if this fails, Kubernetes **removes the pod from the Service endpoints** (stops sending traffic). Used during startup and when the pod is temporarily unhealthy.

---

### Q18. Why do you use Helm charts? What is the `_helpers.tpl` file?

**Why Helm:**
We have 9 microservices with nearly identical Kubernetes resources (Deployment, Service, Ingress, ServiceAccount, ConfigMap, HPA). Without Helm, that would be 9 × 6 = 54 YAML files to maintain. Helm gives us one **shared chart** (`pharma-service`) with a `values.yaml` that changes per service per environment.

```
gitops/helm-charts/          ← one shared chart
gitops/envs/dev/
  values-pharma-ui.yaml      ← pharma-ui specific overrides
  values-api-gateway.yaml    ← api-gateway specific overrides
  ...
```

**`_helpers.tpl`:**  
A special Helm template file (not rendered as a Kubernetes manifest) that defines **reusable named templates** — Go template functions used across all templates:

```
{{- define "pharma-service.fullname" -}}
  {{ .Values.fullnameOverride | default (printf "%s-%s" .Release.Name .Chart.Name) }}
{{- end -}}

{{- define "pharma-service.labels" -}}
  app.kubernetes.io/name: {{ include "pharma-service.fullname" . }}
  app.kubernetes.io/instance: {{ .Release.Name }}
  app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}
```

This keeps labels and names consistent across the Deployment, Service, Ingress, and HPA without copy-pasting.

---

### Q19. What Kubernetes services do your pods use?

Each microservice uses the following Kubernetes resources (all templated in the shared Helm chart):

| Resource | Purpose |
|---|---|
| `Deployment` | Runs the container, manages rolling updates |
| `Service` (ClusterIP) | Internal DNS name + load balancing within the cluster |
| `Ingress` | Declares ALB routing rules (pharma-ui and api-gateway only) |
| `ServiceAccount` | Identity for the pod — IRSA annotations grant AWS permissions |
| `ConfigMap` | Non-sensitive config: `API_BASE_URL`, `ENV`, app settings |
| `HorizontalPodAutoscaler` | Scales pods based on CPU/memory (currently disabled, ready to enable) |

---

### Q20. How do you add a new microservice to the existing environment?

1. **Create the service code** in the `backend` repo with its own directory.
2. **Add an ECR repository** in `infra/envs/dev/main.tf` — add the name to the `repositories` list and apply.
3. **Create a CI workflow** `backend/.github/workflows/ci-new-service.yml` that calls `_java-build.yml`.
4. **Add a values file** in the gitops repo: `envs/dev/values-new-service.yaml` with image, resources, probes.
5. **Create an ArgoCD Application** at `argocd/apps/dev/new-service-app.yaml` pointing to the helm chart with the new values file.
6. **Add to ArgoCD AppProject** if the destination namespace is new.
7. Merge to `develop` → CI builds and pushes image → ArgoCD deploys automatically.

---

## 5. GitOps & ArgoCD

### Q21. Can you explain the Application file in ArgoCD?

The `Application` is ArgoCD's core CRD. It tells ArgoCD **where to get the config** and **where to deploy it**:

```yaml
# argocd/apps/dev/pharma-ui-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pharma-ui-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # deletes K8s resources when app is deleted
spec:
  project: pharma          # which AppProject controls allowed sources/destinations

  source:
    repoURL: https://github.com/zenpharma/gitops.git
    targetRevision: HEAD
    path: helm-charts      # use the shared Helm chart
    helm:
      valueFiles:
        - ../envs/dev/values-pharma-ui.yaml    # with these values

  destination:
    server: https://kubernetes.default.svc     # this cluster
    namespace: dev                             # deploy into the dev namespace

  syncPolicy:
    automated:
      prune: true       # delete removed resources
      selfHeal: true    # revert manual kubectl changes
    retry:
      limit: 5
      backoff: { duration: 5s, factor: 2, maxDuration: 3m }
```

When anyone merges a commit to the gitops repo that changes `values-pharma-ui.yaml`, ArgoCD detects it within ~3 minutes and rolls out the new image automatically.

---

### Q22. What is the difference between an ArgoCD Project and an Application?

| | **AppProject** | **Application** |
|---|---|---|
| Kind | `AppProject` | `Application` |
| Purpose | **Governance/policy** — defines what is allowed | **Deployment unit** — defines what to deploy |
| Controls | Allowed source repos, allowed destination clusters/namespaces, allowed resource kinds | Specific source repo + path + values → specific destination |
| Count | One per team/product | One per service per environment |

In our project:
- **One `AppProject`** named `pharma` — allows the gitops repo as source and `dev`, `qa`, `prod` namespaces as destinations.
- **Many `Application`s** — `pharma-ui-dev`, `api-gateway-dev`, `pharma-ui-qa`, etc.

Think of the Project as the security policy, and the Application as the actual "deploy this thing."

---

### Q23. What is your image promotion policy?

Images flow **one direction only** and are **never rebuilt** — the same image tag that passed all CI gates in dev goes to QA and prod unchanged.

```
DEV (auto, on push to develop)
  │
  │ manual: promote-qa.yml (workflow_dispatch)
  │   1. Reads image tag from envs/dev/values-<svc>.yaml
  │   2. Opens a PR in gitops to update envs/qa/values-<svc>.yaml
  │   3. QA team reviews PR → merges
  │   4. ArgoCD auto-syncs QA
  ▼
QA
  │
  │ manual: promote-prod.yml (workflow_dispatch)
  │   Same pattern → PR to envs/prod/values-<svc>.yaml
  ▼
PROD
```

Image tag format: `sha-<first-7-chars-of-git-commit-sha>` (e.g., `sha-a3f9b21`).  
This ties every running image back to an exact git commit, enabling full traceability.

---

## 6. Docker & Image Management

### Q24. Can you explain multi-stage Docker build? What is the advantage?

A **multi-stage build** uses multiple `FROM` statements in one Dockerfile. Each stage can copy artifacts from the previous stage, leaving behind build tools and intermediate files.

**Frontend (pharma-ui):**
```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder    # full Node.js + npm
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY src ./src
COPY public ./public
RUN npm run build                 # produces /app/build/

# Stage 2: Serve
FROM nginx:1.25-alpine            # tiny Nginx image
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Backend (Java services):**
```dockerfile
# Stage 1: Build (from CI — Maven compiles .jar before Docker)
FROM eclipse-temurin:17-jre   # JRE only, not full JDK
WORKDIR /app
RUN groupadd -r pharma && useradd -r -g pharma pharma
COPY target/*.jar app.jar
USER pharma
EXPOSE 8080
```

**Advantages:**
- **Smaller final image** — the Node.js builder (>600MB) is not in the final image; only Nginx (~25MB) is shipped.
- **No build tools in production** — no npm, gcc, or Maven in the running container = smaller attack surface.
- **Security** — the final image has fewer packages and fewer CVEs to worry about.
- **Reproducibility** — the same Dockerfile builds both dev and prod images.

---

## 7. Environment & Secret Management

### Q25. What is External Secrets Operator and how does it work?

**External Secrets Operator (ESO)** is a Kubernetes controller that reads secrets from external systems (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, etc.) and creates native Kubernetes `Secret` objects in the cluster.

**How we use it:**

```
AWS Secrets Manager
  pharma-dev-db-secret = { username, password, host }
        ↑ reads (IRSA role)
External Secrets Operator (pod in external-secrets namespace)
        ↓ creates
Kubernetes Secret (in dev namespace)
  name: pharma-dev-db-secret
        ↓ mounted as
Pod environment variables
  SPRING_DATASOURCE_URL, SPRING_DATASOURCE_USERNAME, SPRING_DATASOURCE_PASSWORD
```

We define an `ExternalSecret` resource that maps Secrets Manager keys to Kubernetes Secret keys. ESO reconciles every 1 hour — if the Secrets Manager value changes, the Kubernetes Secret is updated automatically.

**Why not put secrets in GitHub or in ConfigMaps?**  
ConfigMaps are not encrypted. GitHub Secrets can't be accessed from running pods. Secrets Manager provides encryption, audit logging, automatic rotation, and fine-grained IAM access.

---

## 8. Troubleshooting

### Q26. One microservice is not working. How do you find the issue?

**Step 1 — Check pod status:**
```bash
kubectl get pods -n dev
# Look for: CrashLoopBackOff, OOMKilled, ImagePullBackOff, Pending, Error
```

**Step 2 — Read pod logs:**
```bash
kubectl logs <pod-name> -n dev
kubectl logs <pod-name> -n dev --previous   # logs from the crashed container
```

**Step 3 — Describe the pod for events:**
```bash
kubectl describe pod <pod-name> -n dev
# Check Events section: image pull failure, OOM, probe failure, node issues
```

**Step 4 — Check ArgoCD sync status:**
- Is ArgoCD showing the app as `OutOfSync` or `Degraded`?
- Are there resource conflicts, Helm rendering errors, or missing CRDs?

**Step 5 — Check External Secrets:**
```bash
kubectl get externalsecret -n dev
# STATUS column — SecretSyncedError means DB credentials are missing/wrong
```

**Step 6 — Test connectivity:**
```bash
# Can the pod reach the DB?
kubectl run debug --rm -it --image=postgres:15-alpine -n dev -- \
  psql -h <RDS_ENDPOINT> -U pharmaadmin -d pharmadb
```

**Step 7 — Check HPA and resource limits:**
```bash
kubectl top pods -n dev
# Is a pod being OOMKilled because memory limits are too low?
```

---

### Q27. A user cannot reach the login page. How do you troubleshoot?

Work from the outside in:

```
User browser → DNS → ALB → pod → (api-gateway → auth-service → RDS)
```

**Step 1 — Check DNS / ALB:**
```bash
curl -v http://<ALB-DNS-NAME>/
# Should return HTTP 200 for the React app
```

**Step 2 — Check ALB Target Health:**
- AWS Console → EC2 → Target Groups → check if the pharma-ui pod is "healthy"
- Unhealthy = pod not passing health checks or not running

**Step 3 — Check pharma-ui pod:**
```bash
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui
kubectl logs <pharma-ui-pod> -n dev
```

**Step 4 — Check Ingress was created:**
```bash
kubectl get ingress -n dev
# ADDRESS column should show the ALB DNS name
```

**Step 5 — Test the login API directly:**
```bash
curl -v http://<ALB-DNS>/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"changeme"}'
# If this fails, the issue is in api-gateway or auth-service, not pharma-ui
```

**Step 6 — Check auth-service:**
```bash
kubectl logs <auth-service-pod> -n dev
# Look for: DB connection errors, invalid JWT config, startup failures
```

**Step 7 — Check External Secrets (JWT secret missing?):**
```bash
kubectl get externalsecret -n dev
kubectl get secret pharma-dev-jwt-secret -n dev -o jsonpath='{.data}' | base64 -d
```

---

### Q28. What is the rollback process?

ArgoCD makes rollback straightforward because the gitops repo is the source of truth.

**Option A — Rollback via ArgoCD UI:**
1. Open ArgoCD → click the application → **History and Rollback**
2. Select a previous revision (each revision = a gitops commit)
3. Click **Rollback** — ArgoCD re-applies the old values (which had the previous image tag)

**Option B — Rollback via GitOps (recommended for audit trail):**
1. Revert the image tag update commit in the gitops repo:
   ```bash
   git revert HEAD   # reverts the "ci(dev): update pharma-ui → sha-a3f9b21" commit
   git push
   ```
2. ArgoCD auto-syncs and deploys the previous image.

**Option C — Emergency kubectl rollout (last resort):**
```bash
kubectl rollout undo deployment/pharma-ui -n dev
# This works but puts ArgoCD out of sync — fix by reverting gitops commit too
```

**What makes rollback safe:**
- We never overwrite image tags — `sha-<7chars>` is immutable
- Old images are retained in ECR (last 10)
- ArgoCD revision history is kept for 10 revisions (`revisionHistoryLimit: 10`)

---

## 9. Operational Topics

### Q29. Can you migrate from GitHub-hosted to self-hosted runners?

*(Answered in Q13 above — see CI/CD section)*

---

### Q30. How do you handle Terraform state locking?

We use **S3 native locking** (Terraform ≥ 1.11), configured with `use_lockfile = true` in `backend.tf`. When a `terraform plan` or `apply` starts, it creates a lock file at:

```
s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock
```

If a workflow is cancelled mid-run, the lock file may persist and block future runs. Release it with:

```bash
aws s3 rm s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock
```

The workflows also include an automatic lock release step on failure or cancellation:
```yaml
- name: Release state lock on failure
  if: failure() || cancelled()
  run: aws s3 rm s3://${{ vars.TF_STATE_BUCKET }}/envs/dev/terraform.tfstate.tflock || true
```

---

### Q31. What is your image promotion policy? (detailed)

*(Answered in Q23 — see GitOps section)*

---

## 10. Python & Automation

### Q32. What activities did you do using Python?

We wrote 6 Python scripts for cluster lifecycle management (in `infra/scripts/`):

| Script | What it does |
|---|---|
| `01_install_prerequisites.py` | Updates kubeconfig, installs AWS Load Balancer Controller via Helm, configures IAM role for the controller |
| `02_bootstrap_argocd.py` | Installs ArgoCD via Helm, registers the gitops repo, creates the `pharma` AppProject |
| `03_setup_external_secrets.py` | Installs ESO, creates `ClusterSecretStore` pointing at Secrets Manager, creates `ExternalSecret` resources |
| `04_run_pipeline.py` | Triggers GitHub Actions workflows programmatically |
| `05_deploy_services.py` | Creates ArgoCD Application manifests and syncs them |
| `06_verify_deployment.py` | Checks pod health, secret sync status, and ALB target health after deployment |

**Why Python instead of Bash?**
- Easier error handling and conditional logic
- Can use `boto3` (AWS SDK) and the Kubernetes Python client
- More readable for complex multi-step workflows
- Easier to add interactive prompts with validation

---

## 11. AI & Advanced Topics

### Q33. In which parts of this project can you take advantage of AI?

Several high-value integration points:

**1. Predictive Alerting:**  
Train a model on historical pod CPU/memory metrics and deployment events. Alert before an OOM happens rather than after a crash. Tools: Amazon SageMaker + CloudWatch metrics.

**2. Intelligent Log Analysis:**  
Instead of grepping logs manually during incidents, feed pod logs to an LLM (Claude API) to summarize the root cause: "The auth-service pod crashed because the DB connection pool was exhausted when RDS hit 100% CPU."

**3. PR Code Review:**  
Add an AI step in the CI pipeline that comments on PRs with security suggestions, anti-patterns, and improvement ideas. Can integrate SonarCloud findings with LLM explanations.

**4. Auto-Generated Release Notes:**  
Use `git log` between tags + LLM to generate human-readable changelogs automatically for each promotion.

**5. Anomaly Detection:**  
Monitor HTTP response codes and latencies from ALB access logs. Use ML to detect unusual patterns (e.g., spike in 5xx on a single service after a deployment).

**6. ChatOps:**  
A Slack bot backed by Claude API that answers "What is currently deployed in QA?" by querying the gitops repo, or "Rollback auth-service in dev" by triggering the rollback workflow.

**7. Infrastructure Cost Optimization:**  
Analyze CloudWatch metrics to identify over-provisioned nodes or pods with too-high resource requests, and suggest right-sizing.

---

### Q34. What is predictive analysis in the context of this project?

Predictive analysis uses **historical data to forecast future events** before they cause failures.

**Examples in ZenPharma:**

- **Pod OOM prediction**: If a pod's memory usage grows 5% per day and its limit is 128Mi, you can predict it will hit the limit in N days. Scale up resources or fix the memory leak proactively.
- **Node capacity prediction**: EKS cluster has 4 nodes. If pod count grows by 2 per week, you can predict when the scheduler will start failing to place pods due to resource exhaustion — and add nodes in advance.
- **DB connection exhaustion**: RDS has a max connection limit. If the number of deployed pods grows (more services, more replicas), predict when connection pool limits will be hit.
- **Deployment failure prediction**: Track which types of PRs (large size, touching many files, late-night commits) historically correlate with failed deployments. Flag high-risk PRs before merge.

Tools: **Amazon CloudWatch Metrics** + **SageMaker** for model training, or simpler threshold-based alerting via CloudWatch Alarms.

---

## Quick Reference — Common Commands

```bash
# Check what's running in dev
kubectl get pods -n dev
kubectl get services -n dev
kubectl get ingress -n dev

# Check ArgoCD app health
kubectl get applications -n argocd

# Check secrets are synced
kubectl get externalsecret -n dev

# Get ALB URL
kubectl get ingress pharma-ui -n dev -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Check logs
kubectl logs -l app.kubernetes.io/name=auth-service -n dev --tail=100

# Check pod resource usage
kubectl top pods -n dev

# Get RDS endpoint
aws rds describe-db-instances \
  --query "DBInstances[?contains(DBInstanceIdentifier,'pharma-dev')].Endpoint.Address" \
  --output text

# Release stuck Terraform lock
aws s3 rm s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock

# Force ESO re-sync
kubectl annotate externalsecret <name> -n dev force-sync=$(date +%s) --overwrite
```
