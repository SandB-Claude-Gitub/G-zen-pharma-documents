# Module 6 — Deploying Pharma-UI to Dev

> Bring everything together: run the CI pipeline, deploy pharma-ui to the DEV environment via ArgoCD, and verify it end-to-end in a browser.
> Estimated time: 1-2 hours.

---

## 6.1 Pre-Deployment Checklist

In Modules 1-5, you built every piece of the pipeline independently: infrastructure, CI workflows, Helm charts, ArgoCD configuration, and External Secrets. Before we deploy for the first time, we need to verify that **all** of those pieces are in place and healthy.

### Step 0: Recreate Infrastructure and Re-Bootstrap the Cluster

If you destroyed infrastructure at the end of Module 3, you need to bring everything back. Terraform recreates the AWS resources, but everything **inside** the cluster (Helm charts, ArgoCD config, External Secrets) must be reinstalled — those are not managed by Terraform.

**Recreate infrastructure:**

```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform apply
```

Or use GitHub Actions (`workflow_dispatch` → action: `apply`).

**Update kubectl context:**

```bash
aws eks update-kubeconfig --name pharma-dev-cluster --region us-east-1
```

**Re-run all three bootstrap scripts from Module 3:**

```bash
cd ~/devops/zenpharma
python3 infra/scripts/01_install_prerequisites.py   # ALB Controller, ArgoCD, ESO
python3 infra/scripts/02_bootstrap_argocd.py         # Register gitops repo, AppProject
python3 infra/scripts/03_setup_external_secrets.py   # ClusterSecretStore, ExternalSecrets
```

> **Why?** Terraform only manages AWS resources (VPC, EKS, RDS, IAM). The Helm charts (ALB Controller, ArgoCD, ESO), Kubernetes Secrets, ClusterSecretStore, and ExternalSecrets all live **inside** the cluster. When the cluster is destroyed and recreated, these are gone. The bootstrap scripts reinstall them.

If you did **not** destroy infrastructure between modules, skip Step 0 and proceed to the checks below.

---

Work through each check below. If any check fails, go back to the relevant module and fix it before proceeding.

### Check 1: EKS Cluster Is Running

```bash
kubectl get nodes
```

**Expected output:**
```
NAME                             STATUS   ROLES    AGE   VERSION
ip-10-0-1-xxx.ec2.internal      Ready    <none>   1h    v1.31.x
ip-10-0-2-xxx.ec2.internal      Ready    <none>   1h    v1.31.x
```

You should see at least 2 nodes in `Ready` status. If not, check that your EKS cluster is running in the AWS Console (EKS → Clusters) and that your `kubectl` context is set correctly:

```bash
aws eks update-kubeconfig --name pharma-dev-cluster --region us-east-1
```

### Check 2: Cluster Prerequisites (ALB Controller, ArgoCD, ESO)

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
kubectl get pods -n argocd
kubectl get pods -n external-secrets
```

**Expected output (summarized):**
```
# ALB Controller — 2 pods Running
aws-load-balancer-controller-xxxxx   1/1   Running   0   1h
aws-load-balancer-controller-xxxxx   1/1   Running   0   1h

# ArgoCD — 5+ pods Running
argocd-application-controller-0         1/1   Running   0   1h
argocd-repo-server-xxxxx               1/1   Running   0   1h
argocd-server-xxxxx                    1/1   Running   0   1h
...

