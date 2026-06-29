# Module 5 — GitOps Repository & Helm Charts

> Build the central GitOps repository that ArgoCD watches to deploy applications to Kubernetes.
> Estimated time: 2-3 hours.

---

## 5.1 Create the GitOps Repository Structure

In Modules 1-4, we built AWS infrastructure, bootstrapped EKS, and created a Docker/CI pipeline for the frontend. Now we need a way to **declare** what should be running in the cluster. That is the GitOps repository.

### What Is a GitOps Repository?

A GitOps repository is a Git repo that contains the **desired state** of your Kubernetes cluster. It holds:

- **Helm charts** — templates for Kubernetes resources
- **Environment-specific values** — configuration per environment (dev, qa, prod)
- **ArgoCD application definitions** — tell ArgoCD what to deploy and where
- **Namespace definitions** — the Kubernetes namespaces your services live in

ArgoCD continuously watches this repository. When you push a change (e.g., update an image tag), ArgoCD detects the diff and applies it to the cluster automatically.

> **Why a separate repository?**
> - **Different change cadence:** Application code changes frequently (feature branches, PRs). Cluster state changes less often and in a more controlled way.
> - **Different permissions:** Developers push to app repos. Only the CI pipeline and DevOps team push to the GitOps repo (e.g., CI updates image tags automatically after a build).
> - **Single source of truth:** One repo describes the entire cluster state. If the cluster is destroyed, you can recreate everything from this repo.
> - **Auditability:** Every deployment is a Git commit. `git log` shows who deployed what, when, and why.

### Step 1: Create the Folder Structure

Navigate to your gitops repository and create the directory structure:

```bash
cd ~/devops/zenpharma/gitops

mkdir -p argocd/apps/dev
mkdir -p argocd/apps/qa
mkdir -p argocd/apps/prod
mkdir -p argocd/install
mkdir -p argocd/projects
mkdir -p db-init
mkdir -p envs/dev
mkdir -p envs/qa
mkdir -p envs/prod
mkdir -p helm-charts/templates
mkdir -p k8s
```

Your directory tree should look like this:

```
gitops/
├── argocd/
│   ├── apps/
│   │   ├── dev/          # ArgoCD Application manifests for dev environment
│   │   ├── qa/           # ArgoCD Application manifests for qa environment
│   │   └── prod/         # ArgoCD Application manifests for prod environment
│   ├── install/          # ArgoCD's own ingress and namespace definitions
│   └── projects/         # ArgoCD AppProject definitions (security boundaries)
├── db-init/              # One-time database initialization scripts
├── envs/
│   ├── dev/              # Helm values files for dev environment
│   ├── qa/               # Helm values files for qa environment
│   └── prod/             # Helm values files for prod environment
├── helm-charts/
│   └── templates/        # Shared Helm chart templates (Deployment, Service, etc.)
└── k8s/                  # Raw Kubernetes manifests (namespaces, etc.)
```

**Directory purposes:**

| Directory | Purpose |
|-----------|---------|
| `argocd/apps/<env>/` | One ArgoCD `Application` manifest per service per environment. Tells ArgoCD what to deploy and where. |
| `argocd/install/` | ArgoCD's own infrastructure — its namespace and ingress to expose the ArgoCD dashboard. |
| `argocd/projects/` | `AppProject` manifests that act as security boundaries — which repos and namespaces an app can access. |
| `db-init/` | SQL scripts to initialize the database (one-time setup, not managed by ArgoCD). |
| `envs/<env>/` | Helm values files. Each service gets one file per environment (e.g., `values-pharma-ui.yaml`). |
| `helm-charts/` | A single shared Helm chart used by all 9 microservices. Templates + default values. |
| `k8s/` | Raw Kubernetes manifests that don't go through Helm (namespaces). |

### Step 2: Initialize Git and Push

```bash
cd ~/devops/zenpharma/gitops
git add .
git commit -m "feat: create gitops repository structure"
git push origin main
```

> **Tag `gitops` repo: `module-5.1-repo-structure`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-5.1-repo-structure -m "Module 5.1: GitOps repository folder structure"
> git push origin module-5.1-repo-structure
> ```

---

## 5.2 Add Kubernetes Namespace Definitions

Kubernetes namespaces provide isolation between environments. Services in the `dev` namespace cannot accidentally affect services in `prod`.

### Step 1: Create the Namespace Manifest

Create `k8s/namespaces.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    env: dev
    managed-by: terraform
```

> **Why only `dev` for now?** We start with a single environment. When you're ready for `qa` and `prod`, you add additional namespace documents to this file (separated by `---`). The `managed-by: terraform` label is a convention that indicates this resource was originally part of the Terraform-managed infrastructure.

### Step 2: Apply the Namespace

```bash
kubectl apply -f k8s/namespaces.yaml
```

**Expected output:**
```
namespace/dev created
```

Verify it exists:

```bash
kubectl get namespaces
```

You should see `dev` in the list alongside `default`, `kube-system`, and other system namespaces.

> **Don't commit yet** — we will commit the namespace manifest along with the DB init script and raw manifests as a single commit in section 5.4.

---

## 5.3 Add Database Initialization Script

Our microservices use a shared PostgreSQL database (provisioned by Terraform in Module 1), but each service gets its own **schema** within that database. This is the schema-per-service pattern.

### Step 1: Create the SQL Script

Create `db-init/01-schemas.sql`:

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

> **Why schema-per-service?**
> - **Isolation:** Each microservice owns its data. The `auth` service only accesses the `auth` schema.
> - **Independent evolution:** Services can change their table structures without affecting others.
> - **Shared infrastructure:** All schemas live in one PostgreSQL instance, which is simpler and cheaper than running 8 separate databases.
> - **Clear boundaries:** If the `drug_catalog` service needs data from `inventory`, it must go through the API — no direct database joins across services.

> **When do we run this SQL?** Not now — we just create the file here and commit it. The actual database initialization happens in **Module 6 (section 6.2)** after the infrastructure is up and running. The RDS instance must be reachable from the cluster before we can connect to it.

> **Don't commit yet** — we will commit the DB init script along with the namespace manifest and raw manifests as a single commit in section 5.4.

---

## 5.4 Deploy Pharma-UI with Raw Kubernetes Manifests

Before we build the Helm chart, let's actually create and deploy `pharma-ui` using raw Kubernetes manifest files. This hands-on exercise gives you a concrete understanding of what each manifest does and how Kubernetes resources work together. Once you see the effort required for a single service, the motivation for Helm will be obvious.

### Step 1: Create a Raw Manifests Directory

```bash
cd ~/devops/zenpharma/gitops
mkdir -p k8s/raw-manifests/dev
```

This directory will hold the raw YAML files for the pharma-ui frontend in the dev environment.

### Step 2: Create the ServiceAccount Manifest

The Deployment references `serviceAccountName: pharma-ui`, so the ServiceAccount must exist before the pod can start.

Create `k8s/raw-manifests/dev/pharma-ui-serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pharma-ui
  namespace: dev
