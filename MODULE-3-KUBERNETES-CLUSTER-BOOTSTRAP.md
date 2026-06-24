# Module 3 — Kubernetes Cluster Bootstrap

> Recreate infrastructure and install cluster prerequisites: AWS Load Balancer Controller, ArgoCD, and External Secrets Operator.
> Estimated time: 1.5–2 hours.

---

## 3.1 Recreate Infrastructure

At the end of Module 2, we destroyed all AWS infrastructure to avoid unnecessary costs. Now we need it back. This section brings the EKS cluster, VPC, RDS, ECR, IAM roles, and Secrets Manager resources back online.

> **Why did we destroy and recreate?**
> - **Cost savings:** An EKS cluster costs ~$0.10/hour ($72/month) plus EC2 node costs. Leaving it running between course sessions wastes money.
> - **Clean state:** Recreating from scratch proves that our Terraform code is fully reproducible. If you can destroy and recreate without manual fixes, your infrastructure-as-code is truly declarative.
> - **Course progression:** Module 1 taught local Terraform. Module 2 taught CI/CD-driven Terraform. Now we start fresh to focus on what runs *inside* the cluster.

### Step 1: Re-Run Terraform Apply

You have two options — pick whichever you used in Module 2.

**Option A: GitHub Actions (recommended)**

1. Go to your `infra` repository on GitHub
2. Click **Actions** tab
3. Select the **Terraform Apply** workflow (or whatever you named it)
4. Click **Run workflow** (workflow_dispatch)
5. Select the `main` branch and click **Run workflow**
6. Wait for the workflow to complete (10–15 minutes)

**Option B: Local Terraform**

```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform init
terraform plan
terraform apply
```

Type `yes` when prompted. This takes 10–15 minutes.

### Step 2: Configure kubectl

Once infrastructure is up, point your local `kubectl` at the new cluster:

```bash
aws eks update-kubeconfig --name pharma-dev-cluster --region us-east-1
```

> **What does this command do?**
> It adds a new context to your `~/.kube/config` file. This context tells `kubectl` how to authenticate with the EKS cluster using your AWS credentials. Every `kubectl` command you run after this will target the EKS cluster.

### Step 3: Verify

```bash
kubectl get nodes
```

**Expected output:**
```
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-3-xxx.ec2.internal   Ready    <none>   5m    v1.32.x
ip-10-0-4-xxx.ec2.internal   Ready    <none>   5m    v1.32.x
ip-10-0-3-yyy.ec2.internal   Ready    <none>   5m    v1.32.x
```

You should see 2–3 nodes in `Ready` status. If the nodes show `NotReady`, wait a few minutes — they are still bootstrapping.

> **No tag needed** — this step has no code changes.

---

## 3.2 Add Bootstrap Scripts to the Infra Repo

The infra repo needs a set of Python scripts that automate cluster setup tasks (installing Helm charts, configuring ArgoCD, setting up External Secrets). These scripts are provided as part of the course materials.

### Step 1: Create the Scripts Directory

```bash
cd ~/devops/zenpharma/infra
git checkout -b feat/bootstrap-scripts
mkdir -p scripts
```

### Step 2: Copy the Scripts

Copy all 6 scripts from the course reference materials into your infra repo:

```bash
cp /path/to/course-materials/scripts/01_install_prerequisites.py scripts/
cp /path/to/course-materials/scripts/02_bootstrap_argocd.py scripts/
cp /path/to/course-materials/scripts/03_setup_external_secrets.py scripts/
cp /path/to/course-materials/scripts/04_run_pipeline.py scripts/
cp /path/to/course-materials/scripts/05_deploy_services.py scripts/
cp /path/to/course-materials/scripts/06_verify_deployment.py scripts/
```

> **Note:** Replace `/path/to/course-materials/scripts/` with the actual location where the course instructor has provided these files. They may be available as a downloadable zip or in a shared repository.


### Step 3: Verify and Push

```bash
ls scripts/
# Expected: 01_install_prerequisites.py  02_bootstrap_argocd.py  03_setup_external_secrets.py
#           04_run_pipeline.py  05_deploy_services.py  06_verify_deployment.py

cd ~/devops/zenpharma/infra
git add scripts
git commit -m "Adding bootstrap scripts"
git push origin feat/bootstrap-scripts
```
Raise a pull request to commit changes onto main. As we are doing changes outside of `envs/dev` and `modules/` it wont trigger a pipeline. 