# ESO — 1+ pods Running
external-secrets-xxxxx                 1/1   Running   0   1h
external-secrets-cert-controller-xxx   1/1   Running   0   1h
external-secrets-webhook-xxxxx         1/1   Running   0   1h
```

All pods must be `Running` with `1/1` ready. If any are in `CrashLoopBackOff` or `Pending`, revisit Module 3.

### Check 3: External Secrets Are Synced

```bash
kubectl get externalsecret -n dev
```

**Expected output:**
```
NAME             STORE              REFRESH INTERVAL   STATUS         READY
db-credentials   aws-secret-store   1h                 SecretSynced   True
```

The `STATUS` column must say `SecretSynced` and `READY` must be `True`. If not, check the ClusterSecretStore and IAM role (Module 3.5).

### Check 4: GitOps Repo Has All Required Files AND Is Pushed to GitHub

The CI pipeline and ArgoCD both read from the **remote** gitops repo on GitHub — not your local copy. A missing file locally means ArgoCD won't render the resource — for example, a missing `service.yaml` template means no Service gets created, and the ALB returns 503.

**Run this single command to verify all required files exist:**

```bash
cd ~/devops/zenpharma/gitops

echo "=== Helm Chart ==="
for f in helm-charts/Chart.yaml \
         helm-charts/values.yaml \
         helm-charts/templates/_helpers.tpl \
         helm-charts/templates/deployment.yaml \
         helm-charts/templates/service.yaml \
         helm-charts/templates/ingress.yaml \
         helm-charts/templates/configmap.yaml \
         helm-charts/templates/serviceaccount.yaml \
         helm-charts/templates/hpa.yaml; do
  [ -f "$f" ] && echo "  ✓ $f" || echo "  ✗ MISSING: $f"
done

echo ""
echo "=== Dev Environment ==="
for f in envs/dev/values-pharma-ui.yaml \
         argocd/apps/dev/pharma-ui-app.yaml \
         argocd/projects/pharma-project.yaml; do
  [ -f "$f" ] && echo "  ✓ $f" || echo "  ✗ MISSING: $f"
done
```

**Expected output — all checkmarks, no MISSING:**
```
=== Helm Chart ===
  ✓ helm-charts/Chart.yaml
  ✓ helm-charts/values.yaml
  ✓ helm-charts/templates/_helpers.tpl
  ✓ helm-charts/templates/deployment.yaml
  ✓ helm-charts/templates/service.yaml
  ✓ helm-charts/templates/ingress.yaml
  ✓ helm-charts/templates/configmap.yaml
  ✓ helm-charts/templates/serviceaccount.yaml
  ✓ helm-charts/templates/hpa.yaml

=== Dev Environment ===
  ✓ envs/dev/values-pharma-ui.yaml
  ✓ argocd/apps/dev/pharma-ui-app.yaml
  ✓ argocd/projects/pharma-project.yaml
```

If any file shows `✗ MISSING`, go back to Module 5 and create it. Every missing template means a missing Kubernetes resource — no `service.yaml` template = no Service = ALB returns 503.

**Verify everything is pushed to GitHub:**

```bash
git status
```

If you see uncommitted or untracked files, commit and push them now:

```bash
git add .
git commit -m "feat: add all gitops content (helm charts, values, argocd apps)"
git push origin main
```

> **Why does this matter?** ArgoCD clones the gitops repo from **GitHub** (not your local copy) to render the Helm chart. If a template file exists locally but wasn't pushed, ArgoCD won't see it — and the corresponding Kubernetes resource simply won't be created. This is a silent failure — ArgoCD reports "Synced" and "Healthy" because it successfully rendered everything it *could* see.

### Check 5: Frontend Repo Has CI Prerequisites

Verify the frontend repository has:

```bash
ls ~/devops/zenpharma/frontend/Dockerfile          # Docker build file
ls ~/devops/zenpharma/frontend/.github/workflows/ci-pharma-ui.yml  # CI workflow
```

Also verify that the GitHub Actions secrets and variables are configured. Go to your frontend repo on GitHub → **Settings** → **Secrets and variables** → **Actions**. You should see:

**Secrets:**

| Secret | Purpose |
|--------|---------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account number (for OIDC role ARN) |
| `GITOPS_TOKEN` | PAT with write access to the gitops repo |
| `SONAR_TOKEN` | SonarCloud authentication token |

**Variables:**

| Variable | Purpose |
|----------|---------|
| `GITOPS_REPO` | `<your-username>/gitops` (owner/repo format) |
| `SONAR_ORG` | SonarCloud organization key |
| `SONAR_PROJECT_KEY_FRONTEND` | SonarCloud project key for frontend |

> **Note:** The frontend CI uses OIDC federation to authenticate with AWS — no `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` needed. The IAM role `pharma-dev-github-actions-role` (created in Module 1) handles authentication automatically.

> **Why a full checklist before deploying?** A first deployment touches every component in the pipeline. If something is misconfigured, the error message may point to the wrong place — ArgoCD might report "sync failed" when the real problem is a missing ExternalSecret. Checking each piece individually now saves hours of debugging later.

---

## 6.2 Initialize Database Schemas

The backend microservices expect their PostgreSQL schemas to exist before they start. We initialize these schemas **once** as a bootstrap step — after this, each microservice manages its own tables via migrations on startup.

### Step 1: Review the Schema SQL

The schema file lives in the gitops repo at `db-init/01-schemas.sql`:

```sql
-- Create schemas for each microservice (schema-per-service pattern)
CREATE SCHEMA IF NOT EXISTS auth;
CREATE SCHEMA IF NOT EXISTS drug_catalog;
CREATE SCHEMA IF NOT EXISTS inventory;
CREATE SCHEMA IF NOT EXISTS manufacturing;
CREATE SCHEMA IF NOT EXISTS quality_control;
CREATE SCHEMA IF NOT EXISTS supplier;
CREATE SCHEMA IF NOT EXISTS distribution;
CREATE SCHEMA IF NOT EXISTS reporting;