```

> **What is a ServiceAccount?** It's the Kubernetes identity that a pod runs as. Every pod runs as some ServiceAccount — if you don't specify one, it uses the `default` ServiceAccount. By creating a dedicated one per service, we can later attach IRSA annotations (for AWS permissions) or RBAC rules (for Kubernetes API access) to individual services without affecting others.

### Step 3: Create the Deployment Manifest

Create `k8s/raw-manifests/dev/pharma-ui-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pharma-ui
  namespace: dev
  labels:
    app: pharma-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pharma-ui
  template:
    metadata:
      labels:
        app: pharma-ui
    spec:
      serviceAccountName: pharma-ui
      securityContext:
        runAsNonRoot: true
      containers:
        - name: pharma-ui
          image: 873135413040.dkr.ecr.us-east-1.amazonaws.com/pharma-ui:sha-b8aa312
          securityContext:
            readOnlyRootFilesystem: true
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
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
```

Let's walk through each section of this Deployment manifest:

**`metadata`** — `name: pharma-ui` is how you refer to this Deployment in `kubectl` commands. `namespace: dev` places it in the dev environment namespace. `labels` are key-value pairs used for selection and organization — the `app: pharma-ui` label connects this Deployment to its Service and other resources.

**`spec.replicas: 1`** — Run one instance (pod) of this application. In production you'd increase this for high availability. If a pod crashes, Kubernetes automatically creates a replacement to maintain the desired count.

**`spec.selector.matchLabels`** — This tells the Deployment which pods it owns. The Deployment manages any pod whose labels match `app: pharma-ui`. This selector must match the labels in `spec.template.metadata.labels` — if they don't match, the Deployment cannot find the pods it creates and will report errors.

**`spec.template`** — The pod template. Every time the Deployment needs to create a pod (on initial creation, after scaling up, or to replace a crashed pod), it uses this template as a blueprint.

**`serviceAccountName: pharma-ui`** — The Kubernetes identity that the pod runs as. ServiceAccounts control what AWS resources the pod can access (via IRSA) and what Kubernetes API calls it can make.

**`securityContext` (pod level)** — `runAsNonRoot: true` — the container cannot run as the root user. If compromised, the attacker has limited system access.

**`containers`** — The container specification:
- `securityContext.readOnlyRootFilesystem: true` (container level) — the container's filesystem is read-only. Attackers cannot write malware or modify application binaries. This is why we need the volume mounts below.
- `image` — the ECR image URL with a Git SHA tag (not `latest`). SHA tags ensure you always know exactly which code version is deployed.
- `ports` — declares that the container listens on port 80 (Nginx). This is informational for humans and tools; it does not open the port.
- `resources` — **requests** are the guaranteed minimum (the scheduler uses these to place pods on nodes); **limits** are the maximum allowed (exceeding memory limits causes OOMKill, exceeding CPU causes throttling). We specify both to prevent a single pod from consuming all node resources and starving other pods.

**Liveness and readiness probes:**
- The **liveness probe** (`/` on port 80) checks if the container is alive. If it fails repeatedly, Kubernetes restarts the pod. For Nginx, we check the root path since it doesn't have Spring Actuator.
- The **readiness probe** (`/` on port 80) checks if the container is ready to serve traffic. If it fails, Kubernetes removes the pod from the Service (stops routing traffic to it) but does not restart it. `initialDelaySeconds: 5` gives Nginx a few seconds to start before checking.

**`volumeMounts` and `volumes`** — Because we set `readOnlyRootFilesystem: true`, Nginx cannot write to its default directories. These three `emptyDir` volumes provide writable storage for:
- `/tmp` — temporary files during request processing
- `/var/cache/nginx` — cached responses
- `/var/run` — the `nginx.pid` file

`emptyDir` volumes are pod-local temporary storage — created when the pod starts and deleted when the pod stops.

### Step 4: Create the Service Manifest

Create `k8s/raw-manifests/dev/pharma-ui-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pharma-ui
  namespace: dev
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: pharma-ui
```

**What a Service does:** A Kubernetes Service provides a **stable network endpoint** for a set of pods. Pods are ephemeral — they get new IP addresses every time they restart. The Service maintains a consistent DNS name (`pharma-ui.dev.svc.cluster.local`) and IP address, and routes traffic to whichever pods are currently healthy.

**`type: ClusterIP`** — The Service is only accessible from within the cluster. Other services can reach it at `http://pharma-ui:80`, but external users cannot. External access is handled by the Ingress (next step).

**`port` vs `targetPort`:**
- `port: 80` — the port the Service listens on. Other services use this port when making requests (e.g., `http://pharma-ui:80`).
- `targetPort: 80` — the port on the container that the Service forwards traffic to. Often the same as `port`, but can differ if the container listens on a non-standard port.

**`selector: app: pharma-ui`** — This is how the Service discovers its pods. It matches any pod with the label `app: pharma-ui` — the same label we defined in the Deployment's pod template. If we scaled to 3 replicas, the Service would automatically load-balance across all 3 pods.

### Step 5: Create the Ingress Manifest

Create `k8s/raw-manifests/dev/pharma-ui-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pharma-ui
  namespace: dev
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: pharma-dev
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: pharma-ui
                port:
                  number: 80
```

**What an Ingress does:** An Ingress exposes HTTP/HTTPS routes from outside the cluster to Services inside the cluster. While the Service gives pods a stable internal address, the Ingress gives them a stable external address (via a load balancer).

**`ingressClassName: alb`** — Tells Kubernetes to use the AWS Load Balancer Controller to fulfill this Ingress. The controller creates an AWS Application Load Balancer (ALB) that routes external traffic to the Service.

**ALB annotations explained:**