> **Tag `infra` repo: `module-3.2-bootstrap-scripts`**
> ```bash
> cd ~/devops/zenpharma/infra
> git tag -a module-3.2-bootstrap-scripts -m "Module 3.2: Add bootstrap scripts to infra repo"
> git push origin module-3.2-bootstrap-scripts
> ```

### Understanding the Bootstrap Scripts

Now let's understand what each script does. There are 6 scripts, each handling a distinct stage of cluster setup:

| # | Script | Purpose | Run in This Module? |
|---|--------|---------|---------------------|
| 1 | `01_install_prerequisites.py` | Installs AWS LB Controller, ArgoCD, External Secrets Operator via Helm | Yes |
| 2 | `02_bootstrap_argocd.py` | Registers the gitops repo in ArgoCD, creates the pharma AppProject | Yes |
| 3 | `03_setup_external_secrets.py` | Creates ClusterSecretStore and ExternalSecret resources for secret syncing | Yes |
| 4 | `04_run_pipeline.py` | Triggers GitHub Actions CI pipelines to build Docker images | No (Module 4+) |
| 5 | `05_deploy_services.py` | Creates ArgoCD Application resources to deploy services | No (Module 5+) |
| 6 | `06_verify_deployment.py` | Runs health checks on all deployed services | No (Module 5+) |

> **Why are there 6 separate scripts instead of one big script?**
> - **Sequential dependencies:** Each script depends on the previous one completing. You cannot register a repo in ArgoCD (script 02) until ArgoCD is installed (script 01). You cannot deploy services (script 05) until images are built (script 04).
> - **Partial re-runs:** If script 03 fails, you can fix the issue and re-run just script 03. You don't need to reinstall ArgoCD.
> - **Course modularity:** We teach cluster setup (scripts 01–03) separately from application deployment (scripts 04–06). Students can checkpoint their progress between scripts.

### Script Dependency Chain

```
01_install_prerequisites.py     ← Helm charts: ALB Controller, ArgoCD, ESO
         │
         ▼
02_bootstrap_argocd.py          ← Registers gitops repo, creates AppProject
         │
         ▼
03_setup_external_secrets.py    ← Creates ClusterSecretStore, ExternalSecrets
         │
         ▼
04_run_pipeline.py              ← Triggers CI builds (Docker images → ECR)
         │
         ▼
05_deploy_services.py           ← Creates ArgoCD Applications
         │
         ▼
06_verify_deployment.py         ← Health checks everything
```

In this module, we run scripts **01, 02, and 03**. By the end, the cluster will have all the "plumbing" in place — but no applications deployed yet.

> **No tag needed** — this is conceptual context.

---

## 3.3 Install Cluster Prerequisites

This section installs three critical Kubernetes add-ons using Helm charts. We will walk through each installation manually first so you understand every command, then introduce the script that automates all of it.

### Understanding What We're Installing

| Tool | Problem It Solves |
|------|------------------|
| **AWS Load Balancer Controller** | Kubernetes `Ingress` resources need to become real AWS Application Load Balancers |
| **ArgoCD** | Kubernetes manifests in your gitops repo need to be automatically synced to the cluster |
| **External Secrets Operator** | Secrets stored in AWS Secrets Manager need to appear as native Kubernetes Secrets |

### Step 1: Add Helm Repositories

Before installing any charts, you need to register the Helm repositories that host them:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo add external-secrets https://charts.external-secrets.io
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

> **What is a Helm chart?**
> A Helm chart is a package of pre-configured Kubernetes manifests. Instead of writing dozens of YAML files for Deployments, Services, ServiceAccounts, RBAC rules, etc., you install a chart with a single command. The chart author handles the complexity; you just provide configuration values (like cluster name or IAM role ARN).

### Step 2: Install AWS Load Balancer Controller (Manual)

Before running the Helm command, you need two values from your AWS infrastructure. Run these commands to look them up:

**Get your VPC ID:**
```bash
aws eks describe-cluster --name pharma-dev-cluster --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" --output text
```
This returns something like: `vpc-0bdce2c84ef7f0507`

**Get your ALB Controller Role ARN:**
```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'alb-controller')].Arn" --output text
```
This returns something like: `arn:aws:iam::873135413040:role/pharma-dev-alb-controller-role`

> **Where do these values come from?** Both were created by Terraform in Module 1. The VPC module created the VPC, and the IAM module created the ALB controller role. If you know your AWS account ID (`aws sts get-caller-identity --query "Account" --output text`), you can also construct the role ARN directly: `arn:aws:iam::<account-id>:role/pharma-dev-alb-controller-role`

Now run the Helm command, replacing the two placeholder values with the outputs from above:

```bash
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=pharma-dev-cluster \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id> \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=<your-alb-role-arn>" \
  --wait --timeout 5m
```

**What each flag does:**

- `clusterName` — tells the controller which EKS cluster to manage. It uses this to tag AWS resources it creates so they can be tracked.
- `region` and `vpcId` — tells the controller where to create ALBs. It needs to know the VPC to place load balancers in the correct subnets.
- `serviceAccount.annotations.eks.amazonaws.com/role-arn` — the IRSA annotation that gives the controller AWS permissions. Without this, the controller pod cannot call AWS APIs to create load balancers.

> **What is the AWS Load Balancer Controller?**
> It is a Kubernetes controller that runs as a pod in `kube-system`. It watches for `Ingress` resources with `ingressClassName: alb`. When it finds one, it calls the AWS API to create an Application Load Balancer, configure target groups, and route traffic to your pods. Without it, `Ingress` resources would just sit there doing nothing.

> **What is the IRSA annotation?**
> The line `serviceAccount.annotations.eks.amazonaws.com/role-arn=<role-arn>` is how IRSA (IAM Roles for Service Accounts) works. When a pod runs with this service account, EKS automatically injects short-lived AWS credentials (via a projected volume token). The pod can then call AWS APIs (like creating load balancers) without storing any access keys. This is the most secure way to give Kubernetes pods AWS permissions.

Verify the controller is running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Expected output:**
```
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
aws-load-balancer-controller-xxxxxxxxxx-yyyyy   1/1     Running   0          2m
```

### Step 3: Install ArgoCD (Manual)

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --wait --timeout 10m
```

After installation, retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Save this password** — you will need it in the next section.

> **What is ArgoCD?**
> ArgoCD is a GitOps continuous delivery tool. It watches a Git repository (your `gitops` repo) and continuously compares the desired state (YAML files in the repo) with the actual state (what is running in the cluster). If they differ, ArgoCD syncs the cluster to match the repo. This means deploying is as simple as merging a PR — no `kubectl apply` needed.

### Step 4: Install External Secrets Operator (Manual)

```bash
helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true \
  --wait --timeout 5m
```

> **What is the External Secrets Operator (ESO)?**
> ESO watches for `ExternalSecret` custom resources in your cluster. Each `ExternalSecret` says "fetch key X from AWS Secrets Manager and create a Kubernetes Secret with that value." ESO does the fetching automatically, refreshing on a schedule (we use `1h`). This way, no one needs to manually create Kubernetes Secrets or copy passwords around — Terraform puts secrets into AWS Secrets Manager, and ESO syncs them into the cluster.

### Verify All Three Installations

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl get pods -n argocd
kubectl get pods -n external-secrets
```

**Expected output (AWS Load Balancer Controller):**
```
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
aws-load-balancer-controller-xxxxxxxxxx-yyyyy   1/1     Running   0          2m
```

**Expected output (ArgoCD):**
```
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          3m
argocd-applicationset-controller-xxxxxxxxxx-xxxxx  1/1     Running   0          3m
argocd-dex-server-xxxxxxxxxx-xxxxx                 1/1     Running   0          3m
argocd-notifications-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          3m
argocd-redis-xxxxxxxxxx-xxxxx                      1/1     Running   0          3m
argocd-repo-server-xxxxxxxxxx-xxxxx                1/1     Running   0          3m
argocd-server-xxxxxxxxxx-xxxxx                     1/1     Running   0          3m
```

