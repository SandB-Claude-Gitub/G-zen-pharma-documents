# Module 8 — Environment Promotion (Dev → QA → Prod)

> Set up the promotion pipeline to move verified images from Dev through QA into Production with appropriate approval gates at each stage.
> Estimated time: 2-3 hours.

---

## 8.1 Understanding the Promotion Flow

Before creating any configuration, let's understand the full lifecycle of a code change — from a developer's commit all the way to production.

### The Full Promotion Lifecycle

```
Developer pushes to develop
     ↓
CI: lint → test → scan → build → Docker → ECR push
     ↓
CI updates envs/dev/values-*.yaml (direct commit to gitops)
     ↓
ArgoCD auto-syncs DEV deployment
     ↓
Developer validates DEV, then manually triggers: promote-qa workflow
     ↓
Opens PR: envs/qa/values-*.yaml → QA team reviews → merge
     ↓
ArgoCD auto-syncs QA deployment
     ↓
Manual trigger: promote-prod workflow
     ↓
Opens PR: envs/prod/values-*.yaml → change board reviews → merge
     ↓
ArgoCD PROD: manual sync → deploy
```

### Why Three Different Transition Models?

Each environment has a deliberately different promotion mechanism. Here is why:

**DEV: Direct commit (no PR)**

The CI pipeline writes the new image tag directly to `envs/dev/values-*.yaml` and pushes to the gitops repo. There is no pull request, no review, no approval. Why? Dev is the fast iteration environment. Every merge to `develop` should deploy automatically so developers can see their changes immediately. Speed matters more than ceremony here.

**QA: Manual trigger, manual merge**

When the DEV deployment is validated, a developer manually triggers the `promote-qa` workflow. This workflow reads the image tag currently running in DEV and opens a PR targeting `envs/qa/values-*.yaml`. A human must review and merge it. Why? QA is where the team validates that what worked in dev actually works in a more realistic setting. The PR gives the QA team a chance to:
- Confirm that dev testing passed before promoting
- Review what exactly is changing (just an image tag, or were configs modified too?)
- Delay the promotion if QA is in the middle of another test cycle

> **Why not auto-promote to QA from the CI pipeline?** If every merge to `develop` automatically created a QA PR, you'd get a new PR for every build — most of which are work-in-progress. Making QA promotion manual keeps the gitops repo clean and ensures only validated builds reach QA.

**PROD: Manual trigger, manual PR, manual ArgoCD sync**

Production has three gates:
1. A human must manually trigger the `promote-prod` workflow (no automated trigger)
2. The workflow opens a PR that requires review and approval before merge
3. ArgoCD's prod app is configured for manual sync (for real production; we use automated sync in this course for simplicity)

Why three gates? Because accidental production deployments are the most expensive mistakes. Each gate ensures a different stakeholder has signed off: the developer who triggered it, the reviewer who approved the PR, and the operator who clicked "sync" in ArgoCD.

> **Why not just one workflow for all environments?** Separating the promotion mechanism per environment lets you add environment-specific checks. For example, QA promotion could require passing integration tests, while PROD promotion could require a change management ticket number.

---

## 8.2 Add QA Environment Configuration

Pharma-ui is running in DEV. Now we create the QA environment configuration — values file and ArgoCD Application manifest. We demonstrate environment promotion with **pharma-ui only**; the same pattern applies identically to every backend service if you want to extend this on your own later.

### Step 1: Create the QA Values File

```bash
cd ~/devops/zenpharma/gitops
```

Create the pharma-ui QA values file:

```bash
cat > envs/qa/values-pharma-ui.yaml << 'EOF'
replicaCount: 1

fullnameOverride: pharma-ui

image:
  repository: <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/pharma-ui
  tag: v1.0.0
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80
  targetPort: 80

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: pharma-qa
  host: qa.pharma.internal
  path: /
  pathType: Prefix

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70

livenessProbe:
  path: /
  port: 80
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

readinessProbe:
  path: /
  port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

configmap:
  API_BASE_URL: "/api"
  AUTH_BASE_URL: "/api/auth"
  ENV: qa

# Nginx requires writable directories even with readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: nginx-cache
    mountPath: /var/cache/nginx
  - name: nginx-run
    mountPath: /var/run

volumes:
  - name: tmp
    emptyDir: {}
  - name: nginx-cache
    emptyDir: {}
  - name: nginx-run
    emptyDir: {}

serviceAccount:
  create: true
  annotations: {}
  name: pharma-ui
EOF
```