-- Grant permissions
GRANT ALL PRIVILEGES ON SCHEMA auth TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA drug_catalog TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA inventory TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA manufacturing TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA quality_control TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA supplier TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA distribution TO pharmaadmin;
GRANT ALL PRIVILEGES ON SCHEMA reporting TO pharmaadmin;
```

> **Why schema-per-service?** Each microservice gets its own PostgreSQL schema within a shared database. This gives you logical isolation (services cannot accidentally read each other's tables) without the operational overhead of running 8 separate database instances. In production, you might separate high-traffic services into their own databases, but for development this is the right tradeoff.

### Step 2: Get the RDS Endpoint

You need the RDS hostname to connect. There are three ways to find it:

**Option A: Terraform output (must run from the envs/dev directory):**
```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform output rds_endpoint
```

**Option B: AWS CLI:**
```bash
aws rds describe-db-instances \
  --query "DBInstances[?DBInstanceIdentifier=='pharma-dev-postgres'].Endpoint.Address" \
  --output text
```

**Option C: AWS Console:**
1. Go to https://console.aws.amazon.com/rds/
2. Click **Databases** → click on `pharma-dev-postgres`
3. Under **Connectivity & security** → copy the **Endpoint**

The endpoint looks like: `pharma-dev-postgres.xxxxxxxxxx.us-east-1.rds.amazonaws.com`

### Step 3: Run the Schema SQL (Manual)

> **Why run from inside the cluster?** The RDS instance is in a private subnet — it is not accessible from the internet. We launch a temporary pod inside the EKS cluster so it can reach the database directly.

Run this single command — it starts a temporary pod, connects to RDS, and drops you into a `psql` prompt:

```bash
kubectl run pg-client --rm -it --restart=Never --image=postgres:17 -n dev \
  --env='PGPASSWORD=<your-db-password>' -- \
  psql -h <rds-endpoint> -U pharmaadmin -d pharmadb
```

Replace `<your-db-password>` and `<rds-endpoint>` with your values.

> **Important:** Use **single quotes** around `--env='PGPASSWORD=...'`. If your password contains `!` or `$`, double quotes will cause zsh to interpret them as special characters.

Once you see the `pharmadb=>` prompt, paste the SQL from Step 1 and press Enter. Then verify with `\dn` and exit with `\q`. The pod auto-deletes when you exit.

### Step 3 (Alternative): Run with a Script

If you prefer not to paste SQL manually, use this script instead. Create `infra/scripts/init-database.sh`:

```bash
#!/bin/bash
# Initialize database schemas for ZenPharma microservices
set -e

