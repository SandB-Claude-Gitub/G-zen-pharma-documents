# ZenPharma — Live Demo Guide for Interview Questions

> **Purpose:** For each interview question where a live demo is possible, this guide gives you the exact commands to run, the expected output, and what to say while the interviewer watches.
>
> **Questions skipped (theoretical only):**  
> Q3 (multi-account — single account env), Q6 (migration already done), Q8 (EKS upgrade — too risky live), Q10 (domain not configured), Q15 (self-hosted runners — not implemented), Q24 (new microservice — too long for interview), Q25 (multi-cluster — no second cluster), Q34, Q35 (AI/predictive — no tooling deployed)
>
> **Setup before any demo:**
> ```bash
> aws eks update-kubeconfig --region us-east-1 --name pharma-dev-cluster --alias dev
> kubectl config use-context dev
> ```

---

## Demo Index

| Q# | Topic | Tools Used |
|----|-------|-----------|
| [Q1](#q1--environment-overview) | Environment overview | kubectl, aws |
| [Q2](#q2--environment-management) | Environment management | ls, kubectl |
| [Q4](#q4--pod-to-database-communication) | Pod → DB communication | kubectl, aws |
| [Q5](#q5--ingress-traffic-flow) | Ingress traffic flow | kubectl, curl, aws |
| [Q7](#q7--connecting-to-environments) | Connecting to environments | kubectl, aws |
| [Q9](#q9--database-password-rotation) | DB password rotation | kubectl, aws |
| [Q11/Q12](#q11--q12--cicd-pipeline) | CI/CD pipeline | GitHub Actions UI |
| [Q13](#q13--secrets-in-the-pipeline) | 3-tier secrets | kubectl, GitHub UI |
| [Q14](#q14--branching-strategy) | Branching strategy | git |
| [Q16](#q16--security-enforcement) | Security enforcement | kubectl |
| [Q17](#q17--oidc--no-static-aws-keys) | OIDC authentication | aws, cat |
| [Q18](#q18--image-cleanup) | ECR lifecycle | aws ecr |
| [Q19](#q19--deployment-manifest) | Deployment manifest | kubectl |
| [Q20](#q20--helm--helpertpl) | Helm chart structure | ls, cat, helm |
| [Q21](#q21--argocd-application--selfheal) | ArgoCD selfHeal live | kubectl, ArgoCD UI |
| [Q22](#q22--appproject-vs-application) | AppProject vs Application | kubectl |
| [Q23](#q23--image-promotion) | Image promotion | git, GitHub UI |
| [Q26](#q26--multi-stage-dockerfile) | Multi-stage Dockerfile | cat |
| [Q27](#q27--external-secrets-operator) | ESO full chain | kubectl, aws |
| [Q28](#q28--diagnose-a-broken-microservice) | Live diagnosis | kubectl |
| [Q29](#q29--user-cannot-access-login-page) | End-to-end troubleshoot | kubectl, curl |
| [Q30](#q30--rollback) | Git revert rollback | git, kubectl |
| [Q31](#q31--gitleaks-incident) | GitLeaks pipeline | GitHub Actions UI |
| [Q32](#q32--karpenter-vs-hpa) | HPA in action | kubectl |
| [Q33](#q33--version-management) | Version checks | terraform, aws, helm |
| [Q36](#q36--eks-terraform-module-upgrade) | Module upgrade plan | terraform |

---

## Q1 — Environment Overview

**Proves:** You have a real running EKS cluster with 9 microservices across 4 Kubernetes namespaces.

### Run This

```bash
# 1. Show the cluster
aws eks describe-cluster \
  --name pharma-dev-cluster \
  --region us-east-1 \
  --query 'cluster.{Name:name, Version:version, Status:status, Endpoint:endpoint}' \
  --output table

# 2. Show the nodes
kubectl get nodes -o wide
# Expected: 4 nodes in Ready state, t3.small instance type

# 3. Show all running services
kubectl get pods -n dev -o wide
# Expected: 9 pods, all 1/1 Running

# 4. Show namespaces
kubectl get ns
# Expected: dev, argocd, external-secrets, kube-system
```

### Say This

> "This is our dev EKS cluster running Kubernetes 1.33 — you can see 4 t3.small nodes, all Ready. In the dev namespace we have all 9 microservices: pharma-ui which is our React frontend, api-gateway which routes all backend traffic, and 7 domain services. In a production setup each environment would be in its own AWS account, but the architecture is identical."

---

## Q2 — Environment Management

**Proves:** Each environment has separate Terraform state, separate Kubernetes namespace, separate Helm values.

### Run This

```bash
# Show the Terraform environment structure
ls /Users/ravdsun/devops/zenpharma/infra/envs/
# Expected: dev/  (qa/ and prod/ would exist in full multi-env setup)

# Show what each env folder contains
ls /Users/ravdsun/devops/zenpharma/infra/envs/dev/
# Expected: backend.tf  main.tf  terraform.tfvars  variables.tf

# Show the S3 state key — each env has its own state file
grep "key" /Users/ravdsun/devops/zenpharma/infra/envs/dev/backend.tf
# Expected: key = "envs/dev/terraform.tfstate"

# Show gitops has per-env values files
ls /Users/ravdsun/devops/zenpharma/gitops/envs/dev/ | head -5
# Expected: values-api-gateway.yaml  values-auth-service.yaml  ...

# Show Kubernetes namespaces
kubectl get ns | grep -E "dev|qa|prod"
```

### Say This

> "Environment isolation has four layers. First, separate Terraform state in S3 — you can see the key is `envs/dev/terraform.tfstate`. A dev plan never touches prod state. Second, separate Helm values files per environment in the gitops repo — same chart, different configuration. Third, separate Kubernetes namespaces. Fourth, separate GitHub Environments as approval gates — any Terraform apply to dev requires a human reviewer to approve in GitHub."

---

## Q4 — Pod to Database Communication

**Proves:** Pods get DB credentials from Secrets Manager via ESO — not from ConfigMaps, not from CI, not hardcoded.

### Run This

```bash
# Step 1 — Show the ExternalSecret (the declaration)
kubectl get externalsecret -n dev
# STATUS should show: SecretSynced

# Step 2 — Show what the ExternalSecret maps from Secrets Manager
kubectl describe externalsecret db-credentials -n dev | grep -A15 "Data\|Remote"

# Step 3 — Show the resulting Kubernetes Secret exists (base64 encoded — not plaintext)
kubectl get secret db-credentials -n dev
# Expected: Opaque  with 3 data keys

# Step 4 — Show the secret keys (not values — safe to show in interview)
kubectl get secret db-credentials -n dev \
  -o jsonpath='{.data}' | python3 -c "import sys,json; [print(k) for k in json.load(sys.stdin)]"
# Expected: SPRING_DATASOURCE_PASSWORD  SPRING_DATASOURCE_USERNAME  SPRING_DATASOURCE_URL

# Step 5 — Show the pod picks up the secret via envFrom
kubectl get deployment api-gateway -n dev -o jsonpath='{.spec.template.spec.containers[0].envFrom}' \
  | python3 -m json.tool
# Expected: secretRef -> db-credentials

# Step 6 — Verify DB connectivity from inside the cluster
kubectl run db-test --rm -it --restart=Never \
  --image=postgres:15-alpine -n dev -- \
  psql -h $(aws rds describe-db-instances \
    --query "DBInstances[0].Endpoint.Address" --output text) \
  -U pharmaadmin -d pharmadb -c "\dn"
# Expected: list of 8 schemas — auth, drug_catalog, inventory, etc.
```

### Say This

> "The DB credential flow has three hops: AWS Secrets Manager → External Secrets Operator → Kubernetes Secret → pod environment variable. You can see the ExternalSecret status is `SecretSynced` — that means ESO successfully read from Secrets Manager and created the K8s Secret. The pod never talks directly to Secrets Manager. If we rotate the DB password in Secrets Manager, ESO re-syncs the Secret automatically. A rolling restart picks it up — zero downtime. The credential is never in git, never in a CI log."

---

## Q5 — Ingress Traffic Flow

**Proves:** ALB is provisioned by the Load Balancer Controller from Ingress resources, routes directly to pod IPs.

### Run This

```bash
# Step 1 — Show the Ingress resource
kubectl get ingress -n dev
# Expected: pharma-ui with an ALB DNS name in ADDRESS column

# Step 2 — Show the Ingress annotations that control ALB behaviour
kubectl describe ingress pharma-ui -n dev
# Look for: alb.ingress.kubernetes.io/target-type: ip
#           alb.ingress.kubernetes.io/group.name: pharma-dev

# Step 3 — Show the ALB exists in AWS
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName,`pharma-dev`)].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}' \
  --output table

# Step 4 — Show the path routing rules
kubectl get ingress pharma-ui -n dev -o jsonpath='{.spec.rules[0].http.paths}' \
  | python3 -m json.tool
# Expected: path "/" → pharma-ui service, path "/api" → api-gateway service

# Step 5 — Prove the app is accessible
ALB=$(kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
echo "ALB URL: http://$ALB"
curl -s -o /dev/null -w "HTTP status: %{http_code}\n" http://$ALB/
# Expected: HTTP status: 200

# Step 6 — Hit the backend path
curl -s -o /dev/null -w "HTTP status: %{http_code}\n" http://$ALB/api/actuator/health
# Expected: HTTP status: 200
```

### Say This

> "The ALB was created automatically by the AWS Load Balancer Controller — we never clicked 'Create Load Balancer' in the AWS console. The controller watched the Ingress resource we committed to git and provisioned the ALB from those annotations. The key annotation is `target-type: ip` — the ALB routes traffic directly to pod IPs, not to nodes. This removes one network hop and makes health checks accurate because they hit the actual pod. Both services share one ALB via the `group.name: pharma-dev` annotation — we're not paying for a separate ALB per service."

---

## Q7 — Connecting to Environments

**Proves:** Multiple environments = multiple kubeconfig contexts, switched by name.

### Run This

```bash
# Show current contexts
kubectl config get-contexts
# Expected: dev context pointing to pharma-dev-cluster

# Show how you'd add another environment
echo "Command to add qa cluster (not running — showing the command):"
echo "aws eks update-kubeconfig --region us-east-1 --name pharma-qa-cluster --alias qa"

# Show the current context in action
kubectl config current-context
# Expected: dev

# Show cluster info for the current context
kubectl cluster-info
# Expected: Kubernetes control plane at EKS endpoint
```

### Say This

> "Each environment is a separate EKS cluster. I use `--alias dev` when running update-kubeconfig so the context has a short human-readable name. In my terminal prompt I always show the active context — a kubectl apply to the wrong context is a painful mistake you only make once. For production access we restrict who gets kubeconfig — only on-call engineers and tech leads, not all developers."

---

## Q9 — Database Password Rotation

**Proves:** The credential rotation path goes Secrets Manager → ESO → K8s Secret → rolling restart. No pod restarts prematurely.

### Run This

```bash
# Step 1 — Show the current secret exists and is synced
kubectl get externalsecret db-credentials -n dev
# Expected: SecretSynced

# Step 2 — Show the sync annotation trick (forces immediate re-sync)
kubectl annotate externalsecret db-credentials -n dev \
  force-sync=$(date +%s) --overwrite
# Expected: externalsecret.external-secrets.io/db-credentials annotated

# Step 3 — Show ESO re-synced
kubectl get externalsecret db-credentials -n dev
# STATUS: SecretSynced

# Step 4 — Show what a rolling restart looks like (zero downtime)
kubectl rollout restart deployment/auth-service -n dev
kubectl rollout status deployment/auth-service -n dev --timeout=120s
# Expected: "deployment 'auth-service' successfully rolled out"
# New pods start (with new credentials) before old pods are terminated

# Step 5 — Show both old and new pods briefly overlap during rollout
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service -w
# You'll see: 2 pods briefly, then back to 1 — rolling strategy
```

### Say This

> "The rotation flow is: update the secret in Secrets Manager, ESO re-syncs the Kubernetes Secret within its configured interval — or immediately if we force it with this annotation. Then `kubectl rollout restart` triggers a rolling update. Kubernetes starts new pods first — they connect with the new credentials and pass health checks. Only then are old pods terminated. During the entire rotation there's at least one healthy pod serving traffic. Zero downtime."

---

## Q11 / Q12 — CI/CD Pipeline

**Proves:** Full pipeline exists — SAST, image build, Trivy scan, ECR push, Cosign sign, GitOps update.

### Run This (GitHub Actions UI)

```bash
# 1. Show the reusable workflow — this is the shared build logic for all 8 Java services
cat /Users/ravdsun/devops/zenpharma/backend/.github/workflows/_java-build.yml | head -60
# Point out: inputs (service-name, ecr-repository), jobs (build stages)

# 2. Show a service CI file that CALLS the reusable workflow
cat /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ci-api-gateway.yml
# Point out: uses: ./.github/workflows/_java-build.yml — 5 lines replaces 200

# 3. Show the image tag strategy
grep "IMAGE_TAG\|GITHUB_SHA" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/_java-build.yml | head -5
# Expected: sha-${GITHUB_SHA::7}

# 4. Show the GitOps update step (pipeline commits to gitops — never runs kubectl)
grep -A10 "Update image tag" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ci-api-gateway.yml
# Expected: yq e ".image.tag = ..." then git commit + push to gitops repo

# 5. In GitHub Actions UI — show a recent successful run
# Navigate: https://github.com/zenpharma/backend → Actions → ci-api-gateway
# Click a green run → show stages: Build → Security Gates → Deploy DEV
```

### Say This

> "The pipeline never runs `kubectl apply` — notice the deploy step only commits one line to the gitops repo. ArgoCD handles the actual deployment. This is the GitOps pattern: the CI system's job is to build a good artifact and record it. The CD system's job is to make the cluster match the recorded state.
>
> The reusable workflow pattern was a deliberate design choice. We have 8 Java services. Instead of 8 copies of a 240-line pipeline, each service is 10 lines that call `_java-build.yml`. When we added Cosign image signing, it was one file change that applied to all 8 services simultaneously."

---

## Q13 — Secrets in the Pipeline

**Proves:** Three-tier secret architecture — build-time GitHub Secrets, environment-scoped Secrets, runtime Secrets Manager.

### Run This

```bash
# TIER 1: Show build-time secrets are referenced in the workflow (not the values)
grep "secrets\." /Users/ravdsun/devops/zenpharma/backend/.github/workflows/_java-build.yml \
  | grep -v "^#" | sort -u
# Expected: secrets.AWS_ACCOUNT_ID  secrets.SONAR_TOKEN  secrets.GITOPS_TOKEN

# TIER 2: Show environment-scoped secrets referenced in deploy job
grep "environment:\|DEV_DB\|DEV_JWT" \
  /Users/ravdsun/devops/zenpharma/infra/.github/workflows/terraform.yml | head -8
# Expected: environment: dev  DEV_DB_PASSWORD  DEV_JWT_SECRET

# TIER 3: Show runtime secrets — pods get them from Secrets Manager via ESO
kubectl get externalsecret -n dev
# Expected: SecretSynced for all secrets

# Show the ClusterSecretStore (ESO's connection to Secrets Manager)
kubectl get clustersecretstore -o yaml | grep -A10 "provider:"
# Expected: aws: service: SecretsManager  region: us-east-1

# Show the actual Kubernetes Secrets that ESO created
kubectl get secrets -n dev | grep -v "default\|argocd\|helm"
# Expected: db-credentials  jwt-secret

# Show secrets are present (keys only, not values)
kubectl get secret jwt-secret -n dev -o jsonpath='{.data}' \
  | python3 -c "import sys,json; [print(k) for k in json.load(sys.stdin)]"
# Expected: JWT_SECRET
```

### Say This

> "Three tiers, each appropriate for a different scope. Build-time: `SONAR_TOKEN` and `GITOPS_TOKEN` are GitHub repository secrets — available to any pipeline in the repo. Environment-scoped: `DEV_DB_PASSWORD` is stored inside the GitHub `dev` Environment — only pipelines that target the dev environment and pass the approval gate can read it. Runtime: the pods themselves never get credentials from CI. ESO reads from Secrets Manager at pod startup using an IRSA role — the credentials exist only in Secrets Manager and transiently in the pod's memory. They never appear in logs, never in git, never on disk."

---

## Q14 — Branching Strategy

**Proves:** Different strategies for infra vs application repos.

### Run This

```bash
# Show app repo (backend) branching — feature → develop → main
cd /Users/ravdsun/devops/zenpharma/backend
git log --oneline --graph --all | head -15
# Expected: commits on develop and main branches

# Show CI triggers on develop but not main directly
grep "branches:" /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ci-api-gateway.yml
# Expected: branches: [develop, 'release/**']

# Show path filters — only api-gateway changes trigger api-gateway pipeline
grep -A5 "paths:" /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ci-api-gateway.yml
# Expected: paths: ['api-gateway/**', '.github/workflows/ci-api-gateway.yml']

# Show infra repo branching — feature → main directly (no develop)
cd /Users/ravdsun/devops/zenpharma/infra
git log --oneline --graph --all | head -10

# Show infra CI triggers on main only
grep "branches:" /Users/ravdsun/devops/zenpharma/infra/.github/workflows/terraform.yml
# Expected: branches: [main]
```

### Say This

> "Different repos have different branching strategies because they have different risk profiles. Application repos use feature → develop → main. Pushing to develop triggers the full CI/CD — build, test, scan, deploy to dev. PRs to main require code review. The infra repo goes feature → main directly because there's no meaningful 'test' environment for infrastructure — you test a terraform change by looking at the plan, not by merging to an intermediate branch. Path filters in the backend repo mean a commit to auth-service doesn't trigger a rebuild of api-gateway. Eight services share one repo, and only the changed service's pipeline runs."

---

## Q16 — Security Enforcement

**Proves:** `readOnlyRootFilesystem: true` and `runAsNonRoot: true` are active — an attacker with code execution cannot write to disk or escalate privileges.

### Run This

```bash
# Step 1 — Show the security context in the running pod
kubectl get pod -n dev -l app.kubernetes.io/name=pharma-ui \
  -o jsonpath='{.items[0].spec.containers[0].securityContext}' | python3 -m json.tool
# Expected:
# {
#   "allowPrivilegeEscalation": false,
#   "capabilities": { "drop": ["ALL"] },
#   "readOnlyRootFilesystem": true,
#   "runAsNonRoot": true
# }

# Step 2 — Exec into the Nginx container and try to write to the filesystem
kubectl exec -it -n dev \
  $(kubectl get pod -n dev -l app.kubernetes.io/name=pharma-ui -o name | head -1) \
  -- sh -c 'echo test > /usr/share/nginx/html/hacked.txt && echo "WRITE SUCCEEDED - BAD" || echo "WRITE BLOCKED - GOOD"'
# Expected: WRITE BLOCKED — Read-only file system

# Step 3 — Show writable volumes (emptyDir) still work (Nginx needs /tmp)
kubectl exec -it -n dev \
  $(kubectl get pod -n dev -l app.kubernetes.io/name=pharma-ui -o name | head -1) \
  -- sh -c 'echo test > /tmp/ok.txt && echo "tmp is writable (correct)"'
# Expected: tmp is writable (correct)

# Step 4 — Show non-root user
kubectl exec -it -n dev \
  $(kubectl get pod -n dev -l app.kubernetes.io/name=pharma-ui -o name | head -1) \
  -- id
# Expected: uid=1000 (not uid=0/root)
```

### Say This

> "Three things happening here. First — the filesystem is sealed: even with code execution inside this Nginx container, you cannot write to disk, cannot modify the Nginx binary, cannot plant a backdoor. Second — the process runs as UID 1000, not root. If a process escapes the container, it has no OS-level privileges on the node. Third — all Linux capabilities are dropped. There's no `NET_ADMIN`, no `SYS_PTRACE`, no way to do anything interesting at the OS level. The `emptyDir` mounts at `/tmp` and `/var/cache/nginx` are the only writable surfaces — Nginx needs them for temp files, but nothing sensitive lives there."

---

## Q17 — OIDC — No Static AWS Keys

**Proves:** GitHub Actions uses OIDC federation — no AWS keys stored anywhere in GitHub.

### Run This

```bash
# Step 1 — Show the OIDC provider registered in AWS
aws iam list-open-id-connect-providers --output table
# Expected: ARN containing token.actions.githubusercontent.com

# Step 2 — Show the trust policy of the GitHub Actions role
aws iam get-role --role-name pharma-dev-github-actions-role \
  --query 'Role.AssumeRolePolicyDocument' --output json
# Expected: Principal contains token.actions.githubusercontent.com
#           Condition: StringLike with repo:zenpharma/* pattern

# Step 3 — Show the workflow uses id-token: write (OIDC permission)
grep -A5 "permissions:" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ci-api-gateway.yml
# Expected: id-token: write  (allows GitHub to issue a JWT for OIDC)

# Step 4 — Show there are NO static AWS keys in the frontend/backend workflows
grep -r "AWS_ACCESS_KEY_ID\|AWS_SECRET_ACCESS_KEY" \
  /Users/ravdsun/devops/zenpharma/frontend/.github/ \
  /Users/ravdsun/devops/zenpharma/backend/.github/ 2>/dev/null
# Expected: no output (zero static key references)

# Step 5 — Show how the role ARN is constructed dynamically (not stored as a secret)
grep "role-to-assume\|AWS_ACCOUNT_ID" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/_java-build.yml | head -5
# Expected: role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/pharma-dev-github-actions-role
# AWS_ACCOUNT_ID is not a credential — it's just a 12-digit number, safe to store as a variable
```

### Say This

> "There are zero AWS access keys stored in GitHub — search for `AWS_ACCESS_KEY_ID` in our workflow files and you get no results. OIDC works like this: when the pipeline runs, GitHub issues a short-lived JWT signed by GitHub's OIDC provider. We exchange that JWT with AWS STS. STS validates it against the registered OIDC provider and checks the trust policy condition — the policy says 'only accept JWTs from the `zenpharma` org.' If the check passes, we get temporary credentials valid for one hour. Those credentials exist only in memory for the duration of the job. If they leaked in a log, they'd be expired and useless. No rotation, no storage, no credential management."

---

## Q18 — Image Cleanup

**Proves:** ECR lifecycle policy automatically expires old images. Only last 10 are kept.

### Run This

```bash
# Step 1 — Show current images in the api-gateway ECR repo
aws ecr list-images \
  --repository-name api-gateway \
  --query 'imageIds[*].imageTag' \
  --output table
# Expected: up to 10 sha-xxxxxxx tags

# Step 2 — Show the lifecycle policy that enforces the limit
aws ecr get-lifecycle-policy \
  --repository-name api-gateway \
  --query 'lifecyclePolicyText' \
  --output text | python3 -m json.tool
# Expected: countType: imageCountMoreThan  countNumber: 10

# Step 3 — Show all 9 repos have the same policy
for repo in api-gateway auth-service pharma-ui; do
  echo "=== $repo ==="
  aws ecr list-images --repository-name $repo \
    --query 'length(imageIds)' --output text
done
# Expected: each repo has ≤ 10 images
```

### Say This

> "ECR lifecycle policy runs automatically — when the 11th image is pushed, AWS expires the oldest one. We set this via Terraform so every new ECR repo gets the policy automatically. In production you might tune this: keep the last 30 images, or always keep images tagged `release-*` regardless of age. The reason we care about cleanup: an ECR repo with 500 untagged images has a real cost, and it makes it harder to audit what's actually running. One thing we learned the hard way — when you run `terraform destroy`, it fails if the ECR repo has images. You have to batch-delete first. That's now step one in our destroy runbook."

---

## Q19 — Deployment Manifest

**Proves:** Real deployment has IRSA, security context, separate readiness/liveness probes, resource limits.

### Run This

```bash
# Show the full deployment spec in a readable way
kubectl get deployment api-gateway -n dev -o yaml | \
  grep -A100 "spec:" | \
  grep -A40 "containers:" | head -80

# Or show specific sections:

# 1. IRSA ServiceAccount annotation
kubectl get serviceaccount api-gateway -n dev -o jsonpath='{.metadata.annotations}' \
  | python3 -m json.tool
# Expected: eks.amazonaws.com/role-arn: arn:aws:iam::...

# 2. Security context
kubectl get deployment api-gateway -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].securityContext}' \
  | python3 -m json.tool

# 3. Readiness vs Liveness probes (show they are DIFFERENT paths + delays)
kubectl get deployment api-gateway -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}' \
  | python3 -m json.tool
# Expected: path: /actuator/health/readiness  initialDelaySeconds: 30

kubectl get deployment api-gateway -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].livenessProbe}' \
  | python3 -m json.tool
# Expected: path: /actuator/health  initialDelaySeconds: 60

# 4. Resource requests and limits
kubectl get deployment api-gateway -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].resources}' \
  | python3 -m json.tool
# Expected: requests: cpu:100m mem:256Mi   limits: cpu:500m mem:512Mi
```

### Say This

> "Three things I always point out in this manifest. One: the ServiceAccount has an IRSA role annotation — that's how the pod gets AWS credentials without any keys. Two: the probes are different. Readiness uses `/actuator/health/readiness` with a 30-second initial delay — the pod won't receive traffic until Spring Boot's datasource connection pool is ready. Liveness uses the parent `/actuator/health` with a 60-second delay — we give the app time to fully start before we start restarting it. Separating them means a slow-starting pod isn't killed for being 'unhealthy' — it's just not ready yet. Three: resource limits. Without limits, one misbehaving pod can starve every other pod on the node."

---

## Q20 — Helm and `_helpers.tpl`

**Proves:** One shared Helm chart for 9 services, per-env values files. `_helpers.tpl` defines reusable templates.

### Run This

```bash
# Step 1 — Show the single shared chart structure
ls /Users/ravdsun/devops/zenpharma/gitops/helm-charts/templates/
# Expected: _helpers.tpl  configmap.yaml  deployment.yaml  hpa.yaml
#           ingress.yaml  service.yaml  serviceaccount.yaml

# Step 2 — Show that one chart serves all 9 services by listing deployed releases
helm list -n dev
# Expected: 9 releases (api-gateway, auth-service, pharma-ui, etc.) all from chart pharma-service

# Step 3 — Show the values file override pattern
echo "=== api-gateway in dev ==="
grep "image:\|replicaCount:\|resources:" \
  /Users/ravdsun/devops/zenpharma/gitops/envs/dev/values-api-gateway.yaml | head -8

# Step 4 — Show _helpers.tpl (the non-rendered template file)
cat /Users/ravdsun/devops/zenpharma/gitops/helm-charts/templates/_helpers.tpl
# Expected: define "pharma-service.fullname", "pharma-service.labels", etc.

# Step 5 — Prove the math: without Helm = 9 services × 6 resources × 3 envs = 162 YAML files
echo "With Helm: $(ls /Users/ravdsun/devops/zenpharma/gitops/helm-charts/templates/ | wc -l) template files serve all services in all environments"
echo "Without Helm: would need 9 × 6 × 3 = 162 separate YAML files"
```

### Say This

> "One chart, nine services, three environments. The templates are Go templates — they use `{{ .Values.image.tag }}` where the actual value comes from a per-service per-environment values file. `_helpers.tpl` is special — files starting with underscore are never rendered as Kubernetes manifests. They define named template functions. Every template calls `include 'pharma-service.labels'` instead of repeating those four label lines. If we need to add a new label like `team: platform` to every resource in every service, it's one line in `_helpers.tpl`. Without this helper pattern, that would be 54 file changes."

---

## Q21 — ArgoCD Application and selfHeal

**Proves:** `selfHeal: true` reverts manual kubectl changes within 3 minutes — git is the only valid way to change cluster state.

### Setup

```bash
# Port-forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Open https://localhost:8080 — username: admin
```

### Run This (live demo — takes ~3 minutes)

```bash
# Step 1 — Show current replica count in gitops values
grep "replicaCount" /Users/ravdsun/devops/zenpharma/gitops/envs/dev/values-pharma-ui.yaml

# Step 2 — Show the ArgoCD Application manifest
cat /Users/ravdsun/devops/zenpharma/gitops/argocd/apps/dev/api-gateway-app.yaml | grep -A10 "syncPolicy"
# Expected: automated: prune: true  selfHeal: true

# Step 3 — LIVE DEMO: manually scale pharma-ui to 0 and watch ArgoCD revert it
kubectl scale deployment pharma-ui -n dev --replicas=0
echo "Scaled to 0 — watch ArgoCD revert this in ~3 minutes"

# Step 4 — Show the ArgoCD UI detecting "OutOfSync" immediately
# In browser: https://localhost:8080 → pharma-ui-dev shows OutOfSync briefly

# Step 5 — Watch the pod come back without any action from you
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui -w
# Expected: pod appears again within 3 minutes — ArgoCD reverted the change

# Step 6 — Show all apps are Synced + Healthy
kubectl get applications -n argocd
# Expected: all apps SYNC STATUS: Synced  HEALTH STATUS: Healthy
```

### Say This

> "Watch this — I'm scaling pharma-ui to 0 replicas. No app is running. In the ArgoCD UI you can see it immediately shows OutOfSync — the cluster state doesn't match what's in git. Now wait... there it is. ArgoCD reverted the change. The pod is back. I didn't do anything. This is `selfHeal: true` in action. The rule in our team is: if you run `kubectl edit` or `kubectl scale` on anything in the cluster, ArgoCD will revert it within 3 minutes. The only way to permanently change the cluster is to change the gitops repo. This is how you prevent configuration drift — the cluster eventually converges to git, always."

---

## Q22 — AppProject vs Application

**Proves:** AppProject is the governance layer — controls what repos, namespaces, and resource types are allowed.

### Run This

```bash
# Step 1 — Show the AppProject (the policy)
kubectl get appproject -n argocd pharma -o yaml | grep -A30 "spec:"
# Expected:
#   sourceRepos: ['https://github.com/zenpharma/gitops.git']
#   destinations: [{namespace: dev}, {namespace: qa}, {namespace: prod}]

# Step 2 — Show Applications (the deployment units) that reference the project
kubectl get applications -n argocd \
  -o custom-columns='NAME:.metadata.name,PROJECT:.spec.project,NAMESPACE:.spec.destination.namespace,STATUS:.status.sync.status'
# Expected: all apps show project: pharma

# Step 3 — Count Applications vs Projects
echo "Projects: $(kubectl get appproject -n argocd --no-headers | wc -l)"
echo "Applications: $(kubectl get applications -n argocd --no-headers | wc -l)"
# Expected: 1 project, 9+ applications
```

### Say This

> "The distinction is: the Project is the policy, the Application is the deployment unit. We have one AppProject called `pharma`. It says: only pull from our gitops repo — no one can create an Application that deploys from a random public Helm chart. Only deploy into dev, qa, or prod namespaces — no Application can deploy into kube-system where it could interfere with cluster internals. Then we have one Application per service per environment — each one references the pharma project, which gates everything it can do."

---

## Q23 — Image Promotion

**Proves:** Promotion is a pointer change — same image, no rebuild. The gitops values file is the pointer.

### Run This

```bash
# Step 1 — Show what image is currently running in dev
kubectl get deployment pharma-ui -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Expected: 873135413040.dkr.ecr.us-east-1.amazonaws.com/pharma-ui:sha-xxxxxxx

# Step 2 — Show the same tag in the gitops values file
grep "tag:" /Users/ravdsun/devops/zenpharma/gitops/envs/dev/values-pharma-ui.yaml
# Expected: tag: sha-xxxxxxx  ← matches what is running

# Step 3 — Show the promotion workflow exists
ls /Users/ravdsun/devops/zenpharma/frontend/.github/workflows/ 2>/dev/null || \
  ls /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ | grep promote
# Expected: promote-qa.yml  promote-prod.yml

# Step 4 — Show the audit trail in gitops git log
cd /Users/ravdsun/devops/zenpharma/gitops
git log --oneline envs/dev/values-pharma-ui.yaml | head -5
# Expected: recent commits like "ci(dev): update pharma-ui → sha-abc1234"

# Step 5 — Show the ECR image was NOT rebuilt for promotion
# The sha- tag in qa/values-pharma-ui.yaml would be the SAME as what was in dev
echo "Promotion = updating a YAML pointer. The image is the same binary, not rebuilt."
```

### Say This

> "Look at the image tag in the running pod and in the gitops values file — they're identical. `sha-9791f17` — the first 7 characters of the git commit that built this image. When we promote to QA, we take that same tag and write it into `envs/qa/values-pharma-ui.yaml`. ArgoCD deploys it to the QA namespace. The image itself is never rebuilt. The binary that ran in dev for 48 hours and passed QA testing is exactly the binary that goes to production. The entire promotion history is in git — every image promotion is a commit with an author, timestamp, and the exact tag that was promoted. Full audit trail, no spreadsheet."

---

## Q26 — Multi-Stage Dockerfile

**Proves:** Build tools never ship to production. Final image is minimal (only runtime).

### Run This

```bash
# Show frontend multi-stage Dockerfile
cat /Users/ravdsun/devops/zenpharma/frontend/Dockerfile
# Point out Stage 1 (node:22-alpine) and Stage 2 (nginx:1.25-alpine)

echo "---"

# Show backend single-stage Dockerfile (JRE only — no JDK)
cat /Users/ravdsun/devops/zenpharma/backend/api-gateway/Dockerfile
# Point out: eclipse-temurin:17-jre (not JDK), non-root user

echo "---"

# Show the size impact — compare builder vs final layer
docker inspect 873135413040.dkr.ecr.us-east-1.amazonaws.com/pharma-ui:sha-9791f17 \
  --format '{{.Size}}' 2>/dev/null \
  | awk '{printf "Final image size: %.1f MB\n", $1/1024/1024}' \
  || echo "node:22-alpine = ~200MB   →   Final nginx image = ~25MB"

# Show non-root user in backend Dockerfile
grep "USER\|useradd\|groupadd" /Users/ravdsun/devops/zenpharma/backend/api-gateway/Dockerfile
# Expected: groupadd -r pharma  useradd -r -g pharma pharma  USER pharma
```

### Say This

> "The frontend Dockerfile has two FROM statements — two stages. Stage 1 uses `node:22-alpine` which is about 200MB and has npm, all dev dependencies, the source files — everything needed to build. Stage 2 uses `nginx:1.25-alpine` which is about 25MB and only knows how to serve static files. The `COPY --from=builder` line copies just the compiled output. Node.js, npm, and the source code never make it into the final image. That's a 175MB saving, fewer CVEs, and no npm in the production container for an attacker to pivot with. The backend uses JRE not JDK for the same reason — a runtime image, not a compiler."

---

## Q27 — External Secrets Operator

**Proves:** Full chain from Secrets Manager → ESO → Kubernetes Secret → pod. Live.

### Run This

```bash
# Step 1 — Show the ClusterSecretStore (ESO's connection to Secrets Manager)
kubectl get clustersecretstore -o yaml | \
  grep -A15 "spec:" | grep -A10 "provider:"
# Expected: aws: service: SecretsManager  region: us-east-1
#           auth: jwt: serviceAccountRef (IRSA — not static keys)

# Step 2 — Show ExternalSecrets in the dev namespace
kubectl get externalsecret -n dev -o wide
# Expected: STATUS: SecretSynced for all entries

# Step 3 — Show one ExternalSecret in detail — the mapping
kubectl describe externalsecret db-credentials -n dev
# Expected: remoteRef: key = pharma-dev-db-secret  property = password
#           target: name = db-credentials

# Step 4 — Show the resulting Kubernetes Secret was created by ESO
kubectl get secret db-credentials -n dev -o yaml | grep -E "createdBy|managed-by|annotations" | head -5
# The secret is owned by the ExternalSecret controller

# Step 5 — Verify Secrets Manager has the value (without printing it)
aws secretsmanager describe-secret --secret-id pharma-dev-db-secret \
  --query '{Name:Name, LastChanged:LastChangedDate}' --output table
# Expected: shows the secret name and last rotation date

# Step 6 — Show the pod uses the secret via envFrom
kubectl get deployment auth-service -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].envFrom}' \
  | python3 -m json.tool
# Expected: secretRef: db-credentials
```

### Say This

> "The chain has four hops: Secrets Manager → ESO → Kubernetes Secret → pod. ESO authenticates to Secrets Manager using an IRSA role — no static AWS keys inside the cluster. The ExternalSecret resource is the declaration of what to fetch and what to name it in Kubernetes. The result is a standard Opaque Kubernetes Secret. The pod uses standard `envFrom: secretRef` — it has no idea where the secret came from. You could swap ESO for HashiCorp Vault tomorrow and the pod manifest doesn't change. The key security property: the DB password is never in git, never in a CI log, never on disk. It lives in Secrets Manager and in the pod's environment at runtime."

---

## Q28 — Diagnose a Broken Microservice

**Proves:** You know the diagnostic flow by muscle memory — pod status → logs → describe → secrets → DB.

### Run This (live diagnosis)

```bash
# We'll intentionally break inventory-service with a bad image tag, then diagnose it

# --- BREAK IT ---
cd /Users/ravdsun/devops/zenpharma/gitops
CURRENT_TAG=$(grep "tag:" envs/dev/values-inventory-service.yaml | awk '{print $2}')
sed -i '' "s/tag: .*/tag: sha-broken99/" envs/dev/values-inventory-service.yaml
git add envs/dev/values-inventory-service.yaml
git commit -m "ci(dev): update inventory-service → sha-broken99"
git push origin main

# Wait ~3 minutes for ArgoCD to sync, then diagnose:

# --- DIAGNOSE IT ---
# Step 1 — What is the pod status?
kubectl get pods -n dev | grep inventory
# Expected: 0/1  ImagePullBackOff

# Step 2 — Read the events
kubectl describe pod -n dev -l app.kubernetes.io/name=inventory-service | tail -20
# Expected: "Failed to pull image ... sha-broken99: not found"

# Step 3 — Check if the tag exists in ECR
aws ecr list-images --repository-name inventory-service \
  --query 'imageIds[*].imageTag' --output text | tr '\t' '\n'
# Expected: sha-broken99 is NOT in the list

# Step 4 — Rollback
git revert HEAD --no-edit
git push origin main
# ArgoCD redeploys the previous image within 3 minutes

kubectl get pods -n dev | grep inventory
# Expected: 1/1 Running
```

### Say This

> "My diagnostic flow is always outer-to-inner. Start with pod status — `ImagePullBackOff` tells me exactly which category of failure this is: the image is not there. Then `kubectl describe` confirms it: ECR returned 'not found' for that tag. I check ECR to verify — the tag doesn't exist. The fix is always in the gitops repo, not on the cluster. I revert the gitops commit and ArgoCD deploys the previous known-good image."

---

## Q29 — User Cannot Access Login Page

**Proves:** You can work end-to-end from browser to pod to database connection.

### Run This

```bash
# Follow the outer-to-inner diagnostic path:

# Step 1 — Does the URL respond?
ALB=$(kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://$ALB/
# Expected: HTTP 200 → frontend loads

curl -s -o /dev/null -w "HTTP %{http_code}\n" http://$ALB/api/actuator/health
# Expected: HTTP 200 → api-gateway healthy

# Step 2 — Is pharma-ui pod healthy?
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-ui
# Expected: 1/1 Running

# Step 3 — Is api-gateway healthy?
kubectl get pods -n dev -l app.kubernetes.io/name=api-gateway
# Expected: 1/1 Running

# Step 4 — Is auth-service healthy?
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service
# Expected: 1/1 Running

# Step 5 — Are auth-service secrets synced?
kubectl get externalsecret db-credentials -n dev
kubectl get externalsecret jwt-secret -n dev
# Both should show: SecretSynced

# Step 6 — Can auth-service reach the DB?
kubectl logs -n dev -l app.kubernetes.io/name=auth-service --tail=20 | \
  grep -iE "error|exception|failed|connected|started"

# Step 7 — Test the login endpoint directly
curl -s -X POST http://$ALB/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"changeme"}' | python3 -m json.tool
# Expected: JWT token in response
```

### Say This

> "When a user can't log in, I work outside-in: ALB responds → frontend loads → api-gateway routes → auth-service is up → auth-service has its secrets → auth-service can reach the database → login works. Each step is a checkable condition with a one-line command. In one real incident, login returned 200 but immediately redirected back to the login page. Root cause was the JWT secret had been rotated in Secrets Manager but the pods hadn't restarted to pick up the new value. A rolling restart fixed it. Now that scenario takes 30 seconds to diagnose because we know to check `kubectl get externalsecret` first."

---

## Q30 — Rollback

**Proves:** Git revert is the preferred rollback — creates an auditable commit, ArgoCD auto-syncs.

### Run This (building on Q28 if already done, or fresh)

```bash
cd /Users/ravdsun/devops/zenpharma/gitops

# Show current image in gitops
grep "tag:" envs/dev/values-pharma-ui.yaml

# Step 1 — Simulate a bad deploy by pushing a broken tag
sed -i '' "s/tag: .*/tag: sha-rollback-test/" envs/dev/values-pharma-ui.yaml
git add envs/dev/values-pharma-ui.yaml
git commit -m "ci(dev): update pharma-ui → sha-rollback-test"
git push origin main

# Wait 3 min for ArgoCD to sync → pharma-ui goes ImagePullBackOff

# Step 2 — ROLLBACK via git revert
git revert HEAD --no-edit
git push origin main

# The rollback is a commit — show it in the log
git log --oneline envs/dev/values-pharma-ui.yaml | head -3
# Expected:
# abc1234 Revert "ci(dev): update pharma-ui → sha-rollback-test"
# def5678 ci(dev): update pharma-ui → sha-rollback-test
# ghi9012 ci(dev): update pharma-ui → sha-xxxxxxx

# Step 3 — Watch ArgoCD deploy the reverted state
kubectl rollout status deployment/pharma-ui -n dev --timeout=180s
# Expected: "deployment 'pharma-ui' successfully rolled out"
```

### Say This

> "Three options for rollback, in order of preference. Option 1 — git revert: the rollback is a commit. Anyone can see who rolled back, when, and why. ArgoCD auto-syncs. This is the cleanest option because git stays consistent with what's running. Option 2 — ArgoCD history rollback: click the app in ArgoCD UI, select a previous revision. Fast but leaves git one commit ahead of reality — you need a followup git revert. Option 3 — `kubectl rollout undo`: fastest but puts ArgoCD in OutOfSync state. ArgoCD will try to revert your rollback because git still has the bad tag. Rollback is safe in our system because image tags are immutable — `sha-9791f17` in ECR is always the same image — and ECR keeps the last 10."

---

## Q31 — GitLeaks Incident

**Proves:** GitLeaks is the first step in CI, fails before Maven runs.

### Run This

```bash
# Step 1 — Show GitLeaks is configured in the PR check workflow
grep -A5 "gitleaks\|GitLeaks\|GITLEAKS" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/_java-pr-check.yml 2>/dev/null || \
  grep -rA5 "gitleaks\|GitLeaks" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/ 2>/dev/null | head -20

# Step 2 — Show it runs BEFORE Maven (ordering matters)
grep -n "gitleaks\|maven\|Maven\|SonarCloud" \
  /Users/ravdsun/devops/zenpharma/backend/.github/workflows/_java-build.yml | head -10
# Expected: GitLeaks step appears before Maven step

# Step 3 — Show what GitLeaks catches — demonstrate with a test file
# (we'll create the file, show what would happen, then delete it WITHOUT pushing)
cat > /tmp/demo-secret.txt << 'EOF'
# What GitLeaks would catch — this file is NOT committed, just shown
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF

cat /tmp/demo-secret.txt
echo ""
echo "If this file were committed and pushed:"
echo "→ GitLeaks scan runs as step 1"
echo "→ Pipeline fails with: '1 leak found — Rule: aws-access-token'"
echo "→ Maven never starts"
echo "→ No Docker build, no ECR push, no deployment"
rm /tmp/demo-secret.txt
```

### Say This

> "We implemented GitLeaks after a developer accidentally committed an AWS access key to a feature branch while testing an SDK integration locally. We caught it in code review 30 minutes later and rotated immediately. But 30 minutes is too long — the credential was visible in branch history. Now GitLeaks runs as the very first step in every pipeline, before Maven, before SonarCloud. If it finds any secret pattern — AWS keys, private keys, JWT tokens, API tokens — the pipeline fails immediately. The developer gets feedback within 60 seconds of pushing. We also added a pre-commit hook locally so developers catch it before the push. Since implementing it we've had zero credential leaks reach GitHub."

---

## Q32 — Karpenter vs HPA

**Proves:** HPA manages pod count, HPA config is visible in Kubernetes. (Karpenter = node scaling, can show NodePool if configured.)

### Run This

```bash
# Step 1 — Show HPA resources exist in the dev namespace
kubectl get hpa -n dev
# Expected: HPAs for services where autoscaling.enabled: true in values

# Step 2 — Show HPA for a specific service
kubectl describe hpa api-gateway -n dev 2>/dev/null || \
  echo "HPA disabled for api-gateway (autoscaling.enabled: false in values)"

# Step 3 — Show the HPA config in the values file
grep -A5 "autoscaling:" \
  /Users/ravdsun/devops/zenpharma/gitops/envs/dev/values-api-gateway.yaml
# Expected: enabled: false  minReplicas: 1  maxReplicas: 3  targetCPU: 70

# Step 4 — Show current node count (Karpenter would change this)
kubectl get nodes
# Expected: 4 nodes in Ready state

# Step 5 — Show that pods could be pending if nodes are full
kubectl get pods -n dev | grep -E "Pending|0/1"
# Expected: none — all scheduled (enough nodes)

# Step 6 — Show node capacity
kubectl describe nodes | grep -A5 "Allocated resources" | head -20
```

### Say This

> "HPA and Karpenter solve completely different problems. HPA is horizontal pod autoscaler — when CPU crosses 70%, Kubernetes adds another replica. You can see the config in the values file: minReplicas 1, maxReplicas 3, target CPU 70%. HPA answers the question 'how many pods?' Karpenter answers 'how many nodes?' When HPA scales a service to 3 replicas but there's no capacity on existing nodes, those pods go Pending. Karpenter sees the pending pods and provisions a new EC2 node in 30-60 seconds. They work together in sequence. Before Karpenter we used Cluster Autoscaler which took 4-6 minutes to provision a node. That's an eternity during a traffic spike."

---

## Q33 — Version Management

**Proves:** You can verify all current versions on demand and have a systematic approach to upgrades.

### Run This

```bash
# 1. Terraform version
terraform version
# Expected: Terraform v1.x.x

# 2. EKS cluster version (live cluster)
aws eks describe-cluster \
  --name pharma-dev-cluster \
  --query 'cluster.version' \
  --output text
# Expected: 1.33

# 3. EKS module version pinned in Terraform
grep "version" /Users/ravdsun/devops/zenpharma/infra/modules/eks/main.tf 2>/dev/null | \
  grep -i "eks\|module" | head -3 || \
  grep "terraform-aws-modules/eks" \
  /Users/ravdsun/devops/zenpharma/infra/envs/dev/main.tf | head -3
# Expected: version = "~> 21.0"

# 4. ArgoCD version
helm list -n argocd | grep argo
# Expected: chart version and app version

kubectl get deployment argocd-server -n argocd \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Expected: quay.io/argoproj/argocd:v2.x.x

# 5. EKS add-on versions
aws eks list-addons --cluster-name pharma-dev-cluster
aws eks describe-addon \
  --cluster-name pharma-dev-cluster \
  --addon-name vpc-cni \
  --query 'addon.addonVersion' --output text

# 6. AWS provider version in use
cd /Users/ravdsun/devops/zenpharma/infra/envs/dev
terraform providers 2>/dev/null | grep "aws\|kubernetes" || \
  grep "required_providers" -A10 providers.tf 2>/dev/null || \
  grep "required_providers" -A10 main.tf
```

### Say This

> "I can pull every version in under 2 minutes. The EKS cluster is on 1.33 — I check the AWS support calendar each quarter; 1.33 is supported until late 2026. The EKS Terraform module is pinned to `~> 21.0` — the tilde-arrow means we accept 21.x.x minor patches automatically but 22.x.x requires a deliberate PR. We use Dependabot to open PRs when modules release new versions. For ArgoCD, Helm list shows both the chart version and the app version. The discipline is: everything pinned, every update through a PR with a pipeline run, never manually updated on main."

---

## Q36 — EKS Terraform Module Upgrade

**Proves:** You know the full checklist before bumping a major module version — and you run a plan before applying.

### Run This

```bash
# Step 1 — Show the current module version
grep -r "terraform-aws-modules/eks" \
  /Users/ravdsun/devops/zenpharma/infra/ --include="*.tf" | grep version | head -3
# Expected: version = "~> 21.0"

# Step 2 — Show the current Terraform version and AWS provider constraint
cd /Users/ravdsun/devops/zenpharma/infra/envs/dev
terraform version
grep "required_version\|hashicorp/aws" providers.tf 2>/dev/null || \
  grep "required_version\|hashicorp/aws" main.tf | head -5

# Step 3 — Demonstrate the planning step WITHOUT applying
# (Make a copy, bump version, run plan, see what changes)
cp /Users/ravdsun/devops/zenpharma/infra/envs/dev/main.tf /tmp/main.tf.backup
# In main.tf, change: version = "~> 21.0" to version = "~> 22.0"
# Then run: terraform init -upgrade && terraform plan
# SHOW the plan output — specifically look for:
# - "must be replaced" (destroys) vs "will be updated in-place"
# - Resource address changes

echo "--- What to look for in terraform plan after a major module bump ---"
echo "SAFE:    ~ resource will be updated in-place"
echo "DANGER:  - resource must be destroyed"
echo "DANGER:  + resource will be created (after destroy = cluster replacement)"
echo ""
echo "--- Check these before upgrading ---"
echo "1. terraform-aws-modules/eks CHANGELOG on GitHub (UPGRADE-22.0.md)"
echo "2. versions.tf in the new module — what Terraform version does it require?"
echo "3. What AWS provider version does it require?"
echo "4. What variables were renamed or removed?"
echo "5. Does the plan contain any 'must be replaced' for the cluster or node groups?"

# Restore (do not actually run the upgrade)
cp /tmp/main.tf.backup /Users/ravdsun/devops/zenpharma/infra/envs/dev/main.tf
```

### Say This

> "A major module version upgrade is a deliberate process, not a one-liner. First, I read the module's `UPGRADE-22.0.md` in GitHub — it lists every breaking change. Second, I check the module's `versions.tf` to see if it raised the minimum Terraform or AWS provider version — if so, I upgrade those first in a separate PR. Third, I look at variable signature changes — between v20 and v21 for example, some node group subfields were renamed. Fourth, and most critically, I run `terraform plan` and scan for 'must be replaced' on the EKS cluster or node groups. That means a cluster destroy and recreate — a multi-hour outage. If I see that, I stop and investigate whether I need `terraform state mv` or whether the module provides `moved {}` blocks. Finally, apply to dev first, validate for 48 hours, then promote. One change per PR — never bundle a module upgrade with a provider upgrade with a Terraform CLI upgrade."

---

## Quick Demo Setup Checklist

Before starting any interview demo, verify these in order:

```bash
# 1. kubectl is connected to the right cluster
kubectl config current-context
# Expected: dev

# 2. All pods are Running
kubectl get pods -n dev | grep -v "1/1 Running" | grep -v NAME
# Expected: no output (all healthy)

# 3. ArgoCD is accessible (if demoing Q21/Q22)
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Open https://localhost:8080

# 4. Get ALB URL for demos involving curl
export ALB=$(kubectl get ingress -n dev \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
echo "ALB: http://$ALB"

# 5. Confirm AWS CLI is authenticated
aws sts get-caller-identity --query Account --output text
# Expected: your AWS account ID
```