> **What changed from DEV?**
> - `alb.ingress.kubernetes.io/group.name`: `pharma-qa` instead of `pharma-dev` — this creates a separate ALB for QA traffic
> - `host`: `qa.pharma.internal` instead of `""` — QA has a hostname for routing
> - `configmap.ENV`: `qa` instead of `dev` — the application knows which environment it's in
> - `image.tag`: starts as `v1.0.0` (a placeholder) — the CI pipeline will update this when promoting from DEV

> **Why a placeholder tag like `v1.0.0`?** When ArgoCD first sees this file, we don't have a QA-promoted image yet. The placeholder prevents ArgoCD from trying to pull an image that doesn't exist. Once the first QA promotion PR is merged, CI will replace this tag with the actual image SHA.

> **Extending to backend services:** If you want to apply this pattern to the 8 backend services later, copy each `envs/dev/values-<service>.yaml` to `envs/qa/values-<service>.yaml` and update `group.name`, `host`, `SPRING_PROFILES_ACTIVE` (or `NODE_ENV` for notification-service), and `serviceAccount.annotations` the same way.

### Step 2: Create the QA ArgoCD Application Manifest

The ArgoCD Application manifest tells ArgoCD:
- Which Helm chart to use
- Which values file to overlay
- Which namespace to deploy into

Create the pharma-ui QA ArgoCD app:

```bash
cat > argocd/apps/qa/pharma-ui-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pharma-ui-qa
  namespace: argocd
  labels:
    env: qa
    app: pharma-ui
    managed-by: terraform
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma

  source:
    repoURL: https://github.com/<YOUR_ORG>/zen-gitops.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/qa/values-pharma-ui.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: qa

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 10
EOF
```

> **Key differences from the DEV ArgoCD app:**
> - `metadata.name`: `pharma-ui-qa` (includes the environment suffix)
> - `metadata.labels.env`: `qa`
> - `helm.valueFiles`: points to `../envs/qa/values-pharma-ui.yaml`
> - `destination.namespace`: `qa` (not `dev`)
> - Sync policy is still `automated` — same as dev. QA promotion is gated by the PR merge, not by ArgoCD sync policy.

> **Extending to backend services:** Each backend service would get its own `argocd/apps/qa/<service>-app.yaml` following the same template — just change `metadata.name`, `app` label, and `helm.valueFiles` to point at that service's QA values file.

### Step 3: Apply the QA ArgoCD App

```bash
kubectl apply -f argocd/apps/qa/pharma-ui-app.yaml
```

**Expected output:**
```
application.argoproj.io/pharma-ui-qa created
```

### Step 4: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add envs/qa/values-pharma-ui.yaml argocd/apps/qa/pharma-ui-app.yaml
git commit -m "feat: add QA environment values and ArgoCD app for pharma-ui"
git push origin main
```

> **Tag `gitops` repo: `module-8.2-qa-environment`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-8.2-qa-environment -m "Module 8.2: QA environment configuration for pharma-ui"
> git push origin module-8.2-qa-environment
> ```

---

## 8.3 Add Prod Environment Configuration

Production follows the same structure as QA but with production-appropriate settings. Again, we configure **pharma-ui only**.

### Step 1: Create the Prod Values File

Create the pharma-ui PROD values file:

```bash
cd ~/devops/zenpharma/gitops

cat > envs/prod/values-pharma-ui.yaml << 'EOF'
replicaCount: 1
fullnameOverride: pharma-ui
image:
  repository: <YOUR_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/pharma-ui
  tag: v1.0.0
  pullPolicy: Always
service:
  type: ClusterIP
  port: 80
  targetPort: 80
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: pharma-prod
  host: prod.pharma.internal
  path: /
  pathType: Prefix
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70
livenessProbe:
  path: /
  port: 80
  initialDelaySeconds: 10
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
readinessProbe:
  path: /
  port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
configmap:
  API_BASE_URL: "/api"
  AUTH_BASE_URL: "/api/auth"
  ENV: prod
# Nginx requires writable directories even with readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: nginx-cache
    mountPath: /var/cache/nginx
  - name: nginx-run
    mountPath: /var/run
volumes:
  - name: tmp
    emptyDir: {}
  - name: nginx-cache
    emptyDir: {}
  - name: nginx-run
    emptyDir: {}
serviceAccount:
  create: true
  annotations: {}
  name: pharma-ui
EOF
```

> **What changed from QA?**
> - `alb.ingress.kubernetes.io/group.name`: `pharma-prod` instead of `pharma-qa` — production traffic goes through its own ALB
> - `host`: `prod.pharma.internal` instead of `qa.pharma.internal`
> - `configmap.ENV`: `prod` instead of `qa`

> **Extending to backend services:** Copy each `envs/qa/values-<service>.yaml` to `envs/prod/values-<service>.yaml` and update `group.name`, `host`, `SPRING_PROFILES_ACTIVE` (or `NODE_ENV`), and `serviceAccount.annotations` the same way.

> **What would be different in a real production environment?**
> In this course, we keep PROD nearly identical to QA for simplicity. In a real production setup, you would typically configure:
> - **Higher replica count** — `replicaCount: 3` or more for high availability
> - **HPA enabled** — `autoscaling.enabled: true` to handle traffic spikes
> - **Stricter resource limits** — larger requests/limits based on production load
> - **Manual ArgoCD sync** — remove `syncPolicy.automated` entirely and require operators to click "Sync" in ArgoCD
> - **Pod Disruption Budgets** — ensure at least N pods are always running during deployments
>
> **Note:** The prod ArgoCD app in this course still has `automated` sync for simplicity. In a real production environment, you would remove automated sync and require manual sync as an additional safety gate.

### Step 2: Create the Prod ArgoCD Application Manifest

Create the pharma-ui PROD ArgoCD app:

```bash
cat > argocd/apps/prod/pharma-ui-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pharma-ui-prod
  namespace: argocd
  labels:
    env: prod
    app: pharma-ui
    managed-by: terraform
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma

  source:
    repoURL: https://github.com/<YOUR_ORG>/zen-gitops.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/prod/values-pharma-ui.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: prod

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 10
EOF
```

> **Extending to backend services:** Each backend service would get its own `argocd/apps/prod/<service>-app.yaml` following the same template.

### Step 3: Apply the Prod ArgoCD App

```bash
kubectl apply -f argocd/apps/prod/pharma-ui-app.yaml
```

**Expected output:**
```
application.argoproj.io/pharma-ui-prod created
```

### Step 4: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add envs/prod/values-pharma-ui.yaml argocd/apps/prod/pharma-ui-app.yaml
git commit -m "feat: add PROD environment values and ArgoCD app for pharma-ui"
git push origin main
```

> **Tag `gitops` repo: `module-8.3-prod-environment`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-8.3-prod-environment -m "Module 8.3: PROD environment configuration for pharma-ui"
> git push origin module-8.3-prod-environment
> ```

---

## 8.4 Promote Pharma-UI from Dev to QA

The CI pipeline you built in Module 4 deploys to DEV automatically. When code merges to `develop`, the CI pipeline:
1. Builds the Docker image and pushes it to ECR
2. Updates `envs/dev/values-pharma-ui.yaml` with the new image tag (direct commit)
3. ArgoCD syncs the DEV deployment

QA promotion is a separate manual step. Once you've validated the DEV deployment, you trigger the `promote-qa-pharma-ui` workflow you created in Module 4.5.

### Step 1: Trigger the QA Promotion Workflow

**Option A — GitHub UI:**

1. Go to your frontend repo: `https://github.com/<YOUR_ORG>/frontend`
2. Click the **Actions** tab
3. In the left sidebar, click **Promote pharma-ui to QA**
4. Click **Run workflow** (top right)
5. Select the `develop` branch
6. Click **Run workflow**