# ── Collect inputs ──────────────────────────────────────────────────────
echo ""
echo "=== Zen Pharma — Database Schema Initializer ==="
echo ""

read -p "  RDS endpoint: " RDS_ENDPOINT
[ -z "$RDS_ENDPOINT" ] && echo "Error: RDS endpoint is required." && exit 1

read -sp "  Database password: " DB_PASSWORD && echo ""
[ -z "$DB_PASSWORD" ] && echo "Error: Password is required." && exit 1

NAMESPACE="dev"
SQL_FILE="$(dirname "$0")/../../gitops/db-init/01-schemas.sql"

if [ ! -f "$SQL_FILE" ]; then
  echo "Error: SQL file not found at $SQL_FILE"
  exit 1
fi

echo ""
echo "  RDS endpoint : $RDS_ENDPOINT"
echo "  Namespace    : $NAMESPACE"
echo "  SQL file     : $SQL_FILE"
echo ""
read -p "  Proceed? [Y/n]: " CONFIRM
[ "${CONFIRM:-Y}" != "Y" ] && [ "${CONFIRM:-Y}" != "y" ] && echo "Aborted." && exit 0

# ── Clean up any leftover pod ───────────────────────────────────────────
kubectl delete pod pg-client -n "$NAMESPACE" --ignore-not-found > /dev/null 2>&1

# ── Start temporary postgres pod ────────────────────────────────────────
echo ""
echo "Starting temporary PostgreSQL pod..."
kubectl run pg-client --restart=Never --image=postgres:17 -n "$NAMESPACE" \
  --env="PGPASSWORD=$DB_PASSWORD" -- sleep 3600 > /dev/null
kubectl wait --for=condition=Ready pod/pg-client -n "$NAMESPACE" --timeout=60s > /dev/null
echo "Pod ready."

# ── Copy and run SQL ────────────────────────────────────────────────────
echo "Copying SQL file and running..."
kubectl cp "$SQL_FILE" pg-client:/tmp/01-schemas.sql -n "$NAMESPACE"
kubectl exec pg-client -n "$NAMESPACE" -- psql \
  -h "$RDS_ENDPOINT" -U pharmaadmin -d pharmadb \
  -f /tmp/01-schemas.sql

# ── Verify ──────────────────────────────────────────────────────────────
echo ""
echo "Verifying schemas..."
kubectl exec pg-client -n "$NAMESPACE" -- psql \
  -h "$RDS_ENDPOINT" -U pharmaadmin -d pharmadb \
  -c '\dn'

# ── Clean up ────────────────────────────────────────────────────────────
kubectl delete pod pg-client -n "$NAMESPACE" > /dev/null
echo ""
echo "Done. Database schemas initialized."
```

Make it executable and run:

```bash
chmod +x infra/scripts/init-database.sh
./infra/scripts/init-database.sh
```

The script handles everything — starts the pod, copies the SQL, runs it, verifies, and cleans up.

### Step 4: Verify Schemas Were Created

Whether you used the manual or script approach, you should see this output:

```
        List of schemas
       Name        |    Owner
-------------------+-------------
 auth              | pharmaadmin
 distribution      | pharmaadmin
 drug_catalog      | pharmaadmin
 inventory         | pharmaadmin
 manufacturing     | pharmaadmin
 public            | pg_database_owner
 quality_control   | pharmaadmin
 reporting         | pharmaadmin
 supplier          | pharmaadmin