| Annotation | Value | Purpose |
|-----------|-------|---------|
| `alb.ingress.kubernetes.io/scheme` | `internet-facing` | Creates a **public** ALB reachable from the internet. The alternative `internal` creates an ALB only reachable within the VPC. |
| `alb.ingress.kubernetes.io/target-type` | `ip` | Routes traffic directly to **pod IPs** rather than to the node (which would use NodePort). `ip` mode is faster and required for Fargate. |
| `alb.ingress.kubernetes.io/group.name` | `pharma-dev` | **Multiple Ingress resources share one ALB.** Without this, every Ingress creates its own ALB (~$16/month each). With `group.name`, pharma-ui and api-gateway share a single ALB, saving costs. |

**Routing rule:** All HTTP requests to path `/` (and any subpath, due to `pathType: Prefix`) are routed to the `pharma-ui` Service on port 80.

### Step 6: Create the ConfigMap Manifest

Create `k8s/raw-manifests/dev/pharma-ui-configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pharma-ui
  namespace: dev
data:
  API_BASE_URL: "/api"
  AUTH_BASE_URL: "/api/auth"
  ENV: "dev"
```

**What a ConfigMap does:** A ConfigMap stores non-sensitive configuration data as key-value pairs. Pods consume ConfigMaps as environment variables (via `envFrom: configMapRef`) or as mounted files. ConfigMaps decouple configuration from container images — the same Docker image can run in dev, qa, and prod with different ConfigMaps providing environment-specific settings.

**How pods consume it:** In a Deployment, you would add an `envFrom` block referencing this ConfigMap:
```yaml
envFrom:
  - configMapRef:
      name: pharma-ui
```
This injects `API_BASE_URL`, `AUTH_BASE_URL`, and `ENV` as environment variables in the container. The React app reads these at runtime to know where to send API requests and which environment it's running in.

### Step 7: Apply the Raw Manifests

Now deploy all four manifests at once:

```bash
kubectl apply -f k8s/raw-manifests/dev/
```

**Expected output:**
```
configmap/pharma-ui created
deployment.apps/pharma-ui created
ingress.networking.k8s.io/pharma-ui created
service/pharma-ui created
serviceaccount/pharma-ui created
```

Verify the resources were created:

```bash
kubectl get pods -n dev
kubectl get svc -n dev
kubectl get ingress -n dev
```

> **Note:** The pod will likely show `ImagePullBackOff` or `ErrImagePull` if you haven't run the CI pipeline yet (Module 6). That's expected — the ECR repository may not have the `sha-b8aa312` image tag yet. The point of this exercise is to understand the manifest structure and see how `kubectl apply` works. We'll clean these up before deploying through ArgoCD.

### Step 8: Clean Up Raw Manifests

```bash
kubectl delete -f k8s/raw-manifests/dev/
```

**Expected output:**
```
configmap "pharma-ui" deleted
deployment.apps "pharma-ui" deleted
ingress.networking.k8s.io "pharma-ui" deleted
service "pharma-ui" deleted
serviceaccount "pharma-ui" deleted
```

> **Do NOT skip this step.** The raw manifests use different label selectors than the Helm chart. If you leave the old Deployment in place, ArgoCD will fail with `spec.selector: field is immutable` when it tries to deploy via Helm — because Kubernetes does not allow changing a Deployment's selector after creation. The only fix is to delete and recreate, which is what this step does.

### The Problem: Manifest Explosion

**You just created 5 files for one service in one environment.** Now imagine doing this for all 9 services across dev, qa, and prod. That's 108 YAML files, and they're 90% identical — only the image tag, resource limits, environment variables, and namespace differ. A change to the health check path means editing 27 Deployment files. This is why we convert to Helm in the next section — one chart template handles all services, and per-service differences go in small values files.

Our system has:

- **9 microservices** (pharma-ui, api-gateway, auth-service, drug-catalog-service, inventory-service, manufacturing-service, qc-service, supplier-service, notification-service)
- **3 environments** (dev, qa, prod)
- **5+ manifest types** per service (ServiceAccount, Deployment, Service, Ingress, ConfigMap, plus HPA, etc.)

That's **9 x 3 x 5 = 135 YAML files** minimum. And they're 90% identical — the only differences are image tags, resource limits, environment variables, and replica counts.

Now imagine you need to change the health check path across all services. You'd have to edit 27 Deployment files. Miss one and you have an inconsistency.

> **This is why we use Helm.** One chart template + one values file per service per environment. Instead of 108 files, we have 7 templates + 27 values files = 34 files. And the templates enforce consistency — change a pattern once, it applies everywhere.

### Commit All Raw Kubernetes Files

Now commit everything from sections 5.2, 5.3, and 5.4 as a single commit:

```bash
cd ~/devops/zenpharma/gitops
git add k8s/namespaces.yaml db-init/01-schemas.sql k8s/raw-manifests/
git commit -m "feat: add namespace, DB init script, and raw K8s manifests for pharma-ui"
git push origin main
```

> **Tag `gitops` repo: `module-5.4-raw-manifests`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-5.4-raw-manifests -m "Module 5.4: Namespace, DB init, and raw Kubernetes manifests"
> git push origin module-5.4-raw-manifests
> ```

---

## 5.5 Create the Shared Helm Chart

In section 5.4, you deployed pharma-ui with 4 raw manifest files. Now let's convert that into a single reusable Helm chart that every microservice can share. Instead of duplicating YAML for each service and environment, we create **one set of templates** with configurable values. Each service customizes the chart through its own small values file.

### Chart.yaml — Chart Metadata

Create `helm-charts/Chart.yaml`:

```yaml
apiVersion: v2
name: pharma-service
description: Common Helm chart for Pharma microservices
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - pharma
  - microservice
  - java
  - spring-boot
maintainers:
  - name: Pharma DevOps Team
    email: devops@pharma.com
```

**Key fields:**
- `apiVersion: v2` — Helm 3 format (Helm 2 used `v1`)
- `name: pharma-service` — the chart name. This appears in labels and resource names unless overridden.
- `version: 1.0.0` — the **chart** version. Bump this when you change templates.
- `appVersion: "1.0.0"` — the **default application** version. Each service overrides this with its own image tag.
- `type: application` — this chart deploys resources directly (vs `library` charts which are just helpers)

### values.yaml — Default Values

Create `helm-charts/values.yaml`:

```yaml
replicaCount: 1

