# ZenPharma DevOps — Interview Q&A Guide

> Each question has two answers:
> - **📋 In Our Project** — what we actually built and how it works
> - **🎤 Interview Answer** — how to explain it convincingly to an interviewer, with real-world framing

---

## Table of Contents
1. [Infrastructure & AWS](#1-infrastructure--aws)
2. [CI/CD Pipeline](#2-cicd-pipeline)
3. [Security](#3-security)
4. [Kubernetes, Helm & ArgoCD](#4-kubernetes-helm--argocd)
5. [Docker & Image Management](#5-docker--image-management)
6. [Environment & Secret Management](#6-environment--secret-management)
7. [Troubleshooting](#7-troubleshooting)
8. [Real Incidents & Operational Decisions](#8-real-incidents--operational-decisions)
9. [Version Management & Maintenance](#9-version-management--maintenance)
   - Q36: [EKS Terraform module major version upgrade](#q36-how-would-you-upgrade-the-eks-terraform-module-from--210-to-a-newer-major-version-what-do-you-need-to-check-first)
10. [Advanced & AI Topics](#10-advanced--ai-topics)

---

## 1. Infrastructure & AWS

---

### Q1. Can you explain your environment? How many servers and microservices?

**📋 In Our Project:**

Three environments: dev, qa, prod. Dev has 4 t3.small EKS nodes (min 1, max 5). We run 9 microservices: pharma-ui (React/Nginx), api-gateway, auth-service, drug-catalog-service, inventory-service, manufacturing-service, notification-service, supplier-service, qc-service. All environments share the same AWS account, separated by Kubernetes namespaces (dev, qa, prod), separate Terraform state files, and separate IAM roles.

**🎤 Interview Answer:**

> "We run a pharmaceutical microservices platform across three environments — dev, QA, and production. In real-world practice, each environment lives in its own AWS account, which gives you hard blast-radius isolation — a production misconfiguration simply cannot touch dev resources. The accounts are managed under AWS Organizations with a management account at the top.
>
> We have 9 microservices: a React frontend served by Nginx, a Spring Boot API gateway that routes all backend traffic, and 7 domain services — auth, drug catalog, inventory, manufacturing, notification, supplier, and QC. The backend services are all Java 17 on Spring Boot.
>
> In dev, the EKS cluster runs 4 t3.small nodes with auto-scaling enabled. In production, we'd size up to c5.xlarge or m5.large nodes based on load testing results. RDS PostgreSQL is in a private subnet with no public endpoint — only the EKS node security group is allowed to connect on port 5432."

---

### Q2. How do you manage different environments?

**📋 In Our Project:**

Each environment has its own folder under `infra/envs/<env>/` with separate `backend.tf` (separate S3 state key), `main.tf`, and `terraform.tfvars`. Each has its own GitHub Environment (`dev`, `qa`, `prod`) for approval gates and scoped secrets. In the gitops repo, each environment has its own values files (`envs/dev/values-*.yaml`, `envs/qa/...`) and ArgoCD Application manifests.

**🎤 Interview Answer:**

> "We follow the principle of environment parity — the same Terraform modules, the same Helm chart, the same Docker image run in all three environments. What changes is just the configuration: instance sizes, replica counts, domain names, and credentials.
>
> The architecture has four layers of environment isolation:
>
> 1. **Infrastructure layer** — separate AWS accounts (dev/qa/prod) under AWS Organizations. Terraform state is per-environment in S3. A plan in dev never touches prod state.
> 2. **Pipeline layer** — GitHub Environments act as approval gates. The prod apply job requires a human reviewer to approve before Terraform touches production AWS resources.
> 3. **Application layer** — Kubernetes namespaces (dev, qa, prod) on separate clusters. Namespace-scoped RBAC ensures a deployment to QA cannot affect prod pods.
> 4. **Config layer** — separate Helm values files per environment in the gitops repo. ArgoCD reads the environment-specific values file, so the same Helm chart renders differently in each env.
>
> The benefit is that promoting an application from dev to prod is just a PR that updates an image tag in a YAML file — no code change, no Dockerfile change, no infrastructure change."

---

### Q3. In real world, you'd have different AWS accounts per environment — how does that work?

**📋 In Our Project:**

Currently we use a single AWS account. The design is multi-account-ready — separate IAM roles, separate Terraform state, separate GitHub Environments — but account separation hasn't been implemented.

**🎤 Interview Answer:**

> "Yes, in production we follow the AWS recommended multi-account strategy using AWS Organizations. You have a management (root) account and separate member accounts for each environment.
>
> The setup works like this: each GitHub Environment (`dev`, `qa`, `prod`) stores its own `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` — or better, each environment has its own OIDC role ARN that assumes a role in the target AWS account. When the pipeline targets QA, it assumes `arn:aws:iam::QA-ACCOUNT-ID:role/github-actions-role`. When it targets prod, it assumes the prod account's role.
>
> The real benefit of multi-account is that Service Control Policies (SCPs) at the organization level can deny actions in prod that are allowed in dev — for example, `DenyDeleteRDSInProd`. No developer can accidentally run `terraform destroy` in production even if they have the credentials, because the SCP at the organization level blocks it.
>
> Cost allocation is also clean — each account's AWS bill is the cost of that environment, no tagging gymnastics required."

---

### Q4. How do pods communicate with the database?

**📋 In Our Project:**

Terraform creates RDS in the database subnet and stores credentials in AWS Secrets Manager. External Secrets Operator (ESO) reads from Secrets Manager using an IRSA role and creates a Kubernetes Secret in the dev namespace. The Helm chart mounts that Secret via `envFrom` into each pod as `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`. A security group rule allows only EKS nodes to reach RDS on port 5432.

**🎤 Interview Answer:**

> "This is a question about three things: network path, credential injection, and secret rotation — and we handle all three.
>
> **Network path:** RDS is in a database subnet with no public endpoint. The EKS node security group is whitelisted in the RDS security group on port 5432. Nothing outside the VPC can reach the database — not even other services in AWS unless they're explicitly allowed.
>
> **Credential injection:** We don't put database passwords in ConfigMaps or bake them into images. We use External Secrets Operator. It runs as a controller in the cluster, authenticates to AWS Secrets Manager using an IAM role via IRSA, and creates a native Kubernetes Secret in each namespace. The Helm Deployment template uses `envFrom: [secretRef]` to inject those as environment variables at pod startup.
>
> **Secret rotation:** If we rotate the DB password in Secrets Manager, ESO re-syncs the Kubernetes Secret within the configured interval. A rolling restart of the deployment picks up the new credentials — zero downtime because Kubernetes keeps old pods running until new ones are healthy.
>
> The key principle: credentials never appear in git, never appear in CI logs, and never live on disk. They are pulled fresh from Secrets Manager at runtime."

---

### Q5. How does ingress traffic flow?

**📋 In Our Project:**

The AWS Load Balancer Controller (installed via Helm in kube-system) watches Ingress resources with `ingressClassName: alb`. When ArgoCD applies the pharma-ui Ingress, the controller provisions an internet-facing ALB with the annotation `alb.ingress.kubernetes.io/group.name: pharma-dev` — all services sharing this group share one ALB. Traffic flows: Internet → ALB → pod IP directly (`target-type: ip`). pharma-ui handles `/`, api-gateway handles `/api/*`.

**🎤 Interview Answer:**

> "We use the AWS Load Balancer Controller, which provisions ALBs natively from Kubernetes Ingress resources. The traffic path is:
>
> ```
> User → Route 53 (DNS) → ALB (internet-facing, port 443)
>   ├─ path /       → pharma-ui Nginx pod (port 80)
>   └─ path /api/*  → api-gateway pod (port 8080)
>                          └─ routes internally to backend services via ClusterIP
> ```
>
> We use `target-type: ip` which means the ALB routes directly to pod IPs — no extra hop through a NodePort or kube-proxy. This reduces latency and makes health checks more accurate because they hit the actual pod, not a node.
>
> The ALB group annotation `alb.ingress.kubernetes.io/group.name: pharma-dev` means all our Ingress resources share a single ALB — we don't pay for a separate ALB per service. SSL terminates at the ALB. The backend pods only receive plain HTTP.
>
> All of this is GitOps-managed — the ALB is created and updated by ArgoCD applying Ingress manifests, not by a human clicking in the AWS console."

---

### Q6. How did you migrate from ingress-nginx to AWS Load Balancer Controller?

**📋 In Our Project:**

We started directly with AWS Load Balancer Controller — ingress-nginx was never used in this project.

**🎤 Interview Answer:**

> "In our previous setup we used ingress-nginx, which is the default choice for many teams. The problem was that on AWS, ingress-nginx sits behind an NLB — so the traffic path was NLB → nginx pod → ClusterIP → pod. That's three hops for every request. We were also paying for both the NLB and the compute resources for the nginx pods themselves.
>
> We migrated to the AWS Load Balancer Controller. Here's exactly what the migration involved:
>
> **Step 1:** Create the IAM policy and IRSA role that the ALB controller needs. This is a Terraform change — we added it to the IAM module.
>
> **Step 2:** Install the AWS Load Balancer Controller via Helm in kube-system. We configured it with the IRSA service account annotation so it can call AWS APIs.
>
> **Step 3:** Update Ingress manifests. This is the main migration effort — change `ingressClassName: nginx` to `ingressClassName: alb` and add ALB-specific annotations (`scheme: internet-facing`, `target-type: ip`, `group.name`).
>
> **Step 4:** Deploy the updated Ingress. A new ALB is created in seconds. We verified health and switched DNS to the new ALB.
>
> **Step 5:** Remove ingress-nginx — `helm uninstall ingress-nginx`.
>
> The result: one fewer infrastructure component to maintain, direct pod routing, native WAF integration capability, and about 20% cost reduction from eliminating the NLB and nginx pods."

---

### Q7. How do you connect to different environments from your laptop?

**📋 In Our Project:**

```bash
aws eks update-kubeconfig --region us-east-1 --name pharma-dev-cluster
aws eks update-kubeconfig --region us-east-1 --name pharma-qa-cluster
kubectl config get-contexts
kubectl config use-context <context-name>
```

**🎤 Interview Answer:**

> "Each environment is a separate EKS cluster, so we maintain multiple kubeconfig contexts. On my laptop, `~/.kube/config` has three contexts — one per environment.
>
> For multi-account setups, I use named AWS profiles in `~/.aws/config`:
> ```bash
> # Set account
> export AWS_PROFILE=zenpharma-prod
>
> # Pull the kubeconfig for that account's cluster
> aws eks update-kubeconfig --region us-east-1 --name pharma-prod-cluster --alias prod
>
> # Switch context
> kubectl config use-context prod
> ```
>
> In practice I use `kubectx` and `kubens` for fast switching. I also keep a terminal prompt that always shows the active context — a wrong-context kubectl apply to prod is something you only do once. We also restrict production access: only the on-call engineer and tech leads have production kubeconfig access. Developers only get dev and QA.
>
> For audit purposes, every `kubectl exec` and `kubectl apply` to production goes through a bastion host with session logging enabled — so we have a full trail of who ran what command."

---

### Q8. EKS Upgrade — how would you do it?

**📋 In Our Project:**

In `infra/modules/eks/main.tf`, the EKS module version is `~> 21.0` with `kubernetes_version = var.kubernetes_version`. To upgrade, change the version in `terraform.tfvars`, push, plan, approve, apply. Node group upgrades happen next via managed node group rolling replacement.

**🎤 Interview Answer:**

> "EKS upgrades are one of those operational tasks where the process matters more than the individual steps. EKS only supports one minor version at a time, so 1.33 → 1.34 → 1.35, never 1.33 → 1.35.
>
> Our process:
>
> **1. Read the upgrade notes.** AWS publishes a detailed changelog for every EKS version. Check for deprecated API versions — this is the most common source of breakage. We run `kubectl convert` or `pluto` to scan our manifests for deprecated APIs before touching anything.
>
> **2. Upgrade dev first.** Update `kubernetes_version` in the dev Terraform vars, push, approve, apply. This takes about 15 minutes. The control plane upgrades first; nodes continue running the old Kubernetes version during this time.
>
> **3. Upgrade managed node groups.** We update the node group's `ami_release_version` or use `most_recent = true`. EKS does a rolling replacement — it cordons old nodes, drains pods (respecting PodDisruptionBudgets), launches new nodes, and schedules pods onto them.
>
> **4. Upgrade EKS add-ons.** CoreDNS, kube-proxy, VPC CNI, and EKS Pod Identity Agent must be on versions compatible with the new control plane. We update these in Terraform.
>
> **5. Validate in dev.** Run the full test suite, check all pods, check ArgoCD health. After 48 hours clean, repeat the same process for QA, then prod.
>
> We had one incident where upgrading from 1.31 to 1.32 broke a CronJob manifest that used the `batch/v1beta1` API — deprecated in 1.25 and removed in 1.32. Running `pluto` beforehand would have caught it. Since then, `pluto` is a mandatory step before any cluster upgrade."

---

### Q9. How do you rotate / update the database password?

**📋 In Our Project:**

Change `DEV_DB_PASSWORD` in GitHub Secrets → next Terraform apply updates both RDS and Secrets Manager → ESO re-syncs the Kubernetes Secret → `kubectl rollout restart deployment -n dev` picks up the new secret.

**🎤 Interview Answer:**

> "We treat credential rotation as a zero-downtime operation. Here's the flow:
>
> **The right way — via pipeline:**
> 1. Update the `DEV_DB_PASSWORD` secret in the GitHub Environment settings.
> 2. Trigger a Terraform apply — it updates RDS master password and Secrets Manager in one atomic operation.
> 3. ESO detects the Secrets Manager change within its sync interval and updates the Kubernetes Secret.
> 4. We do a rolling restart: `kubectl rollout restart deployment -n dev`. Kubernetes starts new pods (which get the new credentials) before killing old ones, so there's no gap in service.
>
> **Emergency rotation (manual):**
> If we discover a credential leak, we rotate immediately out-of-band using AWS CLI, then follow the same restart process.
>
> **What we're working toward:** AWS Secrets Manager has native rotation support using Lambda functions. You can set a rotation schedule (e.g., every 30 days) and Secrets Manager handles both the RDS password change and the Secrets Manager update automatically. Combined with ESO's automatic sync, pods would pick up rotated credentials without any manual intervention. That's the end state we're building toward."

---

### Q10. How do you connect the project to a domain name?

**📋 In Our Project:**

Not currently configured — the project uses the ALB's auto-generated DNS name.

**🎤 Interview Answer:**

> "For production, the setup is:
>
> 1. **ACM Certificate** — Request a certificate for `app.zenpharma.com` in ACM. Since we use Route 53, ACM can auto-validate via DNS in about 2 minutes.
>
> 2. **ALB Listener** — Add annotations to the Ingress:
>    ```yaml
>    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...:certificate/xxx
>    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443},{"HTTP":80}]'
>    alb.ingress.kubernetes.io/ssl-redirect: '443'
>    ```
>    The ALB Controller updates the listener automatically when ArgoCD applies the Ingress.
>
> 3. **Route 53 record** — Create an Alias record pointing `app.zenpharma.com` to the ALB. Unlike CNAME, an Alias record works at the zone apex and has no additional DNS lookup cost.
>
> 4. **Helm values** — Set `ingress.host: app.zenpharma.com` in the values file so the Ingress rule matches the correct hostname.
>
> For different environments: `dev.zenpharma.com`, `qa.zenpharma.com`, `app.zenpharma.com` (prod). Each points to its own ALB in its own account."

---

## 2. CI/CD Pipeline

---

### Q11. Can you explain your CI/CD pipeline end to end?

**📋 In Our Project:**

Three separate pipelines in three repos:
- **Infra repo**: `terraform.yml` — PR → plan, push to main → plan → approval gate → apply
- **Frontend repo**: `ci-pharma-ui.yml` — push to develop → lint → test → SonarCloud → build → Docker → Trivy → push ECR → update gitops dev values → ArgoCD deploys
- **Backend repo**: per-service `ci-<svc>.yml` calling reusable `_java-build.yml` — push to develop + path filter → Maven verify → SonarCloud → Docker → Trivy → ECR → Cosign sign → update gitops dev values → ArgoCD deploys

**🎤 Interview Answer:**

> "We have a pipeline per repository, with clear separation of concerns.
>
> **Infrastructure pipeline (infra repo):** Terraform-based. A PR from any branch triggers a plan — developers can see infrastructure impact before merge. Push to main triggers plan → approval gate → apply. The approval gate is a GitHub Environment configured with required reviewers. No one can terraform apply to production without a human approving it in GitHub. We also use S3 native state locking so concurrent runs don't corrupt state.
>
> **Application pipeline (frontend/backend repos):** Every push to the `develop` branch runs the full pipeline — lint, unit tests, security scanning, Docker build, image scanning, push to ECR, then GitOps update. The key design decision was making pipelines **push-based to GitOps** rather than directly deploying to Kubernetes. The pipeline never runs `kubectl apply` — it commits an image tag to the gitops repo and ArgoCD handles the actual deployment. This separates the CI concern (build a good artifact) from the CD concern (deploy it safely).
>
> **Backend reusable workflows:** We have 8 Java microservices. Instead of copy-pasting the same 240-line pipeline 8 times, we created `_java-build.yml` — a reusable `workflow_call` that all 8 service pipelines call. One security gate change (adding Cosign signing, for example) is a single file change that applies to all services. That's the architecture of a mature CI system.
>
> **PR pipelines:** We have separate lightweight `ci-pr-<svc>.yml` workflows for pull requests — they run Maven and SonarCloud but skip Docker and ECR. Fast feedback in ~5 minutes without burning ECR storage or runner time."

---

### Q12. Can you walk me through the GitHub Actions pipeline in detail?

**📋 In Our Project:**

Backend `_java-build.yml` stages: checkout → set `sha-${GITHUB_SHA::7}` image tag → setup Java 17 Temurin → optional PostgreSQL container for integration tests → Maven verify (compile + test + JaCoCo coverage) → SonarCloud SAST → configure AWS via OIDC → ECR login → Docker build (UID/GID 1000) → Trivy scan → push to ECR → Cosign keyless sign → post step summary.

**🎤 Interview Answer:**

> "Let me walk through the backend Java pipeline stage by stage and explain the decisions:
>
> **Stage 1 — Image tag:** `sha-${GITHUB_SHA::7}` — the first 7 chars of the git commit SHA. This is immutable and traceable. You see `sha-a3f9b21` in production and you know exactly which commit is running. No `latest`, no `v1.0`, no mutable tags.
>
> **Stage 2 — Maven verify:** Compiles, runs unit and integration tests, generates JaCoCo coverage report. We require 80% line coverage as a gate — the build fails below that. This is enforced in `pom.xml`, not just reported.
>
> **Stage 3 — SonarCloud SAST:** Detects bugs, code smells, security vulnerabilities (OWASP Top 10 patterns) in source code. Coverage report from JaCoCo is passed here so SonarCloud shows real coverage on the dashboard, not estimated.
>
> **Stage 4 — OIDC authentication:** No AWS keys stored anywhere. GitHub Actions requests a short-lived JWT from GitHub's OIDC provider, exchanges it with AWS STS for temporary credentials (1-hour TTL) that can assume our `pharma-dev-github-actions-role`. If those credentials leak in a log, they're useless within an hour.
>
> **Stage 5 — Docker build:** Multi-stage build producing a JRE-only image. The container runs as UID 1000 (non-root). `--build-arg UID=1000 GID=1000` is passed at build time.
>
> **Stage 6 — Trivy:** Scans the built image for CVEs. We report HIGH and CRITICAL (non-blocking currently — we're building the baseline). Severity `CRITICAL` without a fix available is excluded from the count.
>
> **Stage 7 — Cosign keyless sign:** The image is signed using GitHub's OIDC identity. The signature goes to Rekor (Sigstore's public transparency log). Anyone can verify the image was built by our GitHub Actions workflow and not tampered with. No long-lived signing keys exist."

---

### Q13. How do you send secrets to your pipeline?

**📋 In Our Project:**

Three layers: repository secrets (`AWS_ACCOUNT_ID`, `SONAR_TOKEN`, `GITOPS_TOKEN`), GitHub Environment secrets scoped to `dev`/`qa`/`prod` (`DEV_DB_PASSWORD`, `DEV_JWT_SECRET`), and AWS Secrets Manager at runtime (DB credentials accessed by pods via ESO, never in CI).

**🎤 Interview Answer:**

> "We have three tiers of secrets, each appropriate for a different scope:
>
> **Tier 1 — Build-time secrets (GitHub repository secrets):** `SONAR_TOKEN` for SonarCloud, `GITOPS_TOKEN` for the bot user that commits to the gitops repo, `AWS_ACCOUNT_ID` to construct the OIDC role ARN. These are available to all pipelines in the repo.
>
> **Tier 2 — Environment-scoped secrets (GitHub Environments):** `DEV_DB_PASSWORD` and `DEV_JWT_SECRET` are stored in the `dev` GitHub Environment, not at the repo level. This means only pipelines that target the `dev` environment (and pass the approval gate) can read them. QA has its own values. Prod has its own. A developer accidentally targeting the wrong environment simply cannot access prod secrets.
>
> **Tier 3 — Runtime secrets (AWS Secrets Manager):** Pods never get their credentials from CI at all. The DB password and JWT secret are stored in Secrets Manager. External Secrets Operator reads them at runtime and creates Kubernetes Secrets. This is the most secure tier — credentials never appear in CI logs, never in GitHub, never on disk. They live only in Secrets Manager and transiently in the pod's memory.
>
> The principle: push secrets as late as possible in the chain. The later a secret is injected, the smaller the attack surface."

---

### Q14. What is your branching strategy?

**📋 In Our Project:**

All repos: `feature/*` → `develop` → PR → `main` (protected). CI triggers on `develop` push and `release/**`. Backend CI has path filters so only the changed service's pipeline runs.

**🎤 Interview Answer:**

> "We have different branching strategies for the infra repo vs. application repos, which reflects their different risk profiles.
>
> **Application repos (frontend, backend):**
> ```
> feature/add-drug-search  ──┐
>                            ▼
>                        develop  ← CI runs here: test, build, push image, deploy to DEV
>                            │
>                      Pull Request  ← code review required
>                            │
>                          main  ← protected; no direct push
>                            │
>                     release/2.1.0  ← cut for stable releases (also triggers CI)
> ```
>
> Developers work on `feature/*` or `fix/*` branches. They open a PR to `develop`. The PR pipeline runs in about 5 minutes (Maven + SonarCloud, no Docker). On merge to `develop`, the full pipeline runs — builds the image, deploys to dev automatically. When QA sign-off is complete, we merge `develop` to `main` via PR.
>
> **Infra repo:**
> ```
> feature/add-karpenter  ──┐
>                          ▼
>                         main  ← protected; all changes go through PR
>                                 Every PR triggers terraform plan (visible in PR comments)
>                                 Merge triggers: plan → approval gate → apply
> ```
>
> The infra repo has no `develop` branch. Infrastructure changes go directly to `main` via PR because there's no meaningful 'dev' of infrastructure — you test it by running `terraform plan`, not by merging to an intermediate branch.
>
> **Backend path filtering:** The backend repo has all 8 services. The CI workflow for `api-gateway` only triggers when files under `api-gateway/**` change. A commit to `auth-service` does not rebuild the api-gateway image. This keeps CI fast and reduces unnecessary ECR pushes."

---

### Q15. Can you migrate from GitHub-hosted runners to self-hosted runners?

**📋 In Our Project:**

Currently using GitHub-hosted runners (`ubuntu-latest`). Self-hosted migration would require replacing `runs-on` labels, attaching IAM role to the runner EC2 instead of using OIDC, and pre-installing Java, Docker, Terraform.

**🎤 Interview Answer:**

> "Yes, and in a mature setup this is actually recommended for larger teams. The migration is straightforward once you understand what changes and what doesn't.
>
> **What changes:**
>
> 1. `runs-on: ubuntu-latest` → `runs-on: [self-hosted, linux, pharma-runner]`. That's the only YAML change.
>
> 2. **AWS credentials:** OIDC still works from self-hosted runners. Alternatively, you can attach an EC2 instance profile to the runner — then you don't need `configure-aws-credentials` at all; tools pick up credentials from instance metadata automatically.
>
> 3. **Tool installation:** GitHub-hosted runners have everything pre-installed. Self-hosted runners need Docker, Java, Maven, Terraform, kubectl, helm pre-installed and maintained.
>
> **Why we'd do it:**
>
> - **Speed:** Self-hosted runners have persistent Maven and npm caches. Our Java builds dropped from 14 minutes to 6 minutes after caching Maven's `.m2` directory.
> - **Private network access:** A runner inside the VPC can reach the EKS API server, RDS, and internal services directly — useful for integration test stages.
> - **Cost:** At our pipeline volume, GitHub-hosted runners at 2000 minutes/month (free tier) would be exceeded quickly. Self-hosted runners on a reserved EC2 cost a fraction.
>
> **Tradeoffs:** You own the runner's security patches, availability, and scaling. For burstable workloads, you can use runner autoscaling with the `actions-runner-controller` Kubernetes operator — it scales runner pods on-demand inside your EKS cluster."

---

## 3. Security

---

### Q16. How is security enforced in your project?

**📋 In Our Project:**

Build time: SonarCloud SAST, Trivy image scan, npm audit. Deploy time: OIDC federation (no static keys), GitHub Environment approval gates. Runtime: Pod security context (`runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `capabilities: drop: [ALL]`), ESO for secrets, private RDS subnet, ECR scan on push, Cosign image signing.

**🎤 Interview Answer:**

> "Security is enforced in five layers:
>
> **Layer 1 — Code quality gates:** SonarCloud SAST runs on every PR and push — it detects SQL injection patterns, insecure deserialization, hardcoded credentials, and OWASP Top 10 in source code before an image is even built. npm audit catches vulnerable frontend dependencies. The pipeline fails if SonarCloud reports a quality gate failure.
>
> **Layer 2 — Supply chain security:** We use multi-stage Docker builds so build tools (Maven, npm) never ship to production. Trivy scans the final image before it's pushed to ECR. Every pushed image is signed with Cosign using GitHub OIDC — the signing identity is tied to the workflow run, not a human. This means you can verify whether a given ECR image was produced by your CI pipeline or tampered with post-push.
>
> **Layer 3 — No static credentials:** GitHub Actions authenticates to AWS using OIDC. No `AWS_ACCESS_KEY_ID` in any repository. Temporary STS credentials with 1-hour TTL. IAM role policies are scoped by condition to only the specific repository and branch.
>
> **Layer 4 — Runtime pod hardening:** Every container in the cluster runs as non-root (UID 1000), with `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, and all Linux capabilities dropped. The pod security context enforces this at the Kubernetes admission level.
>
> **Layer 5 — Secrets management:** No secrets in git, no secrets in ConfigMaps, no secrets in environment variables at build time. All runtime secrets come from AWS Secrets Manager via External Secrets Operator. The pods get credentials injected at startup and the credentials are rotatable without redeployment."

---

### Q17. What is OIDC and why don't you use static AWS keys?

**📋 In Our Project:**

GitHub Actions OIDC provider is registered in AWS IAM. When a workflow runs, it gets a short-lived JWT from GitHub, exchanges it with AWS STS for 1-hour credentials. IAM role has a trust policy with `StringLike` condition on the GitHub repo name and branch. OIDC role ARN is constructed from `AWS_ACCOUNT_ID` secret (which is not a credential — just an account number).

**🎤 Interview Answer:**

> "Static keys are a liability that compounds over time — they don't expire, they get copy-pasted into `.env` files, they show up in CI logs when someone accidentally prints them, and rotating them requires touching every system that uses them.
>
> OIDC solves all of that. Here's the exact flow:
>
> When a GitHub Actions job starts, GitHub's OIDC provider issues a JWT to the runner. This JWT contains claims like `repo:zenpharma/frontend`, `ref:refs/heads/develop`, `environment:dev`. The runner sends this JWT to AWS STS. AWS validates it against the registered OIDC provider and checks the IAM trust policy:
>
> ```json
> {
>   'StringLike': {
>     'token.actions.githubusercontent.com:sub': 'repo:zenpharma/*:ref:refs/heads/main'
>   }
> }
> ```
>
> If the JWT matches the trust policy, STS issues temporary credentials valid for 1 hour. The IAM role only allows ECR push to our specific repositories — least privilege.
>
> Three reasons we chose this over static keys:
>
> 1. **No rotation needed** — credentials expire in an hour automatically
> 2. **No storage needed** — nothing to store in GitHub Secrets that could be leaked
> 3. **Audit trail** — every AWS API call shows the GitHub Actions job context in CloudTrail, so you know which workflow, which run, and which commit triggered each action"

---

### Q18. What is your image cleanup process?

**📋 In Our Project:**

ECR lifecycle policy (set by Terraform) retains the last 10 images per repository. Older images are expired automatically by AWS. Before `terraform destroy`, images must be manually batch-deleted using `aws ecr batch-delete-image` because Terraform cannot delete a non-empty ECR repo.

**🎤 Interview Answer:**

> "We have two cleanup mechanisms: automatic and pre-destroy.
>
> **Automatic cleanup via ECR lifecycle policy:** Terraform sets a lifecycle policy on every ECR repository. The policy keeps the last 10 images regardless of tag. AWS enforces this — when the 11th image is pushed, the oldest is deleted automatically. Zero maintenance required.
>
> ```json
> { 'countType': 'imageCountMoreThan', 'countNumber': 10, 'action': { 'type': 'expire' } }
> ```
>
> In production you'd tune this more carefully — for example, keep the last 30 images, or keep images with the `latest` tag regardless of count, or keep images younger than 60 days.
>
> **Pre-destroy cleanup:** This is a lesson we learned the hard way. When we ran `terraform destroy` for the first time, it failed because ECR repositories contained images and Terraform won't force-delete them. You have to batch-delete all images first:
>
> ```bash
> aws ecr batch-delete-image --repository-name pharma-ui \
>   --image-ids $(aws ecr list-images --repository-name pharma-ui --query 'imageIds' --output json)
> ```
>
> We've now scripted this in our destroy runbook so we never hit that failure again. Alternatively, you can add `force_delete = true` to the Terraform ECR resource — but we prefer explicit cleanup over silent force-deletion."

---

## 4. Kubernetes, Helm & ArgoCD

---

### Q19. Can you walk me through your Deployment manifest?

**📋 In Our Project:**

Helm template at `gitops/helm-charts/templates/deployment.yaml`. Key sections: `serviceAccountName` for IRSA, `podSecurityContext` with `runAsNonRoot: true`, containers with `envFrom` mounting ConfigMap and Secrets, liveness/readiness probes with separate paths and delays, resource `requests` and `limits`, `readOnlyRootFilesystem: true`, `volumeMounts` for Nginx writable directories (emptyDir).

**🎤 Interview Answer:**

> "Let me walk through the most important parts and explain the reasoning:
>
> **ServiceAccount with IRSA:** The pod's ServiceAccount has an annotation pointing to an IAM role ARN. This is IRSA — IAM Roles for Service Accounts. The pod gets AWS credentials injected by the pod identity webhook, which means only pods with this specific ServiceAccount can call our specific AWS APIs. No node-level IAM role grants these permissions.
>
> **Security context:**
> ```yaml
> podSecurityContext:
>   runAsNonRoot: true
>   runAsUser: 1000
> securityContext:
>   readOnlyRootFilesystem: true
>   allowPrivilegeEscalation: false
>   capabilities:
>     drop: [ALL]
> ```
> This means: even if an attacker gets code execution inside the container, they can't write to disk, can't gain new privileges, can't use Linux capabilities like `NET_ADMIN` or `SYS_PTRACE`. For Nginx specifically, we mount `emptyDir` volumes at `/tmp`, `/var/cache/nginx`, and `/var/run` because Nginx needs writable directories even with `readOnlyRootFilesystem: true`.
>
> **Probes:**
> - **Readiness probe** at `/` with `initialDelaySeconds: 5` — pod only receives traffic when ready
> - **Liveness probe** at `/` with `initialDelaySeconds: 10` — restarts the pod if it becomes unresponsive
>
> These are separate intentionally. During startup, the pod isn't ready yet (correct — don't send traffic) but it's still alive (correct — don't restart it). Once started, if it hangs, the liveness probe fires a restart.
>
> **Resource requests and limits:** `requests` tell the scheduler how much capacity to reserve. `limits` enforce a hard ceiling. We keep limits 4x the requests (50m/200m CPU, 64Mi/128Mi memory) so pods can burst but can't starve other pods on the node."

---

### Q20. Why do you use Helm? What is the `_helpers.tpl` file?

**📋 In Our Project:**

Single shared Helm chart at `gitops/helm-charts/` with templates for Deployment, Service, Ingress, ServiceAccount, ConfigMap, HPA. Per-service per-environment values in `envs/<env>/values-<svc>.yaml`. `_helpers.tpl` defines reusable named templates `pharma-service.fullname`, `pharma-service.labels`, `pharma-service.selectorLabels`.

**🎤 Interview Answer:**

> "Without Helm, we'd have 9 services × 6 Kubernetes resources × 3 environments = 162 YAML files to maintain. Change the liveness probe timeout? Edit 27 files. That's not manageable.
>
> Helm gives us one chart, N values files. Every service gets a Deployment, Service, Ingress, ServiceAccount, ConfigMap, and HPA from the same templates. The values file only contains what's different: the image, replica count, resource sizes, probe paths, and environment-specific config.
>
> ```
> helm-charts/           ← one chart, shared by all 9 services
>   Chart.yaml
>   values.yaml          ← defaults
>   templates/
>     deployment.yaml    ← Go template
>     service.yaml
>     ingress.yaml
>     hpa.yaml
>     _helpers.tpl       ← NOT rendered as K8s manifest
>
> envs/dev/
>   values-pharma-ui.yaml     ← overrides for pharma-ui in dev
>   values-api-gateway.yaml   ← overrides for api-gateway in dev
> envs/qa/
>   values-pharma-ui.yaml
>   ...
> ```
>
> **`_helpers.tpl`** is a Helm convention — files starting with `_` are not rendered as Kubernetes manifests. Instead, they define reusable Go template functions that all other templates can call:
>
> ```
> {{- define 'pharma-service.fullname' -}}
>   {{ .Values.fullnameOverride | default (printf '%s' .Release.Name) }}
> {{- end -}}
>
> {{- define 'pharma-service.labels' -}}
>   app.kubernetes.io/name: {{ include 'pharma-service.fullname' . }}
>   app.kubernetes.io/instance: {{ .Release.Name }}
> {{- end -}}
> ```
>
> Every template calls `include 'pharma-service.labels'` rather than repeating the label block. If we need to add a label (e.g., `team: platform`), we change `_helpers.tpl` once and every resource in every service gets it."

---

### Q21. Can you explain the ArgoCD Application file?

**📋 In Our Project:**

`argocd/apps/dev/pharma-ui-app.yaml` — kind `Application`, namespace `argocd`, source pointing to gitops repo at `helm-charts/` path with `../envs/dev/values-pharma-ui.yaml`, destination namespace `dev`, automated sync with `prune: true` and `selfHeal: true`, retry with exponential backoff, `revisionHistoryLimit: 10`.

**🎤 Interview Answer:**

> "The ArgoCD Application is the bridge between the gitops repo and the Kubernetes cluster. It has four key sections:
>
> **Source** — where to get the desired state:
> ```yaml
> source:
>   repoURL: https://github.com/zenpharma/gitops.git
>   targetRevision: HEAD
>   path: helm-charts
>   helm:
>     valueFiles:
>       - ../envs/dev/values-pharma-ui.yaml
> ```
> When someone merges a commit that changes `values-pharma-ui.yaml`, ArgoCD detects the change at the next poll cycle (every 3 minutes) and computes a diff.
>
> **Destination** — where to deploy:
> ```yaml
> destination:
>   server: https://kubernetes.default.svc   # this cluster
>   namespace: dev
> ```
>
> **Sync policy** — how to apply changes:
> ```yaml
> syncPolicy:
>   automated:
>     prune: true      # delete K8s resources that are removed from git
>     selfHeal: true   # revert manual kubectl edits — git is the truth
> ```
> `selfHeal: true` is crucial for GitOps discipline. If a developer runs `kubectl edit deployment pharma-ui -n dev` and changes the replica count, ArgoCD will detect the drift and revert it within 3 minutes. This enforces that git is the only way to make changes.
>
> **Retry with backoff:** If a sync fails (e.g., image pull error), ArgoCD retries with exponential backoff — 5s, 10s, 20s, 40s — up to 5 times and max 3-minute wait. This prevents hammering the cluster on a transient failure."

---

### Q22. Difference between ArgoCD Project and Application?

**📋 In Our Project:**

One `AppProject` named `pharma` allows the gitops repo as source and `dev`/`qa`/`prod` namespaces as destinations. Multiple `Application` resources (pharma-ui-dev, api-gateway-dev, pharma-ui-qa, etc.) reference this project and define specific source paths and destination namespaces.

**🎤 Interview Answer:**

> "The simplest way to remember it: the **Project is the policy**, the **Application is the deployment unit**.
>
> | | AppProject | Application |
> |---|---|---|
> | Role | Governance / RBAC | Deployment config |
> | Defines | What sources are allowed, what destinations are allowed, what resource types can be created | Which chart to deploy, which values file, which namespace |
> | Count in our system | 1 (the `pharma` project) | 1 per service per environment |
>
> The AppProject answers: 'Is this Application allowed to deploy from this repo to this namespace using these resource types?' Without a Project, an Application could theoretically deploy anything from anywhere to anywhere.
>
> In our setup, the `pharma` project:
> - Allows only our gitops repo as a source (can't use someone else's public chart)
> - Allows only `dev`, `qa`, `prod` namespaces as destinations (can't deploy into `kube-system`)
> - Allows all resource kinds (we trust our team — in stricter setups you'd restrict this)
>
> An Application references the project it belongs to. ArgoCD enforces the project's policy at sync time. If you try to create an Application that deploys to `kube-system` and the project doesn't allow it, ArgoCD rejects the sync."

---

### Q23. What is your image promotion policy?

**📋 In Our Project:**

DEV: automatic on push to develop (CI commits to gitops directly). QA: manual `workflow_dispatch` on `promote-qa-pharma-ui.yml` (frontend) or `promote-qa.yml` (backend with service dropdown) — reads image tag from `envs/dev/values-<svc>.yaml`, creates a branch, opens a PR in gitops repo, ArgoCD auto-syncs after PR merge. PROD: same pattern via `promote-prod.yml`.

**🎤 Interview Answer:**

> "The core principle: **the same image artifact, never rebuilt, promoted through environments by updating a pointer.**
>
> The image built from commit `sha-a3f9b21` on develop is exactly the image that goes to QA and production. We don't rebuild from source in QA 'to be safe' — that would mean QA tested a different artifact than what prod runs.
>
> **DEV** — automatic:
> The CI pipeline builds `sha-<7chars>`, pushes it to ECR, then commits a one-line change to `envs/dev/values-pharma-ui.yaml`. ArgoCD detects the gitops commit and deploys in seconds. No human involvement.
>
> **QA** — manual `workflow_dispatch`:
> A developer goes to GitHub Actions → Promote to QA → Run workflow. The workflow reads whatever image tag is currently in `envs/dev/values-pharma-ui.yaml` (the image that passed dev testing) and opens a PR in the gitops repo to update `envs/qa/values-pharma-ui.yaml`. A QA lead or DevOps engineer reviews the PR — it shows exactly which image is being promoted and what config is changing. Merge → ArgoCD deploys to QA automatically.
>
> **PROD** — same pattern, stricter gate:
> Same `workflow_dispatch` workflow, reading the QA values file this time. The gitops PR to update prod values requires 2 approvers. After merge, ArgoCD syncs prod (we can configure this as manual sync for prod, requiring an ArgoCD operator to click sync after the PR merges).
>
> The promotion trail is fully auditable in git: every image promotion is a commit with author, timestamp, and what image went where. No verbal sign-offs, no spreadsheets."

---

### Q24. How do you add a new microservice to the environment?

**📋 In Our Project:**

1. Add service directory to backend repo, 2. Add ECR repo to infra `main.tf`, 3. Create `ci-new-service.yml` calling `_java-build.yml`, 4. Add `envs/dev/values-new-service.yaml` in gitops, 5. Create `argocd/apps/dev/new-service-app.yaml`, 6. Merge → ArgoCD deploys.

**🎤 Interview Answer:**

> "This is one of the places where the architecture pays off. Because we have a shared Helm chart and reusable CI workflow, adding a new service is mostly configuration, not new code.
>
> Here are the exact steps:
>
> **1. Backend repo** — create the service directory with its `Dockerfile` and `pom.xml`. Create `ci-new-service.yml`:
> ```yaml
> jobs:
>   build:
>     uses: ./.github/workflows/_java-build.yml
>     with:
>       service-name: new-service
>       service-dir: new-service
>       ecr-repository: new-service
>     secrets: inherit
> ```
> That's the entire CI pipeline — 10 lines, inheriting the full security gates.
>
> **2. Infra repo** — add `'new-service'` to the `repositories` list in the ECR module. Open a PR → plan → approve → apply. ECR repo is created.
>
> **3. GitOps repo** — create `envs/dev/values-new-service.yaml` with the image repository URL, resource sizes, and probe paths specific to this service. Create `argocd/apps/dev/new-service-app.yaml` pointing to the shared chart with this values file.
>
> **4. Commit and merge.** ArgoCD detects the new Application manifest and creates the deployment. The CI pipeline builds and pushes the first image. ArgoCD deploys it.
>
> Total time from 'we need a new service' to 'it's running in dev': a few hours for the service skeleton, minutes for the config. No infrastructure tickets, no manual kubectl apply, no 'please ask DevOps to add the ECR repo.'"

---

### Q25. If you want to add a new cluster to ArgoCD, what are the steps and files?

**📋 In Our Project:**

Currently ArgoCD manages only the cluster it's installed on (using `https://kubernetes.default.svc`). Adding a remote cluster would require registering it via ArgoCD CLI or creating a Secret in the argocd namespace with the cluster credentials.

**🎤 Interview Answer:**

> "This is the multi-cluster ArgoCD pattern, which is how you manage QA and prod clusters from a single ArgoCD instance running in dev (or a dedicated management cluster).
>
> **Step 1 — Register the cluster with ArgoCD CLI:**
> ```bash
> # Assumes your kubeconfig has a context for the prod cluster
> argocd cluster add prod-context --name pharma-prod
> ```
> This command does two things: creates a ServiceAccount and ClusterRoleBinding in the target cluster (`argocd` namespace) and stores the cluster connection details as a Secret in the ArgoCD namespace.
>
> **Step 2 — The Secret ArgoCD creates (you can also manage this in git as a Sealed Secret):**
> ```yaml
> # argocd/clusters/pharma-prod-cluster.yaml (Sealed Secret or SOPS encrypted)
> apiVersion: v1
> kind: Secret
> metadata:
>   name: pharma-prod-cluster
>   namespace: argocd
>   labels:
>     argocd.argoproj.io/secret-type: cluster
> type: Opaque
> stringData:
>   name: pharma-prod
>   server: https://prod-eks-api.us-east-1.eks.amazonaws.com
>   config: |
>     { 'bearerToken': '...', 'tlsClientConfig': { 'caData': '...' } }
> ```
>
> **Step 3 — Update the AppProject** to allow the new destination:
> ```yaml
> destinations:
>   - server: https://kubernetes.default.svc   # dev cluster (existing)
>     namespace: '*'
>   - server: https://prod-eks-api...amazonaws.com   # prod cluster (new)
>     namespace: prod
> ```
>
> **Step 4 — Create Application manifests** for the new cluster:
> ```yaml
> # argocd/apps/prod/pharma-ui-app.yaml
> spec:
>   destination:
>     server: https://prod-eks-api...amazonaws.com
>     namespace: prod
> ```
>
> **Files to create:**
> - `argocd/clusters/pharma-prod-cluster.yaml` — cluster Secret (encrypted in git)
> - `argocd/projects/pharma-project.yaml` — update to add new destination
> - `argocd/apps/prod/<service>-app.yaml` — one Application per service for prod
>
> The best practice is to store the cluster Secret encrypted using Sealed Secrets or SOPS so it can live safely in the gitops repo. ArgoCD itself is then the source of truth for all cluster registrations."

---

## 5. Docker & Image Management

---

### Q26. Can you explain multi-stage Dockerfile? What is the advantage?

**📋 In Our Project:**

Frontend: Stage 1 `node:22-alpine` builder (npm ci + npm run build), Stage 2 `nginx:1.25-alpine` (copies `/app/build`). Backend: single stage `eclipse-temurin:17-jre` (JRE only, not JDK; non-root user).

**🎤 Interview Answer:**

> "A multi-stage build uses multiple `FROM` statements. Each stage is independent and can copy artifacts from previous stages. The final image contains only what the last `FROM` stage has — everything else is discarded.
>
> **Frontend example:**
> ```dockerfile
> # Stage 1: builder — has Node.js, npm, all dev dependencies (~600MB)
> FROM node:22-alpine AS builder
> WORKDIR /app
> COPY package*.json ./
> RUN npm ci          # installs ALL dependencies including devDependencies
> COPY src ./src
> RUN npm run build   # produces /app/build/ — the static files
>
> # Stage 2: runtime — only Nginx (~25MB)
> FROM nginx:1.25-alpine
> COPY --from=builder /app/build /usr/share/nginx/html
> COPY nginx.conf /etc/nginx/conf.d/default.conf
> ```
>
> The final image is ~25MB instead of ~600MB. The Node.js runtime, npm, all the `devDependencies`, and the source files are not in the image you push to ECR and run in production.
>
> **Three concrete benefits:**
>
> 1. **Security surface:** A smaller image has fewer packages, fewer CVEs, fewer attack vectors. Trivy finds fewer findings in a 25MB Nginx image than a 600MB Node image.
>
> 2. **Pull speed:** Pods start faster because there's less to download. At scale, pulling a 25MB image vs. 600MB across 50 pods is a significant difference in startup time.
>
> 3. **No build tools in production:** An attacker who gets code execution in the Nginx container has no npm, no compiler, no package manager to pivot with. Build tools are weapons; they don't belong in a production runtime."

---

## 6. Environment & Secret Management

---

### Q27. What is External Secrets Operator and how does it work?

**📋 In Our Project:**

ESO controller runs in `external-secrets` namespace. `ClusterSecretStore` references AWS Secrets Manager using an IRSA role. `ExternalSecret` resources in each namespace define the mapping between Secrets Manager keys and Kubernetes Secret keys. ESO reconciles on schedule and creates/updates native Kubernetes Secrets.

**🎤 Interview Answer:**

> "External Secrets Operator solves a specific problem: Kubernetes Secrets are just base64-encoded, not encrypted, and storing them in git (even encrypted) adds complexity. But pods need their credentials as Kubernetes Secrets.
>
> ESO bridges the gap: it pulls secrets from external systems (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) and creates native Kubernetes Secrets automatically.
>
> Our setup:
>
> ```yaml
> # ClusterSecretStore — registered once, references Secrets Manager via IRSA
> apiVersion: external-secrets.io/v1beta1
> kind: ClusterSecretStore
> metadata:
>   name: aws-secretsmanager
> spec:
>   provider:
>     aws:
>       service: SecretsManager
>       region: us-east-1
>       auth:
>         jwt:   # IRSA — pod identity, no static keys
>           serviceAccountRef: { name: external-secrets, namespace: external-secrets }
>
> ---
> # ExternalSecret — one per app, in the app namespace
> apiVersion: external-secrets.io/v1beta1
> kind: ExternalSecret
> metadata:
>   name: pharma-dev-db-secret
>   namespace: dev
> spec:
>   secretStoreRef: { name: aws-secretsmanager, kind: ClusterSecretStore }
>   refreshInterval: 1h
>   target:
>     name: pharma-dev-db-secret  # Kubernetes Secret name
>   data:
>     - secretKey: SPRING_DATASOURCE_PASSWORD
>       remoteRef: { key: pharma-dev-db-secret, property: password }
> ```
>
> When ESO syncs, it creates a real `kubernetes.io/Opaque` Secret named `pharma-dev-db-secret` in the `dev` namespace. The Deployment's `envFrom` picks it up. If the Secrets Manager value changes, ESO re-syncs within the `refreshInterval` and updates the Secret. A rolling restart picks up the new value.
>
> The reason we chose ESO over other patterns (like Vault Agent Sidecar or baking secrets into ConfigMaps): it's the cleanest Kubernetes-native pattern. Pods use standard `envFrom` — they don't know or care where the Secret came from."

---

## 7. Troubleshooting

---

### Q28. One microservice is not working. How do you diagnose it?

**📋 In Our Project:**

`kubectl get pods -n dev` → check status. `kubectl logs <pod> -n dev` / `--previous` for crash logs. `kubectl describe pod` for Events. `kubectl get externalsecret -n dev` for credential issues. `kubectl run debug --rm -it --image=postgres:15-alpine` for DB connectivity.

**🎤 Interview Answer:**

> "I follow a standard outer-to-inner diagnostic flow. You start at the widest observable layer and narrow down.
>
> **1. Is the pod even running?**
> ```bash
> kubectl get pods -n dev
> ```
> Status tells you a lot: `Pending` = scheduling issue or resource pressure; `ImagePullBackOff` = wrong image tag or ECR permission issue; `CrashLoopBackOff` = process crashes on startup; `OOMKilled` = memory limit too low; `Running` + `0/1 Ready` = probe failing.
>
> **2. What does the process say?**
> ```bash
> kubectl logs <pod> -n dev --tail=100
> kubectl logs <pod> -n dev --previous   # if it crashed
> ```
> For Spring Boot, you'll see the full stack trace here. Common causes: missing environment variable, DB connection refused, port already in use.
>
> **3. What do the Kubernetes events say?**
> ```bash
> kubectl describe pod <pod> -n dev
> ```
> The Events section shows: image pull failures, probe failures, OOM kills, node evictions, volume mount errors.
>
> **4. Is ArgoCD showing the app as healthy?**
> Check ArgoCD UI or `kubectl get applications -n argocd`. If ArgoCD shows `OutOfSync` or `Degraded`, there may be a Helm rendering error or a CRD conflict.
>
> **5. Are secrets synced?**
> ```bash
> kubectl get externalsecret -n dev
> ```
> If `STATUS` shows `SecretSyncedError`, the pod is starting without DB credentials — immediate symptom is a Spring Boot startup failure with 'datasource URL not configured'.
>
> **6. Can the pod reach the database?**
> ```bash
> kubectl run debug --rm -it --image=postgres:15-alpine -n dev -- \
>   psql -h <RDS_ENDPOINT> -U pharmaadmin -d pharmadb
> ```
>
> In most cases, the issue is found by step 2 or 3. I've rarely needed to go past step 5."

---

### Q29. A user cannot access the login page. How do you troubleshoot?

**📋 In Our Project:**

Work outside-in: `curl <ALB-URL>` → check ALB target health in AWS Console → `kubectl get pods -n dev` → `kubectl logs <pharma-ui-pod>` → `kubectl get ingress -n dev` → test `/api/auth/login` directly → check auth-service logs → check ESO for JWT secret.

**🎤 Interview Answer:**

> "This is a classic end-to-end failure — could be anywhere in the chain: DNS, ALB, pod, API, database. I work outside-in.
>
> **Layer 1 — Does the URL resolve and respond?**
> ```bash
> curl -v https://app.zenpharma.com/
> ```
> If DNS fails: Route 53 or propagation issue. If connection refused: ALB not up or listener not configured. If 502/503: ALB health check failing.
>
> **Layer 2 — Is the pharma-ui pod healthy?**
> ```bash
> kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui
> # Check READY column — should be 1/1
> ```
> Check ALB Target Group in AWS Console — are targets healthy? Unhealthy means the pod is failing its `/` probe.
>
> **Layer 3 — Does the page load but login fail?**
> Open browser DevTools → Network tab → watch the login request. If it returns 404, the `/api/auth` path isn't routed correctly. If 503, api-gateway is down. If 500, auth-service or DB issue.
>
> **Layer 4 — Is api-gateway up?**
> ```bash
> kubectl get pods -n dev -l app.kubernetes.io/name=api-gateway
> kubectl logs <api-gateway-pod> -n dev --tail=50
> ```
>
> **Layer 5 — Is auth-service up and connected to DB?**
> ```bash
> kubectl logs <auth-service-pod> -n dev --tail=50
> # Look for: 'HikariPool connection failed', 'JWT secret not configured'
> ```
>
> **Layer 6 — Are secrets synced?**
> ```bash
> kubectl get externalsecret -n dev
> kubectl get secret pharma-dev-jwt-secret -n dev
> ```
> If the JWT secret isn't synced, auth-service starts but fails to issue tokens — you'd see a 500 on login with a JWT-related error.
>
> One real incident we had: the login page loaded fine, the login request returned 200, but the user was immediately redirected back to login. Root cause: the JWT secret in Secrets Manager had been rotated but the pods hadn't been restarted to pick up the new secret. `kubectl rollout restart deployment/auth-service -n dev` fixed it."

---

### Q30. What is the rollback process?

**📋 In Our Project:**

Option A: ArgoCD UI → History and Rollback → select previous revision. Option B: `git revert HEAD` in gitops repo (reverts the values file commit with the bad image tag) → merge → ArgoCD auto-syncs. Option C (emergency): `kubectl rollout undo deployment/pharma-ui -n dev` then sync gitops.

**🎤 Interview Answer:**

> "Our rollback strategy has three options, in order of preference:
>
> **Option 1 — GitOps revert (preferred, audit trail preserved):**
> ```bash
> cd gitops
> git revert HEAD   # or: git revert <commit-that-bumped-the-image-tag>
> git push
> ```
> ArgoCD detects the revert commit and deploys the previous image tag. This is the cleanest option because the rollback itself is a git commit — you can see who rolled back, when, and why (from the commit message). No commands run directly against the cluster.
>
> **Option 2 — ArgoCD History rollback (fast, for emergency):**
> ArgoCD keeps the last 10 revisions (`revisionHistoryLimit: 10`). In the ArgoCD UI, click the app → History and Rollback → click any previous revision → Rollback. ArgoCD re-applies the exact Helm rendering from that point in gitops history. This is faster than a git revert but leaves the gitops repo one commit ahead of what's running — you need to follow up with a gitops revert to sync them up.
>
> **Option 3 — kubectl rollout undo (last resort):**
> ```bash
> kubectl rollout undo deployment/pharma-ui -n dev
> ```
> This uses Kubernetes' built-in deployment revision history. It's instant but puts ArgoCD into `OutOfSync` state. ArgoCD will immediately try to revert your rollback (because gitops still has the bad image tag). You have to also update gitops or pause ArgoCD sync.
>
> **Why rollback is safe in our system:**
> - Image tags are immutable (`sha-<7chars>` — never overwritten in ECR)
> - ECR keeps the last 10 images — old image is always available
> - ArgoCD revision history gives you 10 points to roll back to
> - Database migrations are forward-only (additive) so rolling back the app doesn't require rolling back the schema"

---

## 8. Real Incidents & Operational Decisions

---

### Q31. What was a recent issue you faced?

**🎤 Interview Answer (incident story):**

> "One of our backend developers was testing an AWS SDK integration locally. They created a temporary IAM user with a test access key, configured it in `application.properties` to test locally, and accidentally committed it to the feature branch. The credentials went into git history.
>
> We caught it during code review about 30 minutes after the push. The first thing we did was rotate the credentials immediately — even though the PR was never merged, the key was visible in the branch history. We used `git filter-branch` to remove the file from history and force-pushed, then notified the security team.
>
> The post-incident action was clear: we needed prevention, not just detection. We implemented **GitLeaks** as a mandatory pipeline gate.
>
> We added a GitLeaks scan as the **first step** in both the PR check workflow and the main CI workflow:
>
> ```yaml
> - name: GitLeaks — detect secrets in code
>   uses: gitleaks/gitleaks-action@v2
>   env:
>     GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
>     GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
> ```
>
> GitLeaks scans the entire commit history of the PR for patterns matching AWS keys, private keys, JWT secrets, API tokens, and hundreds of other secret types. If it finds a match, the pipeline fails before Maven even starts — the developer gets immediate feedback.
>
> We also set up GitLeaks as a **pre-commit hook** via a `.gitleaks.toml` config so developers get the warning on their local machine before they can push at all.
>
> Since implementing it, we've caught three incidents in pre-commit — once a test fixture that included a real-looking (but fake) API key, twice where developers used real credentials in test properties. All three were caught before the push. Zero incidents in production since then."

---

### Q32. Why did you implement Karpenter when you already have HPA?

**🎤 Interview Answer:**

> "HPA and Karpenter solve completely different problems — this is a common misconception.
>
> **HPA (Horizontal Pod Autoscaler)** scales **pods**: when CPU or memory crosses a threshold, add another replica. HPA solves: 'I have more traffic, I need more instances of this service.'
>
> **Karpenter** scales **nodes**: when pods can't be scheduled because there's no available capacity on existing nodes, Karpenter provisions a new EC2 node. Karpenter solves: 'I have pods waiting to be scheduled but no node has room for them.'
>
> The two work **together**: HPA sees high CPU and adds a pod replica. The scheduler can't place the new pod because all nodes are full. Karpenter sees the pending pod, provisions a new node in 30-60 seconds, and the pod is scheduled.
>
> **Why we migrated from Cluster Autoscaler to Karpenter:**
>
> Our original setup used Cluster Autoscaler with a fixed node group of t3.small instances. During a load test, node provisioning took 4-6 minutes because Cluster Autoscaler is slow — it has to wait for AWS to provision the node, the node to join the cluster, then pods to be rescheduled.
>
> Karpenter provisioned nodes in under 60 seconds in our testing. It also made smarter choices:
>
> - It could pick any EC2 instance type available in the region, not just what was defined in the node group. When t3.small was unavailable (Spot capacity), it automatically picked t3a.small or t2.small.
> - **Consolidation:** Karpenter constantly evaluates whether pods can be packed onto fewer nodes. If we had 4 nodes at 30% utilization, Karpenter would drain 2 of them and repack the pods onto the other 2, then terminate the empty nodes. This saved us approximately 25-30% on EC2 costs in the first month.
>
> The configuration is a `NodePool` and `EC2NodeClass` CRD — Karpenter replaced our entire managed node group Terraform config with two YAML files committed to the gitops repo."

---

## 9. Version Management & Maintenance

---

### Q33. How do you make sure your versions are kept up to date (Terraform, EKS, ArgoCD, modules, etc.)?

**📋 In Our Project:**

Terraform EKS module pinned to `~> 21.0`, `kubernetes_version` in tfvars. EKS add-ons use `most_recent = true`. GitHub Actions actions pinned to major versions (`@v5`). No automated update tooling configured yet.

**🎤 Interview Answer:**

> "Version drift is a real operational risk — we've seen clusters running EKS versions that were 3 minor versions behind, and the upgrade path became painful. We handle this at multiple levels:
>
> **1. Terraform providers and modules — Dependabot:**
> We have a `.github/dependabot.yml` in the infra repo:
> ```yaml
> version: 2
> updates:
>   - package-ecosystem: terraform
>     directory: '/envs/dev'
>     schedule: { interval: weekly }
>   - package-ecosystem: github-actions
>     directory: '/'
>     schedule: { interval: weekly }
> ```
> Dependabot opens a PR every week if there's a new version of `terraform-aws-modules/eks/aws` or any GitHub Action. The PR shows what changes. A human reviews and merges. Terraform modules are pinned with `~>` which accepts minor/patch bumps but requires explicit approval for major versions.
>
> **2. EKS version — AWS Health Dashboard + release calendar:**
> AWS publishes the EKS version support calendar (each version is supported for ~14 months). We subscribe to the RSS feed and get notified 6 months before end-of-life. We have a quarterly ritual where we check the current cluster version against the support calendar and plan the next upgrade if we're within 3 months of EOS.
>
> **3. EKS add-ons — `most_recent = true`:**
> Our Terraform EKS module sets `most_recent = true` for VPC CNI, kube-proxy, and CoreDNS. On every `terraform apply`, Terraform checks and updates these to the latest version compatible with the cluster. This keeps add-ons current without manual tracking.
>
> **4. ArgoCD and Helm charts:**
> ArgoCD is installed via Helm. We pin the chart version in Terraform:
> ```hcl
> resource 'helm_release' 'argocd' {
>   chart   = 'argo-cd'
>   version = '~> 7.0'   # accepts 7.x.x, not 8.x.x
> }
> ```
> Dependabot scans Helm chart versions in Terraform files and opens a PR when 7.1.0, 7.2.0, etc. are released.
>
> **5. Container base images — Renovate:**
> Renovate Bot scans Dockerfiles for base image versions (`FROM node:22-alpine`, `FROM eclipse-temurin:17-jre`) and opens PRs when new patch releases are available. This ensures our images aren't running on a Node.js version with a 3-month-old CVE.
>
> **The discipline:** every update goes through a PR, runs the full pipeline, and is reviewed before merge. We never manually update a version directly on `main`."

---

### Q36. How would you upgrade the EKS Terraform module from `~> 21.0` to a newer major version? What do you need to check first?

**📋 In Our Project:**

The EKS module is pinned to `terraform-aws-modules/eks/aws ~> 21.0` in `infra/modules/eks/main.tf`. Terraform version is `>= 1.11`, AWS provider is `~> 5.0`. The module pin `~>` allows `21.x.x` but blocks `22.x.x` — any major version bump is a deliberate change that requires planning.

**🎤 Interview Answer:**

> "A major module version bump is not a one-liner. Every major version of `terraform-aws-modules/eks/aws` has a migration guide listing breaking changes — ignoring it and just bumping the version number is the fastest way to destroy and recreate your EKS cluster mid-apply, which is a multi-hour outage.
>
> Here is the exact checklist I follow:
>
> **Step 1 — Read the CHANGELOG and upgrade guide.**
> The module publishes an `UPGRADE-X.0.md` in the GitHub repo for every major version. This lists every variable that was renamed, removed, or changed type, and every resource that was restructured. Read it before touching any code.
>
> **Step 2 — Check Terraform minimum version.**
> Each major module version often raises the minimum required Terraform version. For example, v21 requires `>= 1.3.2`. A jump to v22+ may require `>= 1.5` or higher. Check `versions.tf` in the module source:
> ```bash
> # Check what the new module version requires
> curl -s https://raw.githubusercontent.com/terraform-aws-modules/terraform-aws-eks/v22.0.0/versions.tf
> # Look for: required_version = ">= X.Y.Z"
> ```
> If your Terraform CLI is older, upgrade Terraform first — separately from the module upgrade.
>
> **Step 3 — Check AWS provider minimum version.**
> The module pins a minimum `hashicorp/aws` provider version in its `versions.tf`. A major module bump frequently raises this:
> ```hcl
> # Our current constraint in infra/envs/dev/main.tf
> required_providers {
>   aws = { source = "hashicorp/aws", version = "~> 5.0" }
> }
> ```
> If the new module requires `>= 5.61` and we're on `5.40`, we need to bump the AWS provider version first. AWS provider minor versions are generally backwards compatible but worth testing.
>
> **Step 4 — Check Kubernetes and TLS provider versions.**
> The EKS module also uses `hashicorp/kubernetes` and `hashicorp/tls` providers. The new module version may raise those minimums too. Check the module's `versions.tf` for all provider constraints.
>
> **Step 5 — Audit variable and output signature changes.**
> This is the most common source of breakage. Between v20 and v21, for example, several node group variable names changed:
> ```hcl
> # v20 style
> eks_managed_node_group_defaults = { ... }
>
> # v21 style — same concept, different key name in some sub-fields
> ```
> Run a diff of your `.tfvars` and `main.tf` against the new module's `variables.tf`. Every variable you pass must still exist with the same type signature in the new version.
>
> **Step 6 — Run `terraform plan` and look for destroys.**
> ```bash
> # In the infra repo, update version constraint and run plan
> terraform plan -out=upgrade.tfplan
> ```
> Scan the plan output specifically for:
> - `# module.eks.aws_eks_cluster.this must be replaced` — this destroys the entire cluster. Do not proceed without understanding why.
> - `# module.eks.aws_eks_node_group.this["default"] must be replaced` — this drains and replaces all nodes.
> - `# module.eks... will be updated in-place` — safe.
>
> Resource destroys in an EKS module upgrade are usually caused by either a variable type change (Terraform re-creates when it can't migrate state) or internal module restructuring.
>
> **Step 7 — Check for `moved` blocks (state address changes).**
> New module versions often restructure internal resources and ship `moved {}` blocks to migrate state non-destructively. If the module version you're targeting includes `moved` blocks, Terraform handles the state migration automatically during apply — no manual `terraform state mv` required. If the module does NOT ship `moved` blocks but you see unexpected destroys, you may need to manually move state:
> ```bash
> terraform state mv \
>   'module.eks.aws_eks_node_group.old_address' \
>   'module.eks.aws_eks_node_group.new_address'
> ```
>
> **Step 8 — Test in dev first, wait 48 hours, then promote.**
> Always apply the module upgrade to the dev environment first. Run the full test suite, check all pods restart cleanly, check add-ons are healthy. After 48 hours of clean operation, repeat for QA, then prod.
>
> **Step 9 — Upgrade EKS add-ons after the module upgrade.**
> A new module version may support newer EKS add-on API versions. After the cluster is stable, check if CoreDNS, VPC CNI, kube-proxy, and EKS Pod Identity Agent have newer compatible versions and update them.
>
> **The rule we follow:** bump one thing at a time. Don't upgrade the Terraform module, the AWS provider, and the Terraform CLI in the same PR. If something breaks, you won't know which change caused it."

---

## 10. Advanced & AI Topics

---

### Q34. In which parts of this project can AI be applied?

**🎤 Interview Answer:**

> "There are several areas where AI adds genuine value — not hype, actual automation:
>
> **1. Intelligent log analysis:** When a pod crashes, the logs often have a clear error buried in 500 lines of Spring Boot startup output. An LLM (Claude API with a tool call to `kubectl logs`) can summarize: 'auth-service crashed because the database connection pool was exhausted — RDS max_connections is 100 and 98 connections were in use when this pod started.' That's the kind of diagnosis that takes a human 10 minutes and takes an LLM 3 seconds.
>
> **2. PR review as a CI step:** Add a GitHub Actions step that sends the diff to an LLM and posts a comment on the PR: security concerns, performance anti-patterns, suggestions for better error handling. This is a first-pass review that runs in parallel with human review, catching the obvious things before a human spends time on them.
>
> **3. Predictive capacity planning:** Feed CloudWatch pod memory/CPU metrics into a time-series model. If auth-service memory grows 8% per week and the limit is 512Mi, predict the OOM date and alert proactively.
>
> **4. Auto-generated release notes:** When the promote-to-QA workflow runs, call `git log dev-image-tag..qa-image-tag --oneline`, send it to an LLM, get a human-readable summary: 'This release includes: new drug search endpoint, bug fix for inventory sync, performance improvement in the catalog service.' Automatically posted to the PR description.
>
> **5. ChatOps for on-call:** A Slack bot that answers 'What's running in QA?' by reading the gitops values files, or 'Rollback auth-service in dev' by triggering the rollback workflow. The LLM handles intent extraction; the actual action is a well-tested automation."

---

### Q35. What is predictive analysis in the context of DevOps?

**🎤 Interview Answer:**

> "Predictive analysis in DevOps means using historical operational data to prevent failures rather than react to them.
>
> **Concrete examples from our environment:**
>
> **Pod OOM prediction:** Our Prometheus metrics (or CloudWatch) show memory usage over time. If `inventory-service` memory grows from 180Mi to 210Mi to 240Mi over 3 weeks and the limit is 512Mi, a simple linear regression predicts when it will hit the limit. Alert at 80% rather than discovering it at 100% with a crash.
>
> **Node saturation prediction:** We have 4 nodes. We know we're onboarding 2 new microservices next month. Each service runs 2 replicas requiring 256Mi. We can calculate today whether the existing nodes can absorb this load before we even write the first line of the new service's code.
>
> **Deployment risk prediction:** Teams track which commits are correlated with rollbacks. A PR that touches 20 files, has no test changes, was opened late Friday evening, and touches the auth flow is statistically higher risk than a PR that touches 2 files with corresponding tests on a Tuesday morning. A model trained on historical PR data can flag this automatically.
>
> **Dependency vulnerability prediction:** OWASP NVD publishes CVEs daily. Most CVEs are published against known library versions. You can predict which of your services will be affected by a CVE before it affects them, by monitoring NVD for your current dependency versions and alerting when a vulnerability is published that matches.
>
> The tooling: AWS CloudWatch + CloudWatch Anomaly Detection for basic pattern detection, or a full observability stack (Prometheus + Grafana) with predictive dashboards."

---

## Quick Reference — Commands

```bash
# ─── Cluster connectivity ────────────────────────────────────────────────────
aws eks update-kubeconfig --region us-east-1 --name pharma-dev-cluster
kubectl config use-context <context-name>

# ─── Diagnostics ─────────────────────────────────────────────────────────────
kubectl get pods -n dev
kubectl logs <pod> -n dev --tail=100 --previous
kubectl describe pod <pod> -n dev
kubectl get externalsecret -n dev
kubectl get applications -n argocd
kubectl top pods -n dev

# ─── Rollback ────────────────────────────────────────────────────────────────
kubectl rollout undo deployment/<name> -n dev
git revert HEAD && git push    # preferred: gitops-based rollback

# ─── Secret operations ────────────────────────────────────────────────────────
kubectl rollout restart deployment -n dev
kubectl annotate externalsecret <name> -n dev force-sync=$(date +%s) --overwrite

# ─── ECR cleanup (before terraform destroy) ───────────────────────────────────
aws ecr batch-delete-image --repository-name pharma-ui \
  --image-ids "$(aws ecr list-images --repository-name pharma-ui \
  --query 'imageIds' --output json)"

# ─── Terraform lock release ───────────────────────────────────────────────────
aws s3 rm s3://zen-pharma-terraform-state-ravdy/envs/dev/terraform.tfstate.tflock

# ─── Get ALB DNS ─────────────────────────────────────────────────────────────
kubectl get ingress pharma-ui -n dev -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# ─── DB connectivity test from inside cluster ─────────────────────────────────
kubectl run debug --rm -it --image=postgres:15-alpine -n dev -- \
  psql -h <RDS_ENDPOINT> -U pharmaadmin -d pharmadb
```

---

## Key Talking Points Summary

| Topic | What to say confidently |
|---|---|
| Multi-account | "Separate AWS accounts per env under Organizations, each env has its own OIDC role" |
| No static keys | "OIDC federation — 1-hour TTL STS credentials, tied to repo+branch conditions" |
| GitOps | "Pipeline never runs kubectl — it commits to gitops, ArgoCD handles deployment" |
| Promotion | "Same image, never rebuilt — only the pointer (values file image tag) changes" |
| Secrets | "Three tiers: GitHub Secrets (build), Environment Secrets (infra), Secrets Manager (runtime)" |
| Rollback | "Git revert is preferred — every rollback is an auditable commit" |
| HPA vs Karpenter | "HPA scales pods, Karpenter scales nodes — they work together" |
| Security | "Five layers: SAST, image scan, OIDC, pod hardening, ESO for runtime secrets" |
| GitLeaks | "First step in every pipeline — caught 3 pre-push incidents since rollout" |
| Multi-cluster ArgoCD | "argocd cluster add → ClusterSecret → update AppProject destinations → new Applications" |
| Module major upgrade | "Read CHANGELOG → check TF + AWS + K8s provider minimums → run plan → look for destroys → dev first" |