**Expected output (External Secrets Operator):**
```
NAME                                                READY   STATUS    RESTARTS   AGE
external-secrets-xxxxxxxxxx-xxxxx                   1/1     Running   0          2m
external-secrets-cert-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
external-secrets-webhook-xxxxxxxxxx-xxxxx           1/1     Running   0          2m
```

All pods should show `Running` with `1/1` ready. If any pod shows `CrashLoopBackOff` or `Error`, check its logs:

```bash
kubectl logs <pod-name> -n <namespace>
```

### Step 5: The Automated Way — Using the Script

> **Why use a script?** In the steps above, you ran each command manually. This is great for learning, but in practice you'd run these commands every time you recreate the cluster (which we do between modules). The `01_install_prerequisites.py` script automates all of the above — Helm repos, ALB Controller, ArgoCD, ESO — in a single run with input prompts and verification built in.

```bash
cd ~/devops/zenpharma
python3 infra/scripts/01_install_prerequisites.py
```

The script will ask you for 4 values:

| Input | Default / How to Find |
|-------|----------------------|
| **EKS cluster name** | `pharma-dev-cluster` |
| **AWS region** | `us-east-1` |
| **VPC ID** | Auto-detected from the cluster. If it fails: `aws eks describe-cluster --name pharma-dev-cluster --region us-east-1 --query "cluster.resourcesVpcConfig.vpcId" --output text` |
| **ALB controller role ARN** | `aws iam list-roles --query "Roles[?contains(RoleName, 'alb-controller')].Arn" --output text` |

These are the same values you used in the manual `helm upgrade --install` commands above. The script also handles ALB webhook certificate refresh (a safety measure for re-installs), displays the ArgoCD admin password, and runs pod verification automatically.

---

## 3.4 Bootstrap ArgoCD

ArgoCD is installed and running, but it doesn't know where to look for Kubernetes manifests. We need to:

1. Register the gitops repository so ArgoCD can clone it
2. Create the `pharma` AppProject so ArgoCD knows which namespaces and repos are allowed

### Understanding What We Need to Configure

ArgoCD uses two key concepts here:

- **Repository registration:** ArgoCD needs credentials to clone your private `gitops` repo. You provide these via a labeled Kubernetes Secret in the `argocd` namespace.
- **AppProject:** A security boundary that defines which repos ArgoCD can pull from and which namespaces it can deploy to.

### Step 1: Create a GitHub Personal Access Token (PAT)

ArgoCD needs read access to your private `gitops` repository. You provide this via a GitHub Personal Access Token.

**How to create a fine-grained PAT:**

1. Go to https://github.com/settings/tokens
2. Click **Fine-grained tokens** in the left sidebar
3. Click **Generate new token**
4. **Token name:** `argocd-gitops-read`
5. **Expiration:** 90 days (or your preference)
6. **Resource owner:** Select your GitHub organization (e.g., `zenpharma`)
7. **Repository access:** Select **Only select repositories** → choose your `gitops` repository
8. **Permissions:** Under **Repository permissions**, set **Contents** to **Read-only**
9. Click **Generate token**
10. **IMPORTANT:** Copy the token immediately — you won't see it again

> **Why a fine-grained token instead of a classic token?**
> - **Least privilege:** Fine-grained tokens can be scoped to a single repository with read-only access. Classic tokens grant access to all repositories.
> - **Auditability:** Fine-grained tokens show which resources they can access in the token settings page.
> - **Organization control:** Organization admins can require fine-grained tokens and set maximum lifetimes.

### Step 2: Register the GitOps Repository in ArgoCD (Manual)

Create a Kubernetes Secret containing the repository URL and credentials, then label it so ArgoCD recognizes it:

```bash
# Create the secret with repo credentials
kubectl create secret generic zen-gitops-repo \
  --namespace argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/<your-org>/gitops.git \
  --from-literal=username=<your-username> \
  --from-literal=password=<your-pat-token> \
  --dry-run=client -o yaml | kubectl apply -f -

# Label it so ArgoCD recognizes it as a repository
kubectl label secret zen-gitops-repo \
  argocd.argoproj.io/secret-type=repository \
  --namespace argocd \
  --overwrite
```