**Option B — GitHub CLI:**

```bash
gh workflow run promote-qa-pharma-ui.yml --repo <YOUR_ORG>/frontend
```

The workflow reads the image tag currently running in DEV (`envs/dev/values-pharma-ui.yaml`) and opens a PR in the gitops repo to promote that exact image to QA.

### Step 2: Find the QA Promotion PR

Navigate to your gitops repository on GitHub:

```
https://github.com/<YOUR_ORG>/gitops/pulls
```

You should see a pull request titled something like:

```
promote(qa): pharma-ui → sha-xxxxxxx
```

Alternatively, use the CLI:

```bash
gh pr list --repo <YOUR_ORG>/gitops
```

**Expected output:**
```
#12  promote(qa): pharma-ui → sha-b8aa312  promote/qa/pharma-ui/sha-b8aa312  main
```

### Step 3: Review the PR

Click on the PR and examine the diff. You should see:

```diff
--- a/envs/qa/values-pharma-ui.yaml
+++ b/envs/qa/values-pharma-ui.yaml
@@ -5,7 +5,7 @@ image:
   repository: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/pharma-ui
-  tag: v1.0.0
+  tag: sha-b8aa312
   pullPolicy: Always
```

> **Why only the image tag changed:** The promotion workflow is designed to change only `image.tag`. Environment-specific configuration (ALB group, host, resource limits) was already set up in Section 8.2. This separation means the PR reviewer can immediately see: "This promotion changes only the image being deployed — no config surprises."

The PR body includes a checklist:
- [ ] DEV smoke test passed
- [ ] No unexpected config changes
- [ ] QA sign-off

### Step 4: Merge the PR

After reviewing, merge the PR. You can do this via the GitHub UI (click **Merge pull request**) or via the CLI:

```bash
gh pr merge <PR_NUMBER> --repo <YOUR_ORG>/gitops --merge
```

### Step 5: Verify QA Deployment

ArgoCD will detect the change within 3 minutes and sync the QA deployment automatically.

```bash
# Wait a moment for ArgoCD to detect the change, then check
kubectl get pods -n qa
```

**Expected output:**
```
NAME                         READY   STATUS    RESTARTS   AGE
pharma-ui-xxxxxxxxx-xxxxx    1/1     Running   0          45s
```

```bash
kubectl get ingress -n qa
```

**Expected output:**
```
NAME        CLASS   HOSTS                  ADDRESS                                      PORTS   AGE
pharma-ui   alb     qa.pharma.internal     k8s-pharmaq-xxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com   80      60s
```

### Step 6: Check ArgoCD UI

Open ArgoCD in your browser. You should see the `pharma-ui-qa` application with:
- **Status:** Synced
- **Health:** Healthy
- **Sync result:** Shows the image tag matching what was in the PR