(9 rows)
```

All 8 custom schemas plus the default `public` schema.

> **This is a one-time bootstrap step.** You only run this SQL once per environment. When backend microservices start for the first time (Module 7), they will create their own tables within these schemas using their built-in migration tools (Flyway for Java services, Sequelize for Node.js).

---

## 6.3 Run the Pharma-UI CI Pipeline

Now we trigger the CI pipeline that builds the pharma-ui Docker image, pushes it to ECR, and updates the gitops repo with the new image tag.

### Trigger the Pipeline

Push a commit to the `develop` branch to trigger the CI pipeline:

```bash
cd ~/devops/zenpharma/frontend
git checkout develop
git commit --allow-empty -m "trigger: initial dev deployment"
git push origin develop
```

### Watch the Pipeline

Go to your frontend repo on GitHub → **Actions** tab. You should see the `CI/CD — pharma-ui` workflow running.

Click on the running workflow to see each job as it executes:

| Job | What It Does |
|-----|-------------|
| **Lint** | Runs ESLint on the source code |
| **Test** | Runs Jest unit tests with coverage |
| **SonarCloud** | SAST scan + code quality analysis |
| **Build** | Creates the production React bundle |
| **Docker Build & Push** | Builds the Docker image, scans with Trivy, pushes to ECR |
| **Deploy DEV** | Updates `image.tag` in gitops repo → ArgoCD auto-syncs |

The pipeline typically takes 3-5 minutes to complete.

### Script Approach

The `04_run_pipeline.py` script automates pipeline triggering and monitoring:

```bash
cd ~/devops/zenpharma
python3 infra/scripts/04_run_pipeline.py
```

The script will ask for:

| Input | Description |
|-------|-------------|
| **GitHub org** | Your GitHub username or organization that owns the repos |
| **Branch** | Branch to build (default: `develop`) |
| **Service selection** | Choose `F` for Frontend only |

The script uses the `gh` CLI to trigger the workflow via `gh workflow run`, then polls for completion every 30 seconds (up to 30 minutes). It displays real-time status updates and a summary when complete.

> **Why use a script?** The manual approach gives you a clear understanding of what happens. The script is useful for subsequent deployments when you want to trigger and monitor without switching to the browser. Both approaches do exactly the same thing — the script just wraps the `gh` CLI commands and adds polling.

### Verify the Pipeline Results

After the pipeline completes successfully, verify two things:

**1. ECR has the image:**

Go to AWS Console → ECR → Repositories → `pharma-ui`. You should see a new image with a tag like `sha-abc1234`.

Or verify via CLI:

```bash
aws ecr describe-images --repository-name pharma-ui --region us-east-1 \
  --query 'imageDetails | sort_by(@, &imagePushedAt) | [-1].imageTags'
```

**Expected output:**
```json
[
    "sha-abc1234"
]
```

**2. GitOps repo has the updated image tag:**

```bash
cd ~/devops/zenpharma/gitops
git pull origin main
cat envs/dev/values-pharma-ui.yaml | grep "tag:"
```

**Expected output:**
```yaml
  tag: sha-abc1234
```

The tag should match the SHA from the CI pipeline. If it still shows the old tag (e.g., `sha-b8aa312`), the CI pipeline's "Update GitOps" step may have failed — check the workflow logs for errors.

---

## 6.4 Deploy Pharma-UI via ArgoCD

The image is in ECR and the gitops repo has the correct tag. Now we tell ArgoCD to deploy it.

### Step 1: Update Placeholder URLs Before Applying

Before applying, make sure both the AppProject and Application manifest have your **actual GitHub URL** — not the `<your-username>` placeholder.

Check and update `argocd/apps/dev/pharma-ui-app.yaml`:

```bash
grep repoURL ~/devops/zenpharma/gitops/argocd/apps/dev/pharma-ui-app.yaml
```

If it shows `<your-username>`, update it to your actual org name (e.g., `zenpharma`):

```yaml
source:
  repoURL: https://github.com/zenpharma/gitops.git   # ← your actual org name