> **Why store credentials in a Kubernetes Secret?**
> ArgoCD reads repository credentials from labeled Secrets in its namespace. This is ArgoCD's native credential management mechanism. The Secret contains the PAT token, which ArgoCD uses to clone the private repository. The `--dry-run=client -o yaml | kubectl apply` pattern is an idempotent upsert — it works whether the Secret exists or not.

### Step 3: Create the pharma AppProject (Manual)

The AppProject defines what ArgoCD is allowed to do — which repos it can pull from and which namespaces it can deploy to. We haven't created this file in the gitops repo yet (that comes in Module 5), so let's apply it directly:

First, create the directory in your gitops repo:

```bash
mkdir -p ~/devops/zenpharma/gitops/argocd/projects
```

Create `~/devops/zenpharma/gitops/argocd/projects/pharma-project.yaml` with this content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: pharma
  namespace: argocd
spec:
  description: Pharma microservices project

  sourceRepos:
    # Replace '<your-username>' with your GitHub username or org name
    - "https://github.com/<your-username>/gitops.git"

  destinations:
    - server: https://kubernetes.default.svc
      namespace: dev
    - server: https://kubernetes.default.svc
      namespace: qa
    - server: https://kubernetes.default.svc
      namespace: prod

  clusterResourceWhitelist:
    - group: "*"
      kind: "*"

  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Now apply it:

```bash
kubectl apply -f ~/devops/zenpharma/gitops/argocd/projects/pharma-project.yaml
```

> **Note:** You may see a warning: `prefer a domain-qualified finalizer name`. This is harmless — Kubernetes is suggesting a naming convention, not reporting an error. The AppProject works fine.

**What this manifest does:**

- **`sourceRepos`** — ArgoCD can only pull from your gitops repo. It cannot deploy from any other repository.
- **`destinations`** — ArgoCD can only deploy to `dev`, `qa`, and `prod` namespaces. It cannot touch `kube-system` or other critical namespaces.
- **`clusterResourceWhitelist`** and **`namespaceResourceWhitelist`** — Which Kubernetes resource types ArgoCD can create. `"*"` means all types.

> **Why an AppProject instead of using the default project?**
> - **Security boundary:** The `pharma` project restricts which repositories and namespaces ArgoCD Applications can use. An Application in this project cannot deploy to `kube-system` or pull from an unauthorized repo.
> - **Multi-tenancy:** In a shared cluster, each team gets their own AppProject with its own permissions. This prevents one team from accidentally deploying into another team's namespace.

Commit this file to the gitops repo via a feature branch and pull request:

```bash
cd ~/devops/zenpharma/gitops
git checkout -b feat/argocd-project
git add argocd/projects/pharma-project.yaml
git commit -m "feat: add ArgoCD pharma AppProject"
git push origin feat/argocd-project
```

Then create a pull request and merge it:

```bash
gh pr create --title "feat: add ArgoCD pharma AppProject" \
  --body "Adds the pharma AppProject manifest for ArgoCD" --base main
gh pr merge --merge
```

### Step 4: Access the ArgoCD UI

Open a port-forward to the ArgoCD server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open your browser to: **https://localhost:8080**

> **Note:** Your browser will show a certificate warning because ArgoCD uses a self-signed TLS certificate. Click "Advanced" → "Proceed to localhost" (or equivalent in your browser).

Log in with:
- **Username:** `admin`
- **Password:** The password displayed by `01_install_prerequisites.py` (from the previous section)

If you lost the password, retrieve it with:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Once logged in, you should see:
- The ArgoCD dashboard with no applications yet
- Under **Settings → Repositories**: your gitops repo listed with a green connection status
- Under **Settings → Projects**: the `pharma` project

### Step 5: The Automated Way — Using the Script

> **Why use a script?** The manual steps above are straightforward, but the script packages them into a single run with input prompts, validation, and error handling. It does the same things: registers the repo as a labeled Secret and applies the AppProject manifest.

```bash
cd ~/devops/zenpharma
python3 infra/scripts/02_bootstrap_argocd.py
```

The script asks for 4 inputs:

| Input | What to Enter |
|-------|--------------|
| **Target environment** | Choose `dev` (option 1) |
| **GitOps repo URL** | `https://github.com/<your-org>/gitops.git` (the HTTPS URL from Step 2) |
| **GitHub username** | Your GitHub username (the same one from Step 2) |
| **GitHub PAT token** | The token you created in Step 1 (input is hidden) |

> **Tag `gitops` repo: `module-3.4-argocd-bootstrap`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git checkout main && git pull origin main
> git tag -a module-3.4-argocd-bootstrap -m "Module 3.4: ArgoCD bootstrapped with gitops repo"
> git push origin module-3.4-argocd-bootstrap
> ```

---

## 3.5 Setup External Secrets

The final bootstrap step wires up the External Secrets Operator to AWS Secrets Manager. After this, Kubernetes Secrets will be automatically created from the secrets that Terraform stored in AWS Secrets Manager (db-credentials and jwt-secret).

### The Full Secret Chain

Understanding the end-to-end flow is critical:

```
Terraform (Module 1)                        External Secrets Operator              Pods
┌──────────────────────┐                   ┌──────────────────────────┐          ┌──────────┐
│ aws_secretsmanager_  │                   │ ExternalSecret CRD       │          │ Backend  │
│ secret_version       │ ── Stores in ──>  │ reads from Secrets Mgr   │ ──────>  │ Service  │
│                      │    AWS Secrets    │ creates K8s Secret       │  mounts  │ reads    │
│ /pharma/dev/         │    Manager        │ in target namespace      │  as env  │ DB_HOST  │
│   db-credentials     │                   │                          │  vars    │ DB_PASS  │
│   jwt-secret         │                   │                          │          │ JWT_SEC  │
└──────────────────────┘                   └──────────────────────────┘          └──────────┘
```

> **Why not just create Kubernetes Secrets directly with Terraform?**
> - **Separation of concerns:** Terraform manages AWS infrastructure. Kubernetes operators manage in-cluster state. Mixing the two creates tight coupling.
> - **Secret rotation:** If a password changes in AWS Secrets Manager, ESO automatically picks up the new value (every hour by default). With Terraform-created Secrets, you'd need to re-run Terraform and restart pods.
> - **No secrets in state files:** Terraform state files contain the values of everything Terraform manages. If Terraform created K8s Secrets directly, the secret values would be in the state file in plaintext.

### Step 1: Create the Target Namespace (Manual)

The ExternalSecrets will create Kubernetes Secrets in the `dev` namespace. Ensure it exists:

```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
```

### Step 2: Annotate ESO Service Account with IRSA (Manual)

The External Secrets Operator needs AWS permissions to read from Secrets Manager. First, look up the ESO role ARN:

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'eso-role')].Arn" --output text
```
This returns something like: `arn:aws:iam::873135413040:role/pharma-dev-eso-role`

Now annotate the service account with the role ARN, then restart the pods so they pick up the new credentials:

```bash
kubectl annotate serviceaccount external-secrets \
  --namespace external-secrets \
  eks.amazonaws.com/role-arn=<your-eso-role-arn> \
  --overwrite
```

Then restart the ESO pods so they pick up the new annotation:

```bash
kubectl rollout restart deployment/external-secrets -n external-secrets
kubectl rollout status deployment/external-secrets -n external-secrets --timeout=120s
```

> **Why annotate and restart?**
> The IRSA annotation tells EKS to inject AWS credentials into the pod. However, if the pod was already running when the annotation was added, it won't have the credentials. The restart ensures new pods are created with the projected service account token volume, allowing them to assume the IAM role.

### Step 3: Create the ClusterSecretStore (Manual)

The ClusterSecretStore tells ESO how to connect to AWS Secrets Manager:

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
EOF
```

> **What is a ClusterSecretStore?**
> It defines *how* to connect to a secret backend (in our case, AWS Secrets Manager). The `jwt` auth method tells ESO to use the service account's IRSA token to authenticate with AWS. A `ClusterSecretStore` is cluster-scoped, meaning ExternalSecrets in any namespace can reference it. A regular `SecretStore` would be namespace-scoped.

### Step 4: Create ExternalSecrets (Manual)

Create the two ExternalSecret resources that tell ESO which secrets to sync from AWS Secrets Manager into Kubernetes:

**ExternalSecret: db-credentials**

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: dev
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
    - secretKey: SPRING_DATASOURCE_USERNAME
      remoteRef:
        key: /pharma/dev/db-credentials
        property: username
    - secretKey: SPRING_DATASOURCE_PASSWORD
      remoteRef:
        key: /pharma/dev/db-credentials
        property: password
    - secretKey: DB_HOST
      remoteRef:
        key: /pharma/dev/db-credentials
        property: host
EOF
```