> **Why is ArgoCD showing Synced even though we just merged?** ArgoCD polls the gitops repo every 3 minutes by default. Once it detects the new commit on `main`, it compares the desired state (the values file) with the live state (what's running in the cluster). Since the image tag changed, ArgoCD triggers a sync — pulling the new image and updating the deployment.

---

## 8.5 Promote Pharma-UI from QA to Prod

Production promotion is intentionally manual. Someone must explicitly decide "this image is ready for production."

### Step 1: Trigger the Promotion Workflow

**Option A — GitHub UI:**

1. Go to your frontend repo: `https://github.com/<YOUR_ORG>/frontend`
2. Click the **Actions** tab
3. In the left sidebar, click **Promote pharma-ui to PROD**
4. Click **Run workflow** (top right)
5. Select the `main` branch (or `develop`)
6. Click **Run workflow**

**Option B — GitHub CLI:**

```bash
gh workflow run promote-prod-pharma-ui.yml --repo <YOUR_ORG>/frontend
```

### Step 2: Understand What the Workflow Does

The `promote-prod-pharma-ui.yml` workflow executes these steps:

1. **Checks out the gitops repo** using the `GITOPS_TOKEN` secret
2. **Reads the image tag from QA** — `yq e '.image.tag' envs/qa/values-pharma-ui.yaml` — this ensures you're promoting exactly what was tested in QA
3. **Validates the PROD values file exists** — fails fast if the file is missing
4. **Creates a branch** named `promote/prod/pharma-ui/<image-tag>`
5. **Updates `envs/prod/values-pharma-ui.yaml`** — changes only `image.tag`
6. **Opens a PR** with a pre-merge checklist

> **Why read the tag from QA instead of accepting it as an input?** Reading from QA guarantees you're promoting the exact image that was tested. If someone passed the tag manually, they could accidentally promote an untested image. The QA values file is the source of truth for "what's proven."

### Step 3: Review the PROD Promotion PR

Navigate to your gitops repository's pull requests. You should see:

```
promote(prod): pharma-ui → sha-b8aa312
```

The PR body contains a structured checklist:

```markdown
## PROD Promotion

Service : pharma-ui
Image   : sha-b8aa312
Promoted from QA by : your-username
Workflow run : https://github.com/<YOUR_ORG>/frontend/actions/runs/12345

## Pre-merge Checklist
- [ ] QA sign-off received
- [ ] Change ticket / CAB approved
- [ ] No unexpected config changes in this diff
- [ ] Runbook link added to change ticket

## Post-merge
ArgoCD app `pharma-prod/pharma-ui` is configured for **manual sync**.
After merging, trigger sync in ArgoCD and monitor the rollout.
```

> **Why this level of ceremony for production?** In a real organization:
> - **QA sign-off** confirms a human verified the application works correctly in QA
> - **Change ticket / CAB approved** means the change went through your organization's change management process
> - **Runbook link** ensures operators know what to do if the deployment fails
> - These items are not enforced by GitHub — they're a communication protocol. The team agrees to check these boxes honestly before merging.

### Step 4: Merge the PR

After completing the checklist review, merge the PR:

```bash
gh pr merge <PR_NUMBER> --repo <YOUR_ORG>/gitops --merge
```

### Step 5: Verify Prod Deployment

ArgoCD syncs PROD automatically (in our course setup):

```bash
kubectl get pods -n prod
```

**Expected output:**
```
NAME                         READY   STATUS    RESTARTS   AGE
pharma-ui-xxxxxxxxx-xxxxx    1/1     Running   0          60s
```

```bash
kubectl get ingress -n prod
```

**Expected output:**
```
NAME        CLASS   HOSTS                   ADDRESS                                       PORTS   AGE
pharma-ui   alb     prod.pharma.internal    k8s-pharmap-xxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com   80      90s
```

Verify in ArgoCD UI that `pharma-ui-prod` shows **Synced** and **Healthy**.

> **Optional exercise — promoting backend services:** This course demonstrates environment promotion end-to-end using pharma-ui only. The exact same pattern applies to all 8 backend services: create QA/Prod values files and ArgoCD apps per service (Modules 8.2/8.3), then trigger `promote-qa.yml` / `promote-prod.yml` with the service dropdown (Module 7.5) and merge the resulting gitops PRs. Try this on your own once you're comfortable with the pharma-ui flow.

---

## 8.6 Rollback Strategy

One of the biggest advantages of GitOps is that rollback is just another Git operation. There is no special `kubectl rollout undo` command, no SSH into servers, no manual intervention in the cluster.

### How Rollback Works in GitOps

When you need to roll back a production deployment:

1. **Find the merge commit** that promoted the bad image
2. **Revert the commit** — this creates a new commit that undoes the image tag change
3. **Push the revert** — ArgoCD detects the change and syncs back to the previous image
4. The old image is still in ECR (the lifecycle policy keeps the last 10 images)

That's it. No cluster access required.

### Step 1: Identify the Bad Deployment

```bash
cd ~/devops/zenpharma/gitops
git log --oneline -10
```

**Expected output:**
```
a1b2c3d (HEAD -> main) Merge pull request #25 from promote/prod/pharma-ui/sha-b8aa312
d4e5f6g promote(prod): pharma-ui → sha-b8aa312
h8i9j0k Merge pull request #20 from promote/qa/pharma-ui/sha-b8aa312
l1m2n3o feat: add PROD environment values and ArgoCD apps
p4q5r6s feat: add QA environment values and ArgoCD apps
...
```

The merge commit `a1b2c3d` is what brought the bad image into prod.

### Step 2: Revert the Merge Commit

```bash
git revert <merge-commit-sha> -m 1
```

> **Why `-m 1`?** When reverting a merge commit, Git needs to know which parent to revert to. `-m 1` means "revert to the first parent" — which is the `main` branch before the merge. This is almost always what you want.

**Expected output:**
```
[main x9y8z7w] Revert "Merge pull request #25 from promote/prod/pharma-ui/sha-b8aa312"
 1 file changed, 1 insertion(+), 1 deletion(-)
```

### Step 3: Push the Revert

```bash
git push origin main
```

ArgoCD will detect the change within 3 minutes and automatically sync the prod namespace back to the previous image.

### Step 4: Verify the Rollback

```bash
# Wait for ArgoCD to sync, then verify
kubectl get pods -n prod
```

The pods should restart with the previous image. You can confirm the image tag:

```bash
kubectl get deployment pharma-ui -n prod -o jsonpath='{.spec.template.spec.containers[0].image}'
```

This should show the previous image tag, not the one you just reverted.

### Why GitOps Rollback Is Superior

| Traditional Rollback | GitOps Rollback |
|---------------------|-----------------|
| `kubectl rollout undo` — requires cluster access | `git revert` — requires only git access |
| No record of who rolled back or why | Full Git history: who, when, why (commit message) |
| Easy to forget which version you rolled back to | The revert commit shows exactly what changed |
| Doesn't work if the issue is in config, only in the image | Works for any change — image tags, config values, resource limits |
| Must be done from a machine with `kubeconfig` access | Can be done from any machine with Git access (even a phone) |

> **Why not just re-promote the old image?** You could, but `git revert` is faster and more explicit. It creates a clear audit trail: "We rolled back this exact change because of incident X." Re-promoting would look like a forward deployment in the Git history, making it harder to trace what happened during an incident review.

---

## 8.7 Course Wrap-Up

You have built a complete DevOps platform from scratch. Let's verify everything is in place and recap what you've accomplished.

### Verify Your Implementation

Your working directory (`~/devops/zenpharma`) should now match the reference implementation (`~/dpp/dpp-assignment3`). Verify the key pieces:

```bash
# Check all repos exist
ls ~/devops/zenpharma/
```

**Expected output:**
```
backend  frontend  gitops  infra
```

```bash
# Check gitops has all three environments
ls ~/devops/zenpharma/gitops/envs/
```

**Expected output:**
```
dev  prod  qa
```

```bash
# Check ArgoCD apps for all environments
ls ~/devops/zenpharma/gitops/argocd/apps/
```

**Expected output:**
```
dev  prod  qa
```

```bash
# Verify pharma-ui pods are running in all three environments
kubectl get pods -n dev -l app.kubernetes.io/name=pharma-service
kubectl get pods -n qa -l app.kubernetes.io/name=pharma-service
kubectl get pods -n prod -l app.kubernetes.io/name=pharma-service
```

Pharma-ui should be `Running` in all three namespaces. (Backend services will only show up if you completed the optional exercise in section 8.6.)

### What You Built Across 8 Modules

| Module | What You Built |
|--------|---------------|
| **Module 1** | AWS credentials, GitHub organization with 4 repos, Terraform modules (VPC, EKS, ECR, RDS, IAM, Secrets Manager) |
| **Module 2** | GitHub Actions workflows for Terraform (plan on PR, apply on merge), remote state with S3/DynamoDB |
| **Module 3** | EKS cluster bootstrap — AWS Load Balancer Controller, External Secrets Operator, ArgoCD, namespaces |
| **Module 4** | Frontend Docker/CI pipeline — multi-stage Dockerfile, lint/test/security/build/push, automated DEV deployment |
| **Module 5** | GitOps repository — shared Helm chart, environment-specific values, ArgoCD Application manifests |
| **Module 6** | First deployment — pharma-ui running in DEV via ArgoCD, end-to-end GitOps flow verified |
| **Module 7** | Backend microservices — Dockerfiles, CI pipelines, ArgoCD apps for all 8 backend services in DEV |
| **Module 8** | Environment promotion — QA and PROD environments, promotion workflows with approval gates, rollback strategy |

### Key Practices Implemented

**Infrastructure as Code**
- Terraform with modular structure (6 modules)
- Remote state in S3 with DynamoDB locking
- CI/CD for infrastructure changes (plan on PR, apply on merge)

**CI/CD Pipelines**
- GitHub Actions for all repos
- Security scanning: SonarCloud (SAST + code quality), Trivy (container images), npm audit / OWASP Dep Check (dependencies)
- Automated image signing with Cosign
- Manual promotion workflows (QA and PROD via workflow_dispatch)

**GitOps**
- ArgoCD as the deployment engine
- Single Helm chart with per-environment values
- Git as the single source of truth for cluster state
- Automated sync with self-heal and pruning

**Secrets Management**
- AWS Secrets Manager as the source of truth
- External Secrets Operator syncs secrets into Kubernetes
- IRSA (IAM Roles for Service Accounts) for secure access
- No secrets in Git — ever

**Environment Promotion**
- Three environments: Dev, QA, Prod
- Increasing gates at each stage: direct commit → manual trigger + PR → manual trigger + PR
- Rollback via `git revert` (not `kubectl rollout undo`)

**Security**
- OIDC federation (no long-lived AWS credentials in GitHub)
- IRSA for pod-level IAM permissions
- Non-root containers with read-only root filesystems
- Container image scanning and signing
- Security-hardened Helm chart defaults (dropped capabilities, non-root user)

### What's Not Covered (Worth Exploring Next)

This course focused on building the deployment platform. Here are areas to explore next:

- **Monitoring and Observability** — Prometheus for metrics, Grafana for dashboards, alerting rules
- **Log Aggregation** — CloudWatch Logs, ELK stack (Elasticsearch, Logstash, Kibana), or Loki
- **DNS and SSL Certificates** — Route 53 for DNS, ACM (AWS Certificate Manager) for TLS certificates, external-dns controller
- **Cost Optimization** — Spot instances for worker nodes, Karpenter for intelligent autoscaling, right-sizing resource requests
- **Multi-Cluster Setup** — Separate EKS clusters for QA and Prod (better isolation, independent scaling)
- **Service Mesh** — Istio or Linkerd for mutual TLS, traffic splitting, canary deployments
- **Database Migrations** — Flyway or Liquibase integrated into the CI pipeline
- **Feature Flags** — LaunchDarkly or Unleash for gradual feature rollouts without redeployment

---

## Module 8 Summary

| What We Built | Details |
|--------------|---------|
| **Promotion flow** | Three-stage pipeline: Dev (auto deploy) → QA (manual trigger + PR) → Prod (manual trigger + PR) |
| **QA environment** | Values files + ArgoCD apps for all 9 services |
| **Prod environment** | Values files + ArgoCD apps for all 8 services |
| **Frontend promotion** | Dedicated `promote-prod-pharma-ui.yml` workflow |
| **Backend promotion** | Consolidated `promote-prod.yml` workflow with service dropdown |
| **Rollback strategy** | `git revert` on the merge commit — ArgoCD auto-syncs the rollback |

| Tag | Repos |
|-----|-------|
| `module-8.2-qa-environment` | gitops |
| `module-8.3-prod-environment` | gitops |

---

Congratulations — you have built a production-grade DevOps platform from scratch.

You started with an empty AWS account and four empty GitHub repositories. Over 8 modules, you provisioned cloud infrastructure with Terraform, containerized applications with Docker, built CI/CD pipelines with GitHub Actions, deployed with ArgoCD and GitOps, managed secrets securely with AWS Secrets Manager and ESO, and set up a multi-environment promotion pipeline with appropriate approval gates at every stage.

Every piece of infrastructure is defined in code. Every deployment is a Git commit. Every promotion is auditable. Every rollback is a `git revert` away.

This is what modern DevOps looks like.