```

Also verify the AppProject matches:

```bash
kubectl get appproject pharma -n argocd -o jsonpath='{.spec.sourceRepos}' && echo ""
```

> **Important:** The `repoURL` in the Application manifest must **exactly match** one of the URLs in the AppProject's `sourceRepos`. If they don't match — even by one character — ArgoCD will reject the Application with `not permitted in project`. This is the most common first-deployment error.

If you updated the file, commit and push:

```bash
cd ~/devops/zenpharma/gitops
git add .
git commit -m "fix: update repoURL placeholders with actual GitHub org"
git push origin main
```

### Step 2: Apply the ArgoCD Application Manifest

```bash
kubectl apply -f ~/devops/zenpharma/gitops/argocd/apps/dev/pharma-ui-app.yaml
```

**Expected output:**
```
application.argoproj.io/pharma-ui-dev created
```

> **What just happened?** You created an ArgoCD `Application` resource in the `argocd` namespace. This tells ArgoCD: "Watch the gitops repo, render the Helm chart at `helm-charts/` using the values file at `envs/dev/values-pharma-ui.yaml`, and apply the resulting Kubernetes manifests to the `dev` namespace." Because the sync policy is set to `automated` with `selfHeal: true`, ArgoCD will start syncing immediately.

Watch the sync progress:

```bash
kubectl get application pharma-ui-dev -n argocd -w
```

**Expected output (evolves over ~30 seconds):**
```
NAME             SYNC STATUS   HEALTH STATUS   
pharma-ui-dev    OutOfSync     Missing         
pharma-ui-dev    Synced        Progressing     
pharma-ui-dev    Synced        Healthy         
```

Press `Ctrl+C` once you see `Synced` and `Healthy`.

Here is what ArgoCD does during the sync:

1. **Clones** the gitops repository
2. **Reads** the Application spec: source path (`helm-charts/`), values file (`envs/dev/values-pharma-ui.yaml`)
3. **Renders** the Helm chart — runs `helm template` with the values file to produce raw Kubernetes YAML
4. **Compares** the rendered manifests to what currently exists in the `dev` namespace
5. **Applies** the diff — creates the Deployment, Service, Ingress, ConfigMap, and ServiceAccount
6. **Monitors** the rollout until all resources are healthy

### Script Approach

The `05_deploy_services.py` script applies ArgoCD Application manifests and polls for sync status:

```bash
cd ~/devops/zenpharma
python3 infra/scripts/05_deploy_services.py
```

The script will ask for:

| Input | Description |
|-------|-------------|
| **Target environment** | Select `dev` |
| **GitHub username** | Your GitHub username (replaces placeholder in the app manifest) |
| **Service selection** | Choose `F` for Frontend only |

The script reads the Application YAML from `gitops/argocd/apps/dev/pharma-ui-app.yaml`, replaces the `your-github-username` placeholder with your actual GitHub username, and applies it via `kubectl apply`. It then polls every 15 seconds (up to 10 minutes) until ArgoCD reports `Synced` and `Healthy`.

> **Why use a script?** The manual approach is a single `kubectl apply` — very simple. The script adds value when deploying multiple services at once (Module 7), where it applies manifests in dependency order and monitors all of them in parallel.

---

## 6.5 Verify the Deployment

Now we confirm that everything is running correctly. We check at three levels: ArgoCD UI, Kubernetes resources, and ALB accessibility.

### Check ArgoCD UI

Start a port-forward to the ArgoCD server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open your browser and navigate to: `https://localhost:8080`

> **Note:** Your browser will warn about an untrusted certificate — this is expected for a self-signed cert. Click "Advanced" and proceed.

Log in with:
- **Username:** `admin`
- **Password:** Retrieve it with:
  ```bash
  kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
  ```

Once logged in, you should see the `pharma-ui-dev` application tile showing:

- **Sync Status:** `Synced` (green checkmark)
- **Health Status:** `Healthy` (green heart)

Click on the `pharma-ui-dev` tile to see the resource tree. You should see these resources:

| Resource | Kind | Status |
|----------|------|--------|
| `pharma-ui` | Deployment | Healthy |
| `pharma-ui` | Service | Healthy |
| `pharma-ui` | Ingress | Healthy |
| `pharma-ui` | ConfigMap | Synced |
| `pharma-ui` | ServiceAccount | Synced |
| `pharma-ui-xxxxx-xxxxx` | Pod | Running |
| `pharma-ui-xxxxx` | ReplicaSet | Healthy |