**ExternalSecret: jwt-secret**

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: jwt-secret
  namespace: dev
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: jwt-secret
    creationPolicy: Owner
  data:
    - secretKey: JWT_SECRET
      remoteRef:
        key: /pharma/dev/jwt-secret
        property: secret
EOF
```

> **Why are SPRING_DATASOURCE_USERNAME and DB_USERNAME both mapped from the same source?**
> Different backend services expect different environment variable names. The Java Spring Boot services expect `SPRING_DATASOURCE_USERNAME` and `SPRING_DATASOURCE_PASSWORD`, while the Node.js services expect `DB_USERNAME` and `DB_PASSWORD`. By mapping both from the same Secrets Manager value, one Kubernetes Secret serves all services without modification.

> **What does `refreshInterval: 1h` mean?**
> ESO will re-read the value from AWS Secrets Manager every hour. If someone rotates a database password in Secrets Manager, the Kubernetes Secret will be updated within an hour. For more frequent updates, you can lower this interval (e.g., `5m`), but each refresh is an AWS API call that counts toward your Secrets Manager quota.

> **What does `creationPolicy: Owner` mean?**
> It means the ExternalSecret "owns" the Kubernetes Secret it creates. If you delete the ExternalSecret, the Kubernetes Secret is also deleted. This prevents orphaned Secrets from lingering in the cluster.

### Step 5: Verify Secrets are Synced

**Check the ClusterSecretStore:**
```bash
kubectl get clustersecretstore
```

**Expected output:**
```
NAME                   AGE   STATUS   CAPABILITIES   READY
aws-secrets-manager    1m    Valid    ReadWrite       True
```

The `STATUS` should be `Valid` and `READY` should be `True`.

**Check ExternalSecrets:**
```bash
kubectl get externalsecret -n dev
```

**Expected output:**
```
NAME              STORE                  REFRESH INTERVAL   STATUS         READY
db-credentials    aws-secrets-manager    1h                 SecretSynced   True
jwt-secret        aws-secrets-manager    1h                 SecretSynced   True
```

Both should show `SecretSynced` and `True`.

**Check the Kubernetes Secrets that were created:**
```bash
kubectl get secrets -n dev
```

**Expected output:**
```
NAME              TYPE     DATA   AGE
db-credentials    Opaque   5      1m
jwt-secret        Opaque   1      1m
```

The `db-credentials` Secret has 5 data keys (`DB_USERNAME`, `DB_PASSWORD`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `DB_HOST`) and `jwt-secret` has 1 data key (`JWT_SECRET`).

**If ExternalSecrets show an error**, debug with:
```bash
kubectl describe externalsecret db-credentials -n dev
```

Look at the `Events` section at the bottom for error messages.

> **Where should these YAML manifests live?** In this module, we applied the ClusterSecretStore and ExternalSecret manifests directly via `kubectl apply`. In a production setup, you'd store them in the gitops repo (under `k8s/external-secrets/`) so ArgoCD manages them. For this course, applying them manually is simpler since they're cluster setup — not application deployments.

### Step 6: The Automated Way — Using the Script

> **Why use a script?** The manual steps above involve creating the namespace, annotating the service account, restarting pods, applying the ClusterSecretStore and ExternalSecret manifests, and then polling to confirm secrets synced. The `03_setup_external_secrets.py` script does all of this in one run, plus it polls for sync status automatically (up to 90 seconds) and displays common failure causes if syncing fails.

```bash
cd ~/devops/zenpharma
python3 infra/scripts/03_setup_external_secrets.py
```

The script asks for 4 values:

| Input | Default / How to Find |
|-------|----------------------|
| **Target environment** | Choose `dev` (option 1) |
| **AWS region** | `us-east-1` |
| **AWS account ID** | `aws sts get-caller-identity --query "Account" --output text` |
| **ESO IAM role name** | Default: `pharma-dev-eso-role`. Verify: `aws iam list-roles --query "Roles[?contains(RoleName, 'eso-role')].RoleName" --output text` |

These are the same values you used in the manual steps above — the account ID and role name combine to form the IRSA role ARN from Step 2, and the region is used in the ClusterSecretStore from Step 3.

> **No tag needed** — section 3.5 has no code committed to a repo (all kubectl runtime commands).

---

## 3.6 Destroy Infrastructure

Unlike Modules 1 and 2, we now have resources running **inside** the cluster that were created outside of Terraform — Helm charts (ALB Controller, ArgoCD, External Secrets Operator), Kubernetes namespaces, and ExternalSecrets. These must be cleaned up **before** running `terraform destroy`, or Terraform will hang waiting for VPC subnets and security groups to be released.

### Step 1: Delete ExternalSecrets and ClusterSecretStore

```bash
kubectl delete externalsecret --all -n dev
kubectl delete clustersecretstore aws-secrets-manager
```

### Step 2: Uninstall Helm Charts

Remove in reverse order of installation:

```bash
helm uninstall external-secrets -n external-secrets
helm uninstall argocd -n argocd
helm uninstall aws-load-balancer-controller -n kube-system
```

### Step 3: Delete Namespaces

```bash
kubectl delete namespace argocd external-secrets dev --ignore-not-found
```

### Step 4: Verify No ALBs Remain

ALBs created by the ALB Controller are AWS resources that block VPC deletion. Wait 2–3 minutes for them to fully delete:

```bash
aws elbv2 describe-load-balancers --region us-east-1 \
  --query "LoadBalancers[].LoadBalancerArn" --output text