image:
  repository: ""
  tag: "latest"
  pullPolicy: IfNotPresent

imagePullSecrets: []

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext:
  fsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

ingress:
  enabled: false
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  host: ""
  path: /
  pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

livenessProbe:
  path: /actuator/health
  port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

readinessProbe:
  path: /actuator/health/readiness
  port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

env: []

envFrom: []

configmap: {}

volumes: []

volumeMounts: []

nodeSelector: {}

tolerations: []

affinity: {}

topologySpreadConstraints: []

terminationGracePeriodSeconds: 30
```

Let's walk through each section:

**`image`** — The container image to deploy. `repository` is empty by default because every service provides its own ECR URL. `tag: "latest"` is a safe default but every service overrides this with a specific Git SHA tag (e.g., `sha-b8aa312`) set by the CI pipeline.

**`fullnameOverride`** — Empty by default. Each service sets this (e.g., `pharma-ui`, `api-gateway`) to get clean Kubernetes resource names. Without it, Helm generates names like `release-name-pharma-service`, which is ugly and hard to reference.

**`serviceAccount`** — Kubernetes ServiceAccounts are identities for pods. `create: true` means Helm creates one per service. Backend services annotate it with an IAM role ARN for IRSA (IAM Roles for Service Accounts), which lets pods access AWS resources like Secrets Manager without hardcoding credentials.

**`podSecurityContext`** and **`securityContext`** — Security hardening:

| Setting | Value | Why |
|---------|-------|-----|
| `runAsNonRoot: true` | Prevents running as root | If a container is compromised, the attacker has limited permissions |
| `runAsUser: 1000` | Runs as UID 1000 | A non-privileged user — cannot install packages, modify system files |
| `fsGroup: 2000` | Group ID for file access | Ensures the container can read/write files owned by this group |
| `readOnlyRootFilesystem: true` | Filesystem is read-only | Prevents attackers from writing malware to the container's filesystem |
| `allowPrivilegeEscalation: false` | Cannot gain more privileges | Blocks `sudo`, `setuid`, and other privilege escalation techniques |
| `capabilities.drop: [ALL]` | Drops all Linux capabilities | Removes network raw, chown, kill, etc. — only keep what's needed |

**`service`** — `ClusterIP` means the service is only accessible within the cluster (not from the internet). External access is handled by the Ingress. Port 8080 is the default for Spring Boot Java services. The frontend overrides this to port 80 (Nginx).

**`ingress`** — Disabled by default. Only services that need external access (pharma-ui, api-gateway) enable it. Uses the `alb` ingress class, which creates an AWS Application Load Balancer.

**`resources`** — Kubernetes resource requests and limits:
- **Requests** (`cpu: 100m`, `memory: 256Mi`) — the *guaranteed minimum*. The Kubernetes scheduler uses these to decide which node to place the pod on. `100m` means 0.1 CPU cores; `256Mi` means 256 mebibytes of RAM.
- **Limits** (`cpu: 500m`, `memory: 512Mi`) — the *maximum allowed*. If a pod exceeds its memory limit, Kubernetes kills it (OOMKilled). If it exceeds CPU, it gets throttled. The frontend overrides these with lower values because Nginx is lightweight.

**`autoscaling`** — Disabled by default. When enabled, the Horizontal Pod Autoscaler (HPA) watches CPU and memory usage and scales replicas between `minReplicas` and `maxReplicas`.

**`livenessProbe` and `readinessProbe`** — Kubernetes health checks:
- **Liveness probe** (`/actuator/health`) — "Is the container alive?" If this fails `failureThreshold` times in a row, Kubernetes **restarts** the pod. Default path is Spring Boot Actuator's health endpoint.
- **Readiness probe** (`/actuator/health/readiness`) — "Is the container ready to receive traffic?" If this fails, Kubernetes **removes the pod from the Service** (stops sending traffic) but does NOT restart it. Useful during startup or when a service is temporarily overloaded.
- `initialDelaySeconds: 60` for liveness — gives Spring Boot 60 seconds to start before checking. Java services are slow to start.
- `initialDelaySeconds: 30` for readiness — starts checking readiness sooner so the pod can receive traffic as soon as it's ready.

**`configmap`** — Empty by default. Each service adds its own environment variables (e.g., `API_BASE_URL`, `SPRING_PROFILES_ACTIVE`).

**`volumes` and `volumeMounts`** — Empty by default. Services that need writable directories (like Nginx with `readOnlyRootFilesystem: true`) mount `emptyDir` volumes.

### Templates

Helm templates are Go-template YAML files. They reference values from `values.yaml` (or an override file) using `{{ .Values.xxx }}` syntax.

#### `_helpers.tpl` — Naming Conventions

Create `helm-charts/templates/_helpers.tpl`:

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "pharma-service.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "pharma-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "pharma-service.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "pharma-service.labels" -}}
helm.sh/chart: {{ include "pharma-service.chart" . }}
{{ include "pharma-service.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "pharma-service.selectorLabels" -}}
app.kubernetes.io/name: {{ include "pharma-service.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "pharma-service.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "pharma-service.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

**Key concepts:**
- `pharma-service.fullname` — this is the most important helper. It determines the name of every Kubernetes resource (Deployment, Service, ConfigMap, etc.). When we set `fullnameOverride: pharma-ui` in a values file, every resource gets the clean name `pharma-ui` instead of the default `release-name-pharma-service`.
- `trunc 63` — Kubernetes DNS names are limited to 63 characters. This prevents naming errors.
- `pharma-service.labels` — standard labels applied to every resource. Includes the chart version, app version, and the `app.kubernetes.io/managed-by: Helm` label.
- `pharma-service.selectorLabels` — a subset of labels used in `selector.matchLabels`. These must be immutable after creation (Kubernetes does not allow changing selectors on existing Deployments).

#### `deployment.yaml` — The Core Template

Create `helm-charts/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "pharma-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "pharma-service.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "pharma-service.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- if or .Values.configmap .Values.envFrom }}
          envFrom:
            {{- if .Values.configmap }}
            - configMapRef:
                name: {{ include "pharma-service.fullname" . }}
            {{- end }}
            {{- with .Values.envFrom }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          {{- end }}
          {{- if .Values.env }}
          env:
            {{- toYaml .Values.env | nindent 12 }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: {{ .Values.livenessProbe.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
            successThreshold: {{ .Values.livenessProbe.successThreshold }}
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path }}
              port: {{ .Values.readinessProbe.port }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
            successThreshold: {{ .Values.readinessProbe.successThreshold }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

This is the most important template. Let's break down the key parts:

**`checksum/config` annotation:**
```yaml
checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
This computes a SHA-256 hash of the rendered ConfigMap. When you change an environment variable in the ConfigMap, the checksum changes, which changes the pod template annotation, which forces Kubernetes to do a **rolling restart** of the pods. Without this, pods would keep running with the old environment variables until manually restarted.

**`envFrom` — how pods get environment variables:**
```yaml
envFrom:
  {{- if .Values.configmap }}
  - configMapRef:
      name: {{ include "pharma-service.fullname" . }}
  {{- end }}
  {{- with .Values.envFrom }}
  {{- toYaml . | nindent 12 }}
  {{- end }}
```
Two sources of environment variables:
1. **ConfigMap** — non-sensitive values defined in the values file (e.g., `API_BASE_URL`, `SPRING_PROFILES_ACTIVE`). Created by the `configmap.yaml` template.
2. **`envFrom` overrides** — additional sources, typically `secretRef` entries for sensitive values (e.g., `db-credentials`, `jwt-secret`). These Kubernetes Secrets are created by the External Secrets Operator (set up in Module 3), which syncs them from AWS Secrets Manager.

**Liveness and readiness probes:**
```yaml
livenessProbe:
  httpGet:
    path: {{ .Values.livenessProbe.path }}
    port: {{ .Values.livenessProbe.port }}
```
Kubernetes calls these HTTP endpoints periodically. The defaults (`/actuator/health` on port 8080) work for all Spring Boot services. The frontend overrides them to `/` on port 80 because Nginx doesn't have Spring Actuator.

**`volumeMounts` and `volumes`:**
```yaml
{{- with .Values.volumeMounts }}
volumeMounts:
  {{- toYaml . | nindent 12 }}
{{- end }}
```
Only rendered if the values file defines volumes. The frontend needs writable `emptyDir` volumes for `/tmp`, `/var/cache/nginx`, and `/var/run` because we set `readOnlyRootFilesystem: true` but Nginx must write temporary files.

**`replicas` conditional:**
```yaml
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```
When HPA (autoscaling) is enabled, we do NOT set replicas in the Deployment — the HPA controls the replica count. Setting both would cause conflicts.

#### `service.yaml` — ClusterIP Service

Create `helm-charts/templates/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "pharma-service.selectorLabels" . | nindent 4 }}
```

**`port` vs `targetPort`:**
- `port` — the port the Service listens on (how other services in the cluster reach it). For example, `http://pharma-ui:80`.
- `targetPort` — the port the container is actually listening on. Usually the same as `port`, but can differ if the container listens on a non-standard port.
- `selector` — matches pods with the same `selectorLabels`. This is how the Service knows which pods to route traffic to.

#### `ingress.yaml` — Conditional ALB Ingress

Create `helm-charts/templates/ingress.yaml`:

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
  {{- end }}
  rules:
    - {{- if .Values.ingress.host }}
      host: {{ .Values.ingress.host | quote }}
      {{- end }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ include "pharma-service.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

This template is **conditional** — it only renders if `ingress.enabled` is `true`. Most backend services don't need direct internet access (they're reached via the api-gateway), so ingress is disabled by default.

**Key ALB annotations explained:**

| Annotation | Value | Purpose |
|-----------|-------|---------|
| `alb.ingress.kubernetes.io/scheme` | `internet-facing` | Creates a **public** ALB (vs `internal` which is only reachable within the VPC) |
| `alb.ingress.kubernetes.io/target-type` | `ip` | Routes traffic directly to **pod IPs** (vs `instance` which routes to the node and uses NodePort). `ip` mode is faster and works with Fargate. |
| `alb.ingress.kubernetes.io/group.name` | `pharma-dev` | **Multiple Ingress resources share one ALB**. Without this, every Ingress creates its own ALB ($16/month each). With it, pharma-ui and api-gateway share one ALB, saving costs. |

> **Why `group.name` saves money:** Each ALB costs ~$16/month plus data processing charges. Without grouping, 2 services with ingress = 2 ALBs = $32/month. With grouping, 2 services share 1 ALB = $16/month. For 3 environments, that's $48/month saved.

#### `configmap.yaml` — Conditional ConfigMap

Create `helm-charts/templates/configmap.yaml`:

```yaml
{{- if .Values.configmap }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.configmap }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

Only created if the values file defines `configmap` entries. The `range` loop iterates over all key-value pairs and quotes the values (important for numeric values like port numbers — Kubernetes ConfigMap data must be strings).

When a service's values file contains:
```yaml
configmap:
  API_BASE_URL: "/api"
  ENV: dev
```
This template renders:
```yaml
data:
  API_BASE_URL: "/api"
  ENV: "dev"
```
These become environment variables in the pod via `envFrom.configMapRef`.

#### `hpa.yaml` — Conditional Horizontal Pod Autoscaler

Create `helm-charts/templates/hpa.yaml`:

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "pharma-service.fullname" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "pharma-service.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

The HPA watches CPU and memory usage of the target Deployment. When average CPU utilization exceeds `targetCPUUtilizationPercentage` (default 70%), it adds more pod replicas up to `maxReplicas`. When usage drops, it scales back down to `minReplicas`.

> **Why disabled by default?** In dev, autoscaling adds complexity without benefit. You enable it in production where traffic patterns are unpredictable. Note that when `autoscaling.enabled` is `true`, the Deployment template omits `replicas` — the HPA takes full control of replica count.

#### `serviceaccount.yaml` — Conditional ServiceAccount

Create `helm-charts/templates/serviceaccount.yaml`:

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "pharma-service.serviceAccountName" . }}
  labels:
    {{- include "pharma-service.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

ServiceAccounts are Kubernetes identities for pods. The key feature is the annotations field, which is used for **IRSA (IAM Roles for Service Accounts)**:

```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::873135413040:role/pharma-dev-eks-role
```

When a ServiceAccount has this annotation, AWS automatically injects temporary IAM credentials into the pod. The pod can then access AWS resources (like Secrets Manager) without hardcoding access keys. This is the recommended way to give EKS pods AWS permissions.

### Step 2: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add helm-charts/
git commit -m "feat: add shared Helm chart for all microservices"
git push origin main
```

> **Tag `gitops` repo: `module-5.5-helm-chart`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-5.5-helm-chart -m "Module 5.5: Shared Helm chart with all templates"
> git push origin module-5.5-helm-chart
> ```

---

## 5.6 Add Pharma-UI Values File for Dev

Now we create the environment-specific values file that tells Helm how to deploy the `pharma-ui` frontend in the `dev` environment.

### Step 1: Create the Values File

Create `envs/dev/values-pharma-ui.yaml`:

```yaml
replicaCount: 1
fullnameOverride: pharma-ui
image:
  repository: 873135413040.dkr.ecr.us-east-1.amazonaws.com/pharma-ui
  tag: sha-b8aa312
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
    alb.ingress.kubernetes.io/group.name: pharma-dev
  host: ""
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
  ENV: dev
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
```

### Step 2: Walk Through Each Override

Let's compare this to the defaults and understand why each value is different:

**`fullnameOverride: pharma-ui`** — All Kubernetes resources for this service will be named `pharma-ui` (Deployment, Service, ConfigMap, ServiceAccount). Without this, they'd be named something like `dev-pharma-service`, which is meaningless.

**`image.repository` and `image.tag`** — Points to the ECR repository created in Module 1. The tag `sha-b8aa312` is a Git commit SHA set by the CI pipeline (Module 4). Using SHA tags instead of `latest` ensures you always know exactly which code is deployed.

**`image.pullPolicy: Always`** — Forces Kubernetes to pull the image every time a pod starts. This ensures you always get the correct image for the tag, even if a node has a cached copy. Important in dev where images change frequently.

**`service.port: 80` and `targetPort: 80`** — Unlike the default (8080 for Java services), Nginx serves on port 80. Other services in the cluster reach pharma-ui at `http://pharma-ui:80`.

**`ingress.enabled: true`** — The frontend needs to be accessible from the internet (users visit it in their browser). Most backend services leave this as `false` — they're reached through the api-gateway.

**`ingress.annotations` with `group.name: pharma-dev`** — This Ingress shares an ALB with the api-gateway. Both services have `group.name: pharma-dev`, so the AWS Load Balancer Controller creates a single ALB with routing rules for both:
- `/` goes to pharma-ui (the frontend)
- `/api` goes to api-gateway (the backend)

**`resources`** — Lower than defaults because Nginx serving static files is lightweight:
- Default: `cpu: 100m/500m`, `memory: 256Mi/512Mi` (sized for Java Spring Boot)
- Pharma-UI: `cpu: 50m/200m`, `memory: 64Mi/128Mi` (Nginx uses very little)

**`livenessProbe` and `readinessProbe`** — Override the defaults entirely:
- Path: `/` instead of `/actuator/health` — Nginx doesn't have Spring Actuator, so we just check if the root page loads
- Port: `80` instead of `8080`
- `initialDelaySeconds: 10` (liveness) / `5` (readiness) instead of `60` / `30` — Nginx starts in milliseconds, not the 30-60 seconds Java needs

**`configmap`** — Three environment variables that the React app reads at runtime:
- `API_BASE_URL: "/api"` — where to send API requests (relative path, routed by ALB to api-gateway)
- `AUTH_BASE_URL: "/api/auth"` — where to send authentication requests
- `ENV: dev` — the current environment (used for display/logging)

**`volumeMounts` and `volumes`** — Three `emptyDir` volumes:
```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: nginx-cache
    mountPath: /var/cache/nginx
  - name: nginx-run
    mountPath: /var/run
```
These are required because we set `readOnlyRootFilesystem: true` in the security context, but Nginx needs to write to:
- `/tmp` — temporary files during request processing
- `/var/cache/nginx` — cached responses
- `/var/run` — the `nginx.pid` file

`emptyDir` volumes are temporary, pod-local storage. They're created when the pod starts and deleted when the pod stops. They don't persist data — they just provide writable directories for the container.

### Contrast: Backend Service Values

To see how a backend Java service differs, here is `envs/dev/values-api-gateway.yaml`:

```yaml
replicaCount: 1
fullnameOverride: api-gateway
image:
  repository: 873135413040.dkr.ecr.us-east-1.amazonaws.com/api-gateway
  tag: sha-32835e1
  pullPolicy: Always
service:
  type: ClusterIP
  port: 8080
  targetPort: 8080
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: pharma-dev
  host: ""
  path: /api
  pathType: Prefix
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70
livenessProbe:
  path: /actuator/health
  port: 8080
  initialDelaySeconds: 60
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
readinessProbe:
  path: /actuator/health/readiness
  port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
configmap:
  SPRING_PROFILES_ACTIVE: dev
  LOG_LEVEL: DEBUG
  SERVER_PORT: "8080"
  # These match the fullnameOverride values of each service
  AUTH_SERVICE_URL: "http://auth-service:8081"
  DRUG_CATALOG_URL: "http://drug-catalog-service:8082"
  NOTIFICATION_URL: "http://notification-service:3000"
  QC_SERVICE_URL: "http://qc-service:8086"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics,prometheus"
envFrom:
  - secretRef:
      name: db-credentials
  - secretRef:
      name: jwt-secret
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::873135413040:role/pharma-dev-eks-role
  name: api-gateway
```

**Key differences from pharma-ui:**

| Aspect | pharma-ui (Frontend) | api-gateway (Backend) |
|--------|---------------------|----------------------|
| **Port** | 80 (Nginx) | 8080 (Spring Boot) |
| **Ingress path** | `/` (serves the UI) | `/api` (API endpoints) |
| **Resources** | 50m/64Mi (lightweight) | 100m/256Mi (JVM needs more) |
| **Probe path** | `/` (Nginx root) | `/actuator/health` (Spring Actuator) |
| **Probe delay** | 10s (Nginx starts fast) | 60s (JVM starts slow) |
| **configmap** | `API_BASE_URL`, `ENV` | `SPRING_PROFILES_ACTIVE`, service URLs |
| **envFrom** | None | `db-credentials`, `jwt-secret` (Secrets) |
| **ServiceAccount** | No IRSA annotation | IRSA annotation for AWS access |
| **Volumes** | 3 (tmp + nginx dirs) | 1 (tmp only for Java) |

> **Why does api-gateway have `envFrom` with secretRef?** Backend services need database credentials and JWT signing keys. These are stored in AWS Secrets Manager and synced to Kubernetes Secrets by the External Secrets Operator. Using `envFrom.secretRef` injects them as environment variables without putting secrets in Git.

### Step 3: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add envs/dev/values-pharma-ui.yaml
git commit -m "feat: add pharma-ui values file for dev environment"
git push origin main
```

> **Tag `gitops` repo: `module-5.6-pharma-ui-values`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-5.6-pharma-ui-values -m "Module 5.6: Pharma-UI values file for dev"
> git push origin module-5.6-pharma-ui-values
> ```

---

## 5.7 Add ArgoCD Configuration

ArgoCD needs to know: which repositories it can access, which namespaces it can deploy to, and how to expose its dashboard. We define all of this in the gitops repo.

### Step 1: Create the ArgoCD AppProject

Create `argocd/projects/pharma-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: pharma
  namespace: argocd
  labels:
    managed-by: terraform
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

  orphanedResources:
    warn: false

  roles:
    - name: pharma-admin
      description: Admin role for pharma project
      policies:
        - p, proj:pharma:pharma-admin, applications, *, pharma/*, allow
      groups:
        - pharma-devops-team
```

**Key fields explained:**

**`sourceRepos`** — Which Git repositories ArgoCD is allowed to pull from. This is a security whitelist — ArgoCD will refuse to deploy from any repo not listed here. We only include our gitops repo since that's where all our Helm charts and values files live.

> **Important:** Replace `<your-username>` with your GitHub username or org name.

**`destinations`** — Which cluster/namespace combinations are allowed. ArgoCD can only deploy to namespaces listed here. This prevents someone from accidentally deploying to `kube-system` or other critical namespaces. We allow `dev`, `qa`, and `prod` — our three application environments.

**`clusterResourceWhitelist`** and **`namespaceResourceWhitelist`** — Which Kubernetes resource types ArgoCD can create. `"*"` means all types. In a stricter setup, you'd limit this to only `Deployment`, `Service`, `Ingress`, `ConfigMap`, etc.

**`orphanedResources.warn: false`** — Don't warn about resources in allowed namespaces that aren't managed by ArgoCD. Some resources (like Secrets created by External Secrets Operator) exist in the namespace but aren't in Git.

**`roles`** — RBAC within the ArgoCD project. The `pharma-admin` role can perform all operations on all applications in the `pharma` project. Members of the `pharma-devops-team` group get this role.

> **Why AppProject as a security boundary?** In a real organization, you might have multiple teams sharing one ArgoCD instance. Team A's applications should not be able to deploy into Team B's namespaces. AppProjects enforce this separation.

### Step 2: Create ArgoCD Namespace and Ingress

Create `argocd/install/argocd-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    managed-by: terraform
    app.kubernetes.io/name: argocd
```

Create `argocd/install/argocd-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  labels:
    managed-by: terraform
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/group.name: pharma-argocd
spec:
  ingressClassName: alb
  rules:
    - host: argocd.pharma.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

**Ingress annotations explained:**

| Annotation | Value | Purpose |
|-----------|-------|---------|
| `backend-protocol: HTTPS` | The ArgoCD server speaks HTTPS | ALB must forward HTTPS traffic, not HTTP. Without this, you get 502 errors because ALB sends HTTP to a server expecting HTTPS. |
| `listen-ports: '[{"HTTPS":443}]'` | ALB listens on port 443 only | The ArgoCD dashboard should only be accessible over HTTPS. |
| `ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06` | Enforce TLS 1.2+ | Disables older TLS versions (1.0, 1.1) that have known vulnerabilities. |
| `group.name: pharma-argocd` | Separate ALB for ArgoCD | ArgoCD gets its own ALB, separate from the application ALB (`pharma-dev`). This is intentional — ArgoCD is management infrastructure, not an application. |

> **Why a separate ALB group for ArgoCD?** The application ALB (`pharma-dev`) serves end users. The ArgoCD ALB (`pharma-argocd`) is for the DevOps team only. Separating them means you can apply different security rules (e.g., restrict ArgoCD's ALB to your office IP range).

### Step 3: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add argocd/projects/ argocd/install/
git commit -m "feat: add ArgoCD project and ingress configuration"
git push origin main
```

> **Tag `gitops` repo: `module-5.7-argocd-config`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-5.7-argocd-config -m "Module 5.7: ArgoCD project and ingress configuration"
> git push origin module-5.7-argocd-config
> ```

---

## 5.8 Create ArgoCD Application Manifest for Pharma-UI (Dev)

This is where everything comes together. The ArgoCD `Application` manifest connects:
- **The Helm chart** (templates in `helm-charts/`)
- **The values file** (overrides in `envs/dev/values-pharma-ui.yaml`)
- **The target namespace** (deploy into `dev`)

### Step 1: Create the Application Manifest

Create `argocd/apps/dev/pharma-ui-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pharma-ui-dev
  namespace: argocd
  labels:
    env: dev
    app: pharma-ui
    managed-by: terraform
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma

  source:
    # Replace 'ravdy' with your GitHub username after forking zen-gitops
    repoURL: https://github.com/<your-username>/gitops.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/dev/values-pharma-ui.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

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
```

### Step 2: Walk Through Every Field

**`metadata`:**
- `name: pharma-ui-dev` — A unique name for this ArgoCD Application. Convention: `<service>-<env>`.
- `namespace: argocd` — All ArgoCD Application resources live in the `argocd` namespace.
- `finalizers: [resources-finalizer.argocd.argoproj.io]` — When you delete this Application, ArgoCD will **also delete all the Kubernetes resources it created** (Deployment, Service, etc.). Without this finalizer, deleting the Application only removes it from ArgoCD's tracking — the pods keep running.

**`spec.project: pharma`:**
Ties this Application to the `pharma` AppProject we created in section 5.7. ArgoCD checks that the `source.repoURL` is in the project's `sourceRepos` and the `destination.namespace` is in the project's `destinations`. If not, the sync is rejected.

**`spec.source`:**
```yaml
source:
  repoURL: https://github.com/<your-username>/gitops.git
  targetRevision: HEAD
  path: helm-charts
  helm:
    valueFiles:
      - ../envs/dev/values-pharma-ui.yaml
```
- `repoURL` — The Git repository to pull from. Must be listed in the AppProject's `sourceRepos`.
- `targetRevision: HEAD` — Track the latest commit on the default branch. You could also pin to a specific tag or branch.
- `path: helm-charts` — The directory within the repo that contains the Helm chart (`Chart.yaml`).
- `helm.valueFiles` — The values file to use when rendering the chart. The path `../envs/dev/values-pharma-ui.yaml` is **relative to `path`** (i.e., relative to `helm-charts/`). Since the values file is in `envs/dev/` and the chart is in `helm-charts/`, we go up one directory with `..`.

> **Why the relative path trick works:** ArgoCD clones the entire gitops repo, then looks at `path: helm-charts` to find `Chart.yaml`. When it processes `helm.valueFiles`, it resolves paths relative to `helm-charts/`. So `../envs/dev/values-pharma-ui.yaml` resolves to `envs/dev/values-pharma-ui.yaml` in the repo root. This lets us keep the chart and values files in separate directories while ArgoCD can still find them.

**`spec.destination`:**
```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: dev
```
- `server` — The Kubernetes cluster to deploy to. `https://kubernetes.default.svc` is the in-cluster API server (ArgoCD is running in the same cluster it deploys to).
- `namespace: dev` — All rendered resources are deployed into the `dev` namespace.

**`spec.syncPolicy`:**
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false
```
- `automated` — ArgoCD automatically syncs when it detects a change in Git. Without this, you'd have to manually click "Sync" in the ArgoCD UI.
- `prune: true` — If you remove a resource from Git (e.g., delete the Ingress from the values file), ArgoCD **deletes it from the cluster** too. Without `prune`, orphaned resources would remain.
- `selfHeal: true` — If someone manually runs `kubectl edit deployment pharma-ui` and changes something, ArgoCD detects the drift and **reverts it back** to match Git. This enforces Git as the single source of truth.
- `allowEmpty: false` — Safety net. If the rendered chart produces zero resources (which usually means something is broken), ArgoCD refuses to sync instead of deleting everything.

**`syncOptions`:**
```yaml
syncOptions:
  - CreateNamespace=true
  - PrunePropagationPolicy=foreground
  - PruneLast=true
```
- `CreateNamespace=true` — If the `dev` namespace doesn't exist, create it automatically.
- `PrunePropagationPolicy=foreground` — When deleting resources, wait for dependents to be deleted first (e.g., delete pods before deleting the Deployment). Ensures clean deletion order.
- `PruneLast=true` — When syncing, create/update resources first, then prune deleted ones last. Prevents downtime during sync.

**`retry`:**
```yaml
retry:
  limit: 5
  backoff:
    duration: 5s
    factor: 2
    maxDuration: 3m
```
If a sync fails (e.g., temporary network issue pulling the image), ArgoCD retries up to 5 times with exponential backoff: 5s, 10s, 20s, 40s, 80s (capped at 3 minutes).

**`revisionHistoryLimit: 10`:**
ArgoCD keeps a history of the last 10 sync operations. You can see these in the ArgoCD UI and rollback to a previous revision if needed.

### How ArgoCD Watches for Changes

ArgoCD polls the gitops repository every **3 minutes** by default (configurable). Here's the full flow when you push a change:

1. **CI pipeline** (Module 4) builds a new Docker image and pushes it to ECR with tag `sha-abc1234`.
2. CI pipeline **updates** `envs/dev/values-pharma-ui.yaml`, changing `image.tag` from `sha-b8aa312` to `sha-abc1234`, and pushes the commit.
3. Within 3 minutes, ArgoCD detects the new commit.
4. ArgoCD re-renders the Helm chart with the updated values file.
5. ArgoCD compares the rendered manifests to what's currently running in the cluster.
6. ArgoCD applies the diff — in this case, it updates the Deployment's `image` field.
7. Kubernetes performs a **rolling update**: starts a new pod with the new image, waits for it to pass the readiness probe, then terminates the old pod.
8. Zero-downtime deployment complete.

> **Webhook alternative:** Instead of polling every 3 minutes, you can configure a GitHub webhook to notify ArgoCD immediately when a push happens. This reduces deployment latency from up to 3 minutes to seconds.

### Step 3: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add argocd/apps/dev/pharma-ui-app.yaml
git commit -m "feat: add ArgoCD application manifest for pharma-ui dev"
git push origin main
```

> **Tag `gitops` repo: `module-5.8-pharma-ui-argocd-app`**
> ```bash
> cd ~/devops/zenpharma/gitops
> git tag -a module-5.8-pharma-ui-argocd-app -m "Module 5.8: ArgoCD Application for pharma-ui dev"
> git push origin module-5.8-pharma-ui-argocd-app
> ```

---

## Module 5 Summary

| What We Built | Details |
|--------------|---------|
| **GitOps repo structure** | Organized directories for Helm charts, environment configs, ArgoCD apps, and K8s manifests |
| **Kubernetes namespaces** | `dev` namespace for environment isolation |
| **Database init script** | SQL file for 8 PostgreSQL schemas (one per microservice) — executed in Module 6, not here |
| **Raw Kubernetes manifests** | Hands-on deployment of pharma-ui with raw YAML (Deployment, Service, Ingress, ConfigMap) to understand manifest structure before Helm |
| **Shared Helm chart** | 7 templates (Deployment, Service, Ingress, ConfigMap, HPA, ServiceAccount, helpers) with security-hardened defaults |
| **Pharma-UI values (dev)** | Environment-specific overrides for the React frontend — Nginx port 80, lightweight resources, emptyDir volumes |
| **ArgoCD AppProject** | Security boundary defining allowed repos, namespaces, and resource types |
| **ArgoCD Ingress** | HTTPS access to the ArgoCD dashboard via ALB |
| **ArgoCD Application** | Automated sync of pharma-ui with self-heal, pruning, and retry |

| Tag | Repo |
|-----|------|
| `module-5.1-repo-structure` | gitops |
| `module-5.4-raw-manifests` | gitops |
| `module-5.5-helm-chart` | gitops |
| `module-5.6-pharma-ui-values` | gitops |
| `module-5.7-argocd-config` | gitops |
| `module-5.8-pharma-ui-argocd-app` | gitops |

> **Next:** [Module 6 — Deploying Pharma-UI to Dev](MODULE-6-DEPLOYING-PHARMA-UI-TO-DEV.md)