### Check Kubernetes Resources

Open a new terminal (keep the port-forward running) and verify the Kubernetes objects:

**Pods:**
```bash
kubectl get pods -n dev
```

**Expected output:**
```
NAME                          READY   STATUS    RESTARTS   AGE
pharma-ui-xxxxx-xxxxx         1/1     Running   0          2m
```

The pod should be `1/1 Running` with `0` restarts.

**Services:**
```bash
kubectl get svc -n dev
```

**Expected output:**
```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
pharma-ui    ClusterIP   172.20.x.x     <none>        80/TCP    2m
```

The service type is `ClusterIP` (internal only) — external access comes through the Ingress/ALB.

**Ingress:**
```bash
kubectl get ingress -n dev
```

**Expected output:**
```
NAME         CLASS   HOSTS   ADDRESS                                              PORTS   AGE
pharma-ui    alb     *       k8s-pharmadev-xxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com   80      2m
```

The `ADDRESS` column shows the ALB hostname. If it is blank, the AWS Load Balancer Controller is still provisioning the ALB — wait 2-3 minutes and check again.

**Ingress details:**
```bash
kubectl describe ingress pharma-ui -n dev
```

Look for these key annotations in the output:
- `alb.ingress.kubernetes.io/scheme: internet-facing` — the ALB is publicly accessible
- `alb.ingress.kubernetes.io/target-type: ip` — routes directly to pod IPs
- `alb.ingress.kubernetes.io/group.name: pharma-dev` — all services share the same ALB

### Check the ALB

Get the ALB hostname:

```bash
kubectl get ingress pharma-ui -n dev -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Expected output:**
```
k8s-pharmadev-xxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com
```

> **Why does the ALB take 2-3 minutes?** The AWS Load Balancer Controller creates an Application Load Balancer in AWS, configures target groups, registers targets, and waits for health checks to pass. This involves several AWS API calls and provisioning steps.

You can also verify the ALB is reachable via `curl`:

```bash
curl -s -o /dev/null -w "%{http_code}" http://$(kubectl get ingress pharma-ui -n dev -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/
```

**Expected output:**
```
200
```

### Automated Verification

The `06_verify_deployment.py` script runs all of the checks above (and more) in a single pass:

```bash
cd ~/devops/zenpharma
python3 infra/scripts/06_verify_deployment.py
```

Select `dev` when prompted for the target environment. The script runs 5 checks:

| Check | What It Verifies |
|-------|-----------------|
| **1. Kubernetes Pods** | All pods in the `dev` namespace are Running and Ready (waits up to 5 minutes) |
| **2. ArgoCD Applications** | All ArgoCD Application resources are `Synced` and `Healthy` |
| **3. External Secrets** | All ExternalSecret resources show `SecretSynced` and `Ready=True` |
| **4. Services/Ingress** | Services and Ingress resources exist, ALB hostname is provisioned |
| **5. HTTP Endpoints** | HTTP health checks against the ALB for each service (pharma-ui at `/`) |

If all checks pass, you will see:

```
============================================
  ALL CHECKS PASSED

  Application URL : http://k8s-pharmadev-xxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com/
  ArgoCD UI       : https://localhost:8080
                    (kubectl port-forward svc/argocd-server -n argocd 8080:443)