```

If any ALBs are listed, wait and check again. **Do NOT proceed until ALBs are gone.**

> **Why?** The ALB Controller creates Elastic Network Interfaces (ENIs) in your VPC subnets. If ALBs exist when Terraform deletes the VPC, it will hang with "subnet has dependencies" errors.

### Step 5: Run Terraform Destroy

**Option A: GitHub Actions (recommended)**

1. Go to `infra` repo → **Actions** → **Terraform Infrastructure**
2. Click **Run workflow**
3. Select `destroy` from the action dropdown
4. Type `destroy` in the confirmation field
5. Click **Run workflow** → Approve in the `dev` environment gate

**Option B: Local Terraform**

```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform destroy
```

Type `yes` when prompted. This takes 10–15 minutes.

### Step 6: Verify Cleanup

```bash
aws eks list-clusters --region us-east-1
aws rds describe-db-instances --region us-east-1 \
  --query 'DBInstances[].DBInstanceIdentifier'
aws ec2 describe-vpcs --region us-east-1 \
  --query 'Vpcs[?Tags].VpcId'
```

All should return empty results.

> **End of Module 3.** Infrastructure is destroyed. Recreate it before Module 4.

---

## Module 3 Summary

| What We Built | Details |
|--------------|---------|
| **EKS Cluster** | Recreated from Terraform (same as Module 1, now via CI/CD or local apply) |
| **Bootstrap Scripts** | 6 Python scripts added to infra repo for automated cluster setup |
| **AWS Load Balancer Controller** | Helm chart in `kube-system`, watches Ingress resources, creates AWS ALBs via IRSA |
| **ArgoCD** | Helm chart in `argocd` namespace, GitOps CD controller with gitops repo registered |
| **External Secrets Operator** | Helm chart in `external-secrets` namespace, syncs AWS Secrets Manager into K8s Secrets |
| **ClusterSecretStore** | Connects ESO to AWS Secrets Manager via IRSA authentication |
| **ExternalSecrets** | `db-credentials` and `jwt-secret` synced into the `dev` namespace |

| Tag | Repos |
|-----|-------|
| `module-3.2-bootstrap-scripts` | infra |
| `module-3.4-argocd-bootstrap` | gitops |

> **Next:** [Module 4 — Dockerize & Build CI/CD for Frontend](MODULE-4-DOCKERIZE-AND-BUILD-CICD-FOR-FRONTEND.md)