============================================
```

> **Why use a script?** The manual checks above teach you what to look for and how to interpret the output. The verification script is useful for quick sanity checks — especially after recreating the cluster or redeploying between modules.

---

## 6.6 Access Pharma-UI from the Browser

Copy the ALB hostname from the previous step and open it in your browser:

```
http://<alb-hostname>/
```

For example:
```
http://k8s-pharmadev-xxxxxxxx-xxxxxxxxxx.us-east-1.elb.amazonaws.com/
```

You should see the **ZenPharma dashboard** — the pharma-ui React application served by Nginx inside the Kubernetes pod.

> **Note:** Backend APIs will not work yet. The UI will load and display the dashboard layout, but any data calls (drug catalog, inventory, suppliers, etc.) will fail with network errors. This is expected — we have not deployed the backend microservices yet. That comes in Module 7.

### What You Just Accomplished

Take a moment to appreciate the full pipeline you built across Modules 1-6:

```
Code Push (frontend repo)
    → GitHub Actions CI (build, scan, push)
        → Docker Image in ECR (sha-tagged)
            → GitOps Repo Updated (image.tag in values file)
                → ArgoCD Detects Change (polls every 3 minutes)
                    → Helm Chart Rendered (with dev values)
                        → Kubernetes Manifests Applied (Deployment, Service, Ingress)
                            → Pod Running in EKS (Nginx serving React app)
                                → ALB Routes Traffic (internet-facing)
                                    → Browser Displays Dashboard
```

Every step is automated. The only manual step was the initial `kubectl apply` of the ArgoCD Application manifest — and even that could be automated. From now on, every push to the `develop` branch will trigger this entire chain automatically.

### Troubleshooting

If something goes wrong, work through these common issues:

**ALB hostname not appearing in Ingress:**
```bash
# Check the ALB Controller logs:
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```
Common causes: IAM role misconfiguration, missing subnet tags, security group issues. The logs will show the specific AWS API error.

**Pods in CrashLoopBackOff:**
```bash
# Check pod logs for error messages:
kubectl logs -n dev deployment/pharma-ui
# Check the previous container's logs if it crashed:
kubectl logs -n dev deployment/pharma-ui --previous
# Check pod events:
kubectl describe pod -n dev -l app.kubernetes.io/name=pharma-ui
```
Common causes: incorrect image tag (image not found in ECR), container port mismatch, readiness probe failing.

**ArgoCD shows "OutOfSync":**
```bash
# Check the sync error:
kubectl get application pharma-ui-dev -n argocd -o jsonpath='{.status.conditions[*].message}'
```
Common causes: Helm chart syntax error, values file references a missing key, gitops repo URL is wrong or inaccessible (check the ArgoCD repo credentials from Module 3).

**502 Bad Gateway from ALB:**

This usually means the ALB is provisioned but the target (pod) is not yet ready. Wait 30-60 seconds and refresh the browser. If it persists:
```bash
# Check that the pod is Ready:
kubectl get pods -n dev
# Check the readiness probe:
kubectl describe pod -n dev -l app.kubernetes.io/name=pharma-ui | grep -A5 "Readiness"
```

**Ingress shows no address after 5+ minutes:**
```bash
# Verify the ALB Controller is running:
kubectl get pods -n kube-system | grep aws-load-balancer
# Check for Ingress events:
kubectl describe ingress pharma-ui -n dev
```
Look for events like `Successfully reconciled` (good) or error messages about security groups or subnets.

---

## Module 6 Summary

| What We Did | Details |
|-------------|---------|
| **Pre-deployment checklist** | Verified EKS nodes, cluster prerequisites, External Secrets, GitOps files, and CI prerequisites |
| **Database schemas** | Initialized 8 PostgreSQL schemas (one per microservice) via a temporary pg-client pod |
| **CI pipeline** | Triggered pharma-ui build: Docker image pushed to ECR with SHA tag, gitops repo updated |
| **ArgoCD deployment** | Applied the pharma-ui Application manifest; ArgoCD synced Helm chart to the `dev` namespace |
| **Verification** | Confirmed pods Running, ArgoCD Synced/Healthy, Ingress provisioned, ALB reachable, UI in browser |
| **End-to-end pipeline** | Code push → CI → ECR → GitOps → ArgoCD → Pod → ALB → Browser — fully automated |

> **Next:** [Module 7 — Dockerize & Deploy Backend Microservices](MODULE-7-DOCKERIZE-AND-DEPLOY-BACKEND-MICROSERVICES.md)
