# Module 7 — Dockerize & Deploy Backend Microservices

> Dockerize and deploy all 8 backend microservices with full CI/CD pipelines, security scanning, and GitOps-driven deployment.
> Estimated time: 3-4 hours.

---

## 7.1 Overview of Backend Architecture

In Module 6 we deployed the frontend (pharma-ui). The frontend shows a login screen and navigation, but every API call fails because there are no backend services running yet. In this module we Dockerize all 8 backend microservices, create CI/CD pipelines, and deploy them to the EKS cluster.

### The 8 Microservices

| Service | Language | Framework | Port | Description |
|---------|----------|-----------|------|-------------|
| api-gateway | Java | Spring Boot | 8080 | Routes all external requests to internal services |
| auth-service | Java | Spring Boot | 8081 | Authentication, JWT token management |
| drug-catalog-service | Java | Spring Boot | 8082 | Drug catalog CRUD operations |
| inventory-service | Java | Spring Boot | 8083 | Inventory tracking and management |
| supplier-service | Java | Spring Boot | 8084 | Supplier management |
| manufacturing-service | Java | Spring Boot | 8085 | Manufacturing batch tracking |
| qc-service | Java | Spring Boot | 8086 | Quality control inspections |
| notification-service | Node.js | Express | 3000 | Email/SMS notifications |

### API Gateway Pattern

All services share the same ALB (via the `pharma-dev` ingress group). The ALB routes traffic based on the URL path — `/api/*` goes to api-gateway, and everything else (`/`) goes to pharma-ui. The api-gateway then routes to the appropriate internal service:

```
                          ┌─ /     → pharma-ui (80)        ← React app (Nginx)
Browser → ALB (pharma-dev)│
                          └─ /api/ → api-gateway (8080)    ← Spring Boot gateway
                                        → auth-service (8081)
                                        → drug-catalog-service (8082)
                                        → inventory-service (8083)
                                        → supplier-service (8084)
                                        → manufacturing-service (8085)
                                        → qc-service (8086)
                                        → notification-service (3000)
```

Pharma-ui is a static React app served by Nginx. It makes API calls to `/api/*`, which the ALB routes to api-gateway. The api-gateway then forwards to the correct backend service based on the path.

> **Why an API Gateway?**
> - Single entry point simplifies security (one place for auth, rate limiting, CORS)
> - Frontend only needs to know one URL (`/api`), not 7 different service endpoints
> - Cross-cutting concerns (logging, tracing, circuit breakers) live in one place
> - In production, you might replace this with AWS API Gateway or Kong, but a Spring Boot gateway works well for learning

### Inter-Service Communication

Inside Kubernetes, services communicate over HTTP using **Kubernetes Service DNS names**. When we deploy `auth-service` with a Kubernetes Service, it becomes reachable at `http://auth-service:8081` from any pod in the same namespace.

The api-gateway configuration includes these URLs:

```yaml
AUTH_SERVICE_URL: "http://auth-service:8081"
DRUG_CATALOG_URL: "http://drug-catalog-service:8082"
NOTIFICATION_URL: "http://notification-service:3000"
QC_SERVICE_URL: "http://qc-service:8086"
```

> **Why not use IP addresses?**
> Pod IPs change every time a pod restarts. Kubernetes DNS names are stable — they resolve to whatever IP the Service's pod currently has. This is the same reason you use domain names on the internet instead of IP addresses.

---

## 7.2 Create Dockerfiles for Each Microservice

Each microservice needs a Dockerfile. There are two patterns: one for the 7 Java/Spring Boot services and one for the Node.js notification-service.

### Step 1: Java Dockerfile (7 services)

All 7 Java services share the **same Dockerfile structure** — the only difference is the `EXPOSE` port, which documents which port each service listens on.

Run all commands from the backend repo root:

```bash
cd ~/devops/zenpharma/backend
```

Here's the Dockerfile pattern (using api-gateway as the example):

```dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
RUN groupadd -r pharma && useradd -r -g pharma pharma
COPY target/*.jar app.jar
USER pharma
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "-Djava.security.egd=file:/dev/./urandom", "app.jar"]
```

**Line-by-line explanation:**

| Line | Explanation |
|------|-------------|
| `FROM eclipse-temurin:17-jre` | Uses the Eclipse Temurin Java 17 **JRE** (Java Runtime Environment), not the JDK. The JRE is smaller (~200 MB vs ~400 MB) because it does not include the compiler, debugger, or development tools. At runtime, we only need to *run* the jar, not compile code. |
| `WORKDIR /app` | Sets the working directory inside the container. |
| `RUN groupadd -r pharma && useradd -r -g pharma pharma` | Creates a non-root system user `pharma`. Running as root inside containers is a security risk — if an attacker escapes the JVM, they would have root access to the container filesystem. |
| `COPY target/*.jar app.jar` | Copies the Spring Boot "fat jar" from the Maven build output. Spring Boot packages everything (application code + all dependencies) into a single executable jar file. This means Maven must run **before** `docker build`. |
| `USER pharma` | Switches to the non-root user for all subsequent commands and the final ENTRYPOINT. |
| `EXPOSE <port>` | Documents which port the service listens on. Each service uses a different port (see table below). This is informational — the actual port is set by Spring Boot in `application.yml`. |
| `ENTRYPOINT ["java", "-jar", "-Djava.security.egd=file:/dev/./urandom", "app.jar"]` | Starts the Spring Boot application. The `-Djava.security.egd` flag tells Java to use `/dev/urandom` instead of `/dev/random` for cryptographic operations, which avoids blocking on entropy-starved containers. |

> **Why JRE instead of JDK?**
> The JDK includes `javac` (compiler), `jdb` (debugger), `jconsole`, and other development tools. None of these are needed at runtime. Using the JRE reduces image size by ~200 MB and removes tools that an attacker could exploit if they gain shell access to the container.

> **Why `target/*.jar`? Where does it come from?**
> Spring Boot's Maven plugin (`spring-boot-maven-plugin`) creates a "fat jar" during `mvn package`. This jar contains the compiled application **plus** all Maven dependencies embedded inside. In CI, the reusable workflow runs `mvn verify` before `docker build`, which produces the jar. Locally, you would run `mvn package -DskipTests` first.

Create the Dockerfile for each Java service with the correct port:

```bash
# Run from ~/devops/zenpharma/backend

for pair in api-gateway:8080 auth-service:8081 drug-catalog-service:8082 \
            inventory-service:8083 supplier-service:8084 \
            manufacturing-service:8085 qc-service:8086; do
  svc="${pair%%:*}"
  port="${pair##*:}"
  cat > ${svc}/Dockerfile << EOF
FROM eclipse-temurin:17-jre
WORKDIR /app
RUN groupadd -r pharma && useradd -r -g pharma pharma
COPY target/*.jar app.jar
USER pharma
EXPOSE ${port}
ENTRYPOINT ["java", "-jar", "-Djava.security.egd=file:/dev/./urandom", "app.jar"]
EOF
  echo "Created ${svc}/Dockerfile (EXPOSE ${port})"
done
```

**Port mapping:**

| Service | EXPOSE Port | Set By |
|---------|:-:|---|
| api-gateway | 8080 | `application.yml: server.port: 8080` |
| auth-service | 8081 | `application.yml: server.port: 8081` |
| drug-catalog-service | 8082 | `application.yml: server.port: 8082` |
| inventory-service | 8083 | `application.yml: server.port: 8083` |
| supplier-service | 8084 | `application.yml: server.port: 8084` |
| manufacturing-service | 8085 | `application.yml: server.port: 8085` |
| qc-service | 8086 | `application.yml: server.port: 8086` |

> **Does EXPOSE actually matter?** Technically no — `EXPOSE` is documentation only. The real port is set by Spring Boot in `application.yml`. But having `EXPOSE` match the actual port helps learners and tools understand which port the container uses without reading the Spring config.

### Step 2: Node.js Dockerfile (notification-service)

The notification-service uses Node.js, so it needs a different Dockerfile with a multi-stage build:

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
FROM node:22-alpine
WORKDIR /app
RUN addgroup -S pharma && adduser -S pharma -G pharma
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src
COPY package.json .
USER pharma
EXPOSE 3000
CMD ["node", "src/index.js"]
```

**Key differences from the Java Dockerfile:**

| Aspect | Java Services | notification-service |
|--------|--------------|---------------------|
| **Build strategy** | Single stage (jar is pre-built by Maven in CI) | Multi-stage (install deps in builder, copy to runtime) |
| **Base image** | `eclipse-temurin:17-jre` (~200 MB) | `node:22-alpine` (~120 MB) |
| **Dependencies** | Embedded in the fat jar | `npm ci --omit=dev` installs only production dependencies |
| **Port** | 8080 | 3000 |
| **Start command** | `java -jar app.jar` | `node src/index.js` |
| **User creation** | `groupadd`/`useradd` (Debian-based) | `addgroup -S`/`adduser -S` (Alpine-based) |

> **Why multi-stage for Node.js but not Java?**
> The Java Dockerfile copies a pre-built jar — Maven already ran in CI. For Node.js, we need `npm ci` to install dependencies. The builder stage runs `npm ci --omit=dev` and the runtime stage copies only the resulting `node_modules`. This keeps the final image clean — no npm cache, no build scripts, no devDependencies.

Create the notification-service Dockerfile:

```bash
cat > notification-service/Dockerfile << 'EOF'
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
FROM node:22-alpine
WORKDIR /app
RUN addgroup -S pharma && adduser -S pharma -G pharma
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src
COPY package.json .
USER pharma
EXPOSE 3000
CMD ["node", "src/index.js"]
EOF
echo "Created notification-service/Dockerfile"
```

### Step 3: Commit and Push Dockerfiles

```bash
cd ~/devops/zenpharma/backend
git add */Dockerfile
git commit -m "feat: add Dockerfiles for all 8 backend microservices"
git push
```

> **Tag `backend` repo: `module-7.2-dockerfiles`**
>
> ```bash
> git tag -a module-7.2-dockerfiles -m "Module 7.2: Dockerfiles for all 8 backend microservices"
> git push origin module-7.2-dockerfiles
> ```

---

## 7.3 Create Reusable GitHub Workflows for Backend

The backend repository is a **monorepo** — all 8 services live in subdirectories of one repo. Without reusable workflows, we would copy-paste the same CI/CD logic 8 times. Instead, we create reusable workflows that each per-service workflow calls.

### Step 1: Create `_java-build.yml` — Reusable Java Build Pipeline

This is the full CI/CD pipeline for Java services. Create `.github/workflows/_java-build.yml`:

```bash
cd ~/devops/zenpharma/backend
mkdir -p .github/workflows
```

```yaml
# Reusable — Java Build + SAST + Container Security + ECR Push + Cosign Sign
#
# Called by every Java/Spring Boot per-service ci-*.yml workflow.
# Pipeline stages (all in a single job, sequential):
#   1. Maven verify  — unit/integration tests + JaCoCo coverage gate (>= 80%)
#   2. SonarCloud    — SAST + code quality + coverage analysis
#   3. OWASP Dep Chk — CVSS >= 7.0 reported (non-blocking)
#   4. Docker build  — multi-stage, non-root UID/GID 1000
#   5. Trivy         — image scan, fail on HIGH/CRITICAL CVEs
#   6. ECR push      — tagged as sha-<7chars>
#   7. Cosign sign   — keyless, GitHub OIDC → Fulcio CA → Rekor transparency log
#
# Outputs: image-tag (sha-<7chars>), registry (ECR registry URL)

name: Reusable — Java Build & Security Gates

on:
  workflow_call:
    inputs:
      service-name:
        description: "Service name (e.g. api-gateway)"
        required: true
        type: string
      service-dir:
        description: "Service directory relative to repo root (e.g. api-gateway)"
        required: true
        type: string
      ecr-repository:
        description: "ECR repository name"
        required: true
        type: string
      aws-region:
        description: "AWS region where ECR lives"
        required: false
        type: string
        default: us-east-1
      needs-database:
        description: "Start a PostgreSQL 15 container for integration tests"
        required: false
        type: boolean
        default: false
    outputs:
      image-tag:
        description: "Docker image tag pushed to ECR (format: sha-<7chars>)"
        value: ${{ jobs.build-and-push.outputs.image_tag }}
      registry:
        description: "ECR registry base URL"
        value: ${{ jobs.build-and-push.outputs.registry }}
    secrets:
      AWS_ACCOUNT_ID:
        required: true
      NVD_API_KEY:
        description: "NIST NVD API key — higher rate limits + faster OWASP NVD updates"
        required: false
```

The YAML above shows only the `workflow_call` interface (inputs, outputs, secrets). The full file is ~250 lines and contains the complete `jobs:` block with all 7 pipeline stages. **Copy the full file from the course reference materials:**

```bash
cp /path/to/course-materials/backend/.github/workflows/_java-build.yml .github/workflows/
```

> **Tip:** These workflow files are too long to type manually. Always copy them from the course reference materials, then review the key sections explained below.

**Inputs explained:**

| Input | Purpose |
|-------|---------|
| `service-name` | Human-readable name, used in job titles and artifact names |
| `service-dir` | Directory containing the service code (e.g., `api-gateway`). Used for `cd` and path references |
| `ecr-repository` | ECR repository name — must match what Terraform created in Module 1 |
| `aws-region` | AWS region for ECR (defaults to `us-east-1`) |
| `needs-database` | If `true`, starts a PostgreSQL 15 container for integration tests. Services like auth-service and inventory-service need this |

**Outputs:** The workflow outputs `image-tag` (e.g., `sha-a1b2c3d`) and `registry` (ECR base URL). The calling workflow uses these to update the GitOps values file.

**What the `jobs:` block does (key steps in order):**

```yaml
jobs:
  build-and-push:
    steps:
      # 1. Checkout code
      - uses: actions/checkout@v5

      # 2. Set image tag from git SHA
      - run: echo "image_tag=sha-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      # 3. Setup Java 17
      - uses: actions/setup-java@v4
        with: { java-version: '17', distribution: temurin, cache: maven }

      # 4. Maven verify (compile + test + JaCoCo coverage)
      - run: cd ${{ inputs.service-dir }} && mvn verify --no-transfer-progress

      # 5. SonarCloud scan
      - uses: SonarSource/sonarcloud-github-action@v3

      # 6. Configure AWS credentials (OIDC — no static keys)
      - uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/pharma-dev-github-actions-role

      # 7. Login to ECR
      - uses: aws-actions/amazon-ecr-login@v2

      # 8. Docker build
      - run: docker build -t $REGISTRY/$ECR_REPO:$IMAGE_TAG ${{ inputs.service-dir }}

      # 9. Trivy scan
      - uses: aquasecurity/trivy-action@master

      # 10. Push to ECR
      - run: docker push $REGISTRY/$ECR_REPO:$IMAGE_TAG
```

> **Note:** This is a simplified view of the key steps. The actual file includes additional steps for OWASP Dependency Check, Cosign signing, coverage uploads, and error handling. Review the full file after copying to understand every step.

**The 7 pipeline stages in detail:**

1. **Maven verify + JaCoCo** — Compiles code, runs unit/integration tests, and enforces a minimum 80% code coverage gate. If coverage drops below 80%, the build fails.

2. **SonarCloud** — Cloud-based SAST and code-quality analysis. Detects security vulnerabilities, code smells, bugs, and duplications. Receives JaCoCo coverage data for unified quality reporting. Posts inline PR comments on new issues.

3. **OWASP Dependency Check** — Scans Maven dependencies against the NIST National Vulnerability Database. Reports any CVE with CVSS score >= 7.0. Non-blocking (advisory only) to avoid breaking builds on unpatched upstream vulnerabilities.

4. **Docker build** — Builds the container image using the Dockerfile from Section 7.2.

5. **Trivy** — Scans the built Docker image for known vulnerabilities in OS packages and application libraries.

6. **ECR push** — Pushes the image to Amazon ECR with a `sha-<7chars>` tag derived from the Git commit SHA. This makes every image traceable back to the exact commit that produced it.

7. **Cosign keyless signing** — Signs the image cryptographically without managing private keys.

> **Why Cosign keyless signing?**
> Traditional image signing requires generating, storing, and rotating private keys — a significant operational burden. Cosign's keyless mode uses GitHub's OIDC identity token to prove "this image was built by this GitHub Actions workflow in this repository." The flow:
> 1. GitHub Actions provides an OIDC token (proves the workflow identity)
> 2. Fulcio CA (run by Sigstore) issues a short-lived signing certificate based on the OIDC token
> 3. The image is signed with this ephemeral certificate
> 4. The signature is recorded in the Rekor transparency log (public, append-only)
>
> Later, Kyverno (a Kubernetes admission controller) can verify that only Cosign-signed images are admitted into the cluster.

### Step 2: Create `_node-build.yml` — Reusable Node.js Build Pipeline

Same pipeline concept, adapted for Node.js. Create `.github/workflows/_node-build.yml`:

```yaml
# Reusable — Node.js Build + SAST + Container Security + ECR Push + Cosign Sign
#
# Called by the notification-service ci-notification.yml workflow.
# Pipeline stages (all in a single job, sequential):
#   1. npm ci + ESLint      — install dependencies + lint (--max-warnings 0)
#   2. npm test             — unit tests with Jest + coverage gate (>= 80%)
#   3. SonarCloud           — SAST + code quality + coverage analysis
#   4. npm audit            — fail if any HIGH/CRITICAL vulnerabilities found
#   5. Docker build         — multi-stage, non-root UID/GID 1000
#   6. Trivy                — image scan, fail on HIGH/CRITICAL CVEs
#   7. ECR push             — tagged as sha-<7chars>
#   8. Cosign sign          — keyless, GitHub OIDC → Fulcio CA → Rekor transparency log
#
# Outputs: image-tag (sha-<7chars>), registry (ECR registry URL)

name: Reusable — Node.js Build & Security Gates

on:
  workflow_call:
    inputs:
      service-name:
        description: "Service name (e.g. notification-service)"
        required: true
        type: string
      service-dir:
        description: "Service directory relative to repo root"
        required: true
        type: string
      ecr-repository:
        description: "ECR repository name"
        required: true
        type: string
      node-version:
        description: "Node.js version to use"
        required: false
        type: string
        default: '22'
      aws-region:
        description: "AWS region where ECR lives"
        required: false
        type: string
        default: us-east-1
    outputs:
      image-tag:
        description: "Docker image tag pushed to ECR (format: sha-<7chars>)"
        value: ${{ jobs.build-and-push.outputs.image_tag }}
      registry:
        description: "ECR registry base URL"
        value: ${{ jobs.build-and-push.outputs.registry }}
    secrets:
      AWS_ACCOUNT_ID:
        required: true
```

**Copy the full file from the course reference materials:**

```bash
cp /path/to/course-materials/backend/.github/workflows/_node-build.yml .github/workflows/
```

**Differences from the Java pipeline:**

| Stage | Java (`_java-build.yml`) | Node.js (`_node-build.yml`) |
|-------|--------------------------|----------------------------|
| Build/Test | `mvn verify` + JaCoCo | `npm ci` + ESLint + `npm test` (Jest) |
| SAST | SonarCloud (Java analysis) | SonarCloud (JavaScript analysis) |
| Dependency scan | OWASP Dependency Check (Maven plugin) | `npm audit --audit-level=high` |
| Coverage | JaCoCo >= 80% | Jest >= 80% |

The Docker build, Trivy scan, ECR push, and Cosign signing stages are identical between the two pipelines.

### Step 3: Create `_java-pr-check.yml` — Lightweight PR Workflow

For feature branches and PRs, we want **fast feedback** — developers should not wait 15+ minutes for a Docker build and ECR push when they just want to know if their code compiles and passes tests.

**Copy the full file from the course reference materials:**

```bash
cp /path/to/course-materials/backend/.github/workflows/_java-pr-check.yml .github/workflows/
```

The `workflow_call` interface:

```yaml
# Reusable — Lightweight Java PR / Feature Branch Check
#
# Used by ci-pr-<service>.yml workflows on feat-*, fix-*, chore-* branches and PRs.
# Intentionally stops before Docker — no image build, no ECR push, no Cosign.
# Goal: fast feedback (~5 min) for developers without burning ECR storage or runner time.
#
# Stages:
#   1. Maven verify  — unit/integration tests + JaCoCo coverage
#   2. SonarCloud    — SAST + code quality + coverage analysis
#   3. OWASP Dep Chk — CVSS >= 7.0 reported (non-blocking)

name: Reusable — Java PR Check (no Docker)

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      service-dir:
        required: true
        type: string
      needs-database:
        required: false
        type: boolean
        default: false
    secrets:
      NVD_API_KEY:
        description: "NIST NVD API key — higher rate limits + faster OWASP NVD updates"
        required: false
```

> **Why a separate PR workflow?**
> The full `_java-build.yml` pipeline runs 7 stages including Docker build, Trivy scan, ECR push, and Cosign signing. That takes ~15 minutes and creates an ECR image that will never be deployed (since feature branches do not deploy). The PR check runs only compile + test + SAST (~5 minutes), giving developers fast feedback without wasting ECR storage or runner minutes.

### Step 4: Create `_node-pr-check.yml` — Lightweight Node.js PR Workflow

The Node.js equivalent for notification-service PRs.

```bash
cp /path/to/course-materials/backend/.github/workflows/_node-pr-check.yml .github/workflows/
```

The `workflow_call` interface:

```yaml
# Reusable — Lightweight Node.js PR / Feature Branch Check
#
# Used by ci-pr-notification.yml on feat-*, fix-*, chore-* branches and PRs.
# No Docker, no ECR, no Cosign. Fast feedback for developers.
#
# Stages:
#   1. ESLint        — lint (--max-warnings 0)
#   2. Jest          — unit tests + coverage
#   3. SonarCloud    — SAST + code quality + coverage analysis
#   4. npm audit     — fail on HIGH/CRITICAL vulnerabilities

name: Reusable — Node.js PR Check (no Docker)

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      service-dir:
        required: true
        type: string
      node-version:
        required: false
        type: string
        default: '22'
```

> **Don't commit yet** — we will commit all workflows together with the Dockerfiles in section 7.5.

---

## 7.4 Create Per-Service CI Workflows

Each service needs two workflow files:
- **`ci-<service>.yml`** — Full CI/CD pipeline, runs on push to `develop`/`release/*` branches
- **`ci-pr-<service>.yml`** — Lightweight PR check, runs on feature branches and PRs

### Step 1: Per-Service CI Workflow (Full Pipeline)

Here is the complete `ci-api-gateway.yml` as the reference example:

```yaml
name: CI/CD — api-gateway

# Triggers on push to develop / release branches only.
# Feature branches and PRs → ci-pr-api-gateway.yml
#
# Jobs:
#   build       → Maven + SonarCloud + Docker + ECR + Cosign
#   deploy-dev  → direct commit to envs/dev/ → ArgoCD auto-syncs
#
# PROD promotion → promote-prod.yml (workflow_dispatch)

on:
  push:
    branches: [develop, 'release/**']
    paths:
      - 'api-gateway/**'
      - '.github/workflows/ci-api-gateway.yml'
  workflow_dispatch:
    inputs:
      ref:
        description: 'Branch to build'
        required: false
        default: 'develop'

permissions:
  id-token: write
  contents: read
  actions: read

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: api-gateway
  GITOPS_REPO: ${{ vars.GITOPS_REPO }}

jobs:

  # ── 1. Build: Maven · SonarCloud · OWASP Dep · Trivy · ECR · Cosign ────────
  build:
    name: Build & Security Gates
    uses: ./.github/workflows/_java-build.yml
    with:
      service-name: api-gateway
      service-dir: api-gateway
      ecr-repository: api-gateway
      aws-region: us-east-1
      needs-database: false
    secrets: inherit

  # ── 2. Deploy to DEV (direct commit — ArgoCD auto-syncs) ──────────────────
  deploy-dev:
    name: Deploy — DEV
    needs: build
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v5

      - name: Checkout GitOps repo
        uses: actions/checkout@v5
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: _gitops

      - name: Update image tag — DEV
        env:
          IMAGE_TAG: ${{ needs.build.outputs.image-tag }}
        run: |
          yq e ".image.tag = \"${IMAGE_TAG}\"" -i \
            "_gitops/envs/dev/values-api-gateway.yaml"
          cd _gitops
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git diff --staged --quiet && echo "Already up to date" && exit 0
          git commit -m "ci(dev): update api-gateway → ${IMAGE_TAG}"
          git pull --rebase origin main
          git push

```

**Walkthrough of the 2 jobs:**

**Job 1: `build`** — Calls the reusable `_java-build.yml` workflow. This runs all 7 stages (Maven, SonarCloud, OWASP, Docker, Trivy, ECR, Cosign). For notification-service, this calls `_node-build.yml` instead.

**Job 2: `deploy-dev`** — After a successful build, this job:
1. Checks out the GitOps repo (`zen-gitops`)
2. Uses `yq` to update `.image.tag` in `envs/dev/values-api-gateway.yaml` to the new SHA tag
3. Commits and pushes directly to the `main` branch of zen-gitops
4. ArgoCD detects the change and auto-syncs the deployment

> **Why `git pull --rebase origin main` before push?**
> In a monorepo with 8 services, multiple CI pipelines may finish around the same time and all try to push to the zen-gitops repo. Without `pull --rebase`, the second push would fail with "rejected — non-fast-forward." The rebase pulls the latest changes (from other services) and replays our commit on top, avoiding merge conflicts.

> **Why `paths` filtering?**
> The `paths` filter ensures that a change to `auth-service/src/...` only triggers the auth-service pipeline, not all 8 pipelines. Without this, every commit would trigger 8 parallel builds even if only one service changed. This is a key benefit of the monorepo approach — granular triggering.

### Step 2: Per-Service PR Check Workflow

Here is the complete `ci-pr-api-gateway.yml`:

```yaml
name: PR Check — api-gateway

# Runs on feature/fix/chore branches and PRs to develop.
# Lint + test + SAST only. No Docker, no ECR, no deploy.

on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'api-gateway/**'
      - '.github/workflows/ci-pr-api-gateway.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'api-gateway/**'

permissions:
  contents: read
  pull-requests: read
  actions: read

jobs:
  pr-check:
    uses: ./.github/workflows/_java-pr-check.yml
    with:
      service-name: api-gateway
      service-dir: api-gateway
      needs-database: false
    secrets: inherit
```

This is much simpler — it just calls the lightweight `_java-pr-check.yml` reusable workflow. No Docker, no ECR, no deploy.

### Step 3: The Notification Service Difference

The notification-service CI workflow (`ci-notification.yml`) calls `_node-build.yml` instead of `_java-build.yml`:

```yaml
name: CI/CD — notification-service

on:
  push:
    branches: [develop, 'release/**']
    paths:
      - 'notification-service/**'
      - '.github/workflows/ci-notification.yml'
  workflow_dispatch:
    inputs:
      ref:
        description: 'Branch to build'
        required: false
        default: 'develop'

permissions:
  id-token: write
  contents: read
  actions: read

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: notification-service
  GITOPS_REPO: ${{ vars.GITOPS_REPO }}

jobs:

  build:
    name: Build & Security Gates
    uses: ./.github/workflows/_node-build.yml
    with:
      service-name: notification-service
      service-dir: notification-service
      ecr-repository: notification-service
      node-version: '22'
      aws-region: us-east-1
    secrets: inherit

  deploy-dev:
    name: Deploy — DEV
    needs: build
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v5

      - name: Checkout GitOps repo
        uses: actions/checkout@v5
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: _gitops

      - name: Update image tag — DEV
        env:
          IMAGE_TAG: ${{ needs.build.outputs.image-tag }}
        run: |
          yq e ".image.tag = \"${IMAGE_TAG}\"" -i \
            "_gitops/envs/dev/values-notification-service.yaml"
          cd _gitops
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git diff --staged --quiet && echo "Already up to date" && exit 0
          git commit -m "ci(dev): update notification-service → ${IMAGE_TAG}"
          git pull --rebase origin main
          git push
```

And `ci-pr-notification.yml` calls `_node-pr-check.yml`:

```yaml
name: PR Check — notification-service

on:
  push:
    branches: ['feat-*', 'fix-*', 'chore-*']
    paths:
      - 'notification-service/**'
      - '.github/workflows/ci-pr-notification.yml'
  pull_request:
    branches: [develop]
    paths:
      - 'notification-service/**'

permissions:
  contents: read
  pull-requests: read
  actions: read

jobs:
  pr-check:
    uses: ./.github/workflows/_node-pr-check.yml
    with:
      service-name: notification-service
      service-dir: notification-service
      node-version: '22'
    secrets: inherit
```

### Step 4: All 16 Workflow Files

For each of the 8 services, you need 2 workflow files. Here is the complete list:

| CI/CD Workflow (full pipeline) | PR Check Workflow (lightweight) |
|-------------------------------|-------------------------------|
| `ci-api-gateway.yml` | `ci-pr-api-gateway.yml` |
| `ci-auth-service.yml` | `ci-pr-auth-service.yml` |
| `ci-drug-catalog.yml` | `ci-pr-drug-catalog.yml` |
| `ci-inventory-service.yml` | `ci-pr-inventory-service.yml` |
| `ci-manufacturing-service.yml` | `ci-pr-manufacturing-service.yml` |
| `ci-supplier-service.yml` | `ci-pr-supplier-service.yml` |
| `ci-qc-service.yml` | `ci-pr-qc-service.yml` |
| `ci-notification.yml` | `ci-pr-notification.yml` |

Plus the reusable workflows and promotion workflows:

| Reusable / Shared Workflows |
|-----------------------------|
| `_java-build.yml` |
| `_java-pr-check.yml` |
| `_node-build.yml` |
| `_node-pr-check.yml` |
| `promote-qa.yml` |
| `promote-prod.yml` |

**Total: 22 workflow files** (16 per-service + 4 reusable + 2 promotion)

The pattern is identical across all Java services — only `service-name`, `service-dir`, `ecr-repository`, and `needs-database` change. Create the remaining workflows by copying `ci-api-gateway.yml` and `ci-pr-api-gateway.yml` and replacing the service-specific values.

> **Why not use a matrix strategy?**
> GitHub Actions supports matrix builds (e.g., `strategy: matrix: service: [api-gateway, auth-service, ...]`). However, monorepo path filtering does not work with matrices — the matrix would trigger all 8 services on every push. Per-service workflow files with `paths` filters give us precise triggering: change one service, build only that service.

> **Don't commit yet** — we will commit all per-service workflows together with the Dockerfiles and reusable workflows in section 7.5.

---

## 7.5 Create Backend Promotion Workflows

In Module 4 (frontend), we created separate QA and PROD promotion workflows for pharma-ui. For the backend, we use the same pattern — **consolidated promotion workflows** with a service dropdown so one workflow handles all services.

### Step 1: Create `promote-qa.yml`

Create `.github/workflows/promote-qa.yml`:

```yaml
name: Promote to QA

# Triggered manually — picks up the image currently running in DEV and opens
# a PR in the GitOps repo to promote it to QA.

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to promote (use the gitops values file name)'
        required: true
        type: choice
        options:
          - api-gateway
          - auth-service
          - catalog-service
          - inventory-service
          - manufacturing-service
          - notification-service
          - supplier-service

permissions:
  id-token: write
  contents: read

env:
  GITOPS_REPO: ${{ vars.GITOPS_REPO }}

jobs:

  promote:
    name: Open QA Promotion PR
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v5

      - name: Checkout GitOps repo
        uses: actions/checkout@v5
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: _gitops

      - name: Install yq
        run: |
          wget -qO /usr/local/bin/yq \
            https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          chmod +x /usr/local/bin/yq

      - name: Read image tag from DEV
        env:
          SVC: ${{ inputs.service }}
        run: |
          DEV_VALUES="_gitops/envs/dev/values-${SVC}.yaml"
          if [[ ! -f "$DEV_VALUES" ]]; then
            echo "::error::DEV values file not found: ${DEV_VALUES}"
            exit 1
          fi
          IMAGE_TAG=$(yq e '.image.tag' "$DEV_VALUES")
          if [[ -z "$IMAGE_TAG" || "$IMAGE_TAG" == "null" ]]; then
            echo "::error::Could not read .image.tag from $DEV_VALUES"
            exit 1
          fi
          echo "IMAGE_TAG=${IMAGE_TAG}" >> "$GITHUB_ENV"
          echo "Promoting image: ${IMAGE_TAG}"

      - name: Validate QA values file exists
        env:
          SVC: ${{ inputs.service }}
        run: |
          QA_VALUES="_gitops/envs/qa/values-${SVC}.yaml"
          if [[ ! -f "$QA_VALUES" ]]; then
            echo "::error::QA values file not found: ${QA_VALUES}"
            echo "::error::Create envs/qa/values-${SVC}.yaml first (Module 8)."
            exit 1
          fi
          echo "QA_VALUES=envs/qa/values-${SVC}.yaml" >> "$GITHUB_ENV"

      - name: Create branch, patch values, open PR
        env:
          GH_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          SVC: ${{ inputs.service }}
          ACTOR: ${{ github.actor }}
          SERVER_URL: ${{ github.server_url }}
          REPOSITORY: ${{ github.repository }}
          RUN_ID: ${{ github.run_id }}
          GITOPS_REPO_NAME: ${{ env.GITOPS_REPO }}
        run: |
          BRANCH="promote/qa/${SVC}/${IMAGE_TAG}"

          cd _gitops
          git checkout -b "$BRANCH"
          yq e ".image.tag = \"${IMAGE_TAG}\"" -i "${QA_VALUES}"
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          if git diff --staged --quiet; then
            echo "::notice::QA already has image ${IMAGE_TAG} — no PR needed."
            exit 0
          fi
          git commit -m "promote(qa): ${SVC} → ${IMAGE_TAG}"
          git push origin "$BRANCH"

          gh pr create \
            --repo "$GITOPS_REPO_NAME" \
            --title "promote(qa): ${SVC} → ${IMAGE_TAG}" \
            --body "## QA Promotion

          Service : ${SVC}
          Image   : ${IMAGE_TAG}
          Promoted from DEV by : ${ACTOR}
          Workflow run : ${SERVER_URL}/${REPOSITORY}/actions/runs/${RUN_ID}

          Merging this PR updates ${QA_VALUES}. ArgoCD will auto-sync QA." \
            --base main \
            --head "$BRANCH"

          echo "::notice title=QA PR opened::Review and merge in ${GITOPS_REPO_NAME}."
```

### Step 2: Create `promote-prod.yml`

Create `.github/workflows/promote-prod.yml`:

```yaml
name: Promote to PROD

# Triggered manually — picks up the image currently running in QA and opens
# a PR in the GitOps repo (vars.GITOPS_REPO) to promote it to PROD.
# PROD ArgoCD app (pharma-prod) requires manual sync after merge.

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to promote (use the zen-gitops name)'
        required: true
        type: choice
        options:
          - api-gateway
          - auth-service
          - catalog-service
          - inventory-service
          - manufacturing-service
          - notification-service
          - supplier-service

permissions:
  id-token: write
  contents: read

env:
  GITOPS_REPO: ${{ vars.GITOPS_REPO }}

jobs:

  promote:
    name: Open PROD Promotion PR
    runs-on: ubuntu-latest
    environment: prod

    steps:
      - uses: actions/checkout@v5

      - name: Checkout GitOps repo
        uses: actions/checkout@v5
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: _gitops

      - name: Install yq
        run: |
          wget -qO /usr/local/bin/yq \
            https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          chmod +x /usr/local/bin/yq

      - name: Validate QA values file exists
        env:
          SVC: ${{ inputs.service }}
        run: |
          QA_VALUES="_gitops/envs/qa/values-${SVC}.yaml"
          if [[ ! -f "$QA_VALUES" ]]; then
            echo "::error::QA values file not found: ${QA_VALUES}"
            echo "::error::Cannot determine which image is running in QA. Create the QA values file first."
            exit 1
          fi
          echo "QA_VALUES=${QA_VALUES}" >> "$GITHUB_ENV"

      - name: Read image tag from QA
        run: |
          IMAGE_TAG=$(yq e '.image.tag' "$QA_VALUES")
          if [[ -z "$IMAGE_TAG" || "$IMAGE_TAG" == "null" ]]; then
            echo "::error::Could not read .image.tag from $QA_VALUES"
            exit 1
          fi
          echo "IMAGE_TAG=${IMAGE_TAG}" >> "$GITHUB_ENV"
          echo "Promoting image: ${IMAGE_TAG}"

      - name: Validate PROD values file exists
        env:
          SVC: ${{ inputs.service }}
          GITOPS_REPO_NAME: ${{ env.GITOPS_REPO }}
        run: |
          PROD_VALUES="_gitops/envs/prod/values-${SVC}.yaml"
          if [[ ! -f "$PROD_VALUES" ]]; then
            echo "::error::PROD values file not found: ${PROD_VALUES}"
            echo "::error::Create envs/prod/values-${SVC}.yaml in ${GITOPS_REPO_NAME} to enable PROD promotion."
            exit 1
          fi
          echo "PROD_VALUES=envs/prod/values-${SVC}.yaml" >> "$GITHUB_ENV"

      - name: Create branch, patch values, open PR
        env:
          GH_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          SVC: ${{ inputs.service }}
          ACTOR: ${{ github.actor }}
          SERVER_URL: ${{ github.server_url }}
          REPOSITORY: ${{ github.repository }}
          RUN_ID: ${{ github.run_id }}
          GITOPS_REPO_NAME: ${{ env.GITOPS_REPO }}
        run: |
          BRANCH="promote/prod/${SVC}/${IMAGE_TAG}"

          cd _gitops
          git checkout -b "$BRANCH"
          yq e ".image.tag = \"${IMAGE_TAG}\"" -i "${PROD_VALUES}"
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          if git diff --staged --quiet; then
            echo "::notice::PROD already has image ${IMAGE_TAG} — no PR needed."
            exit 0
          fi
          git commit -m "promote(prod): ${SVC} → ${IMAGE_TAG}"
          git push origin "$BRANCH"

          printf '%s\n' \
            "## PROD Promotion" \
            "" \
            "Service : ${SVC}" \
            "Image   : ${IMAGE_TAG}" \
            "Promoted from QA by : ${ACTOR}" \
            "Workflow run : ${SERVER_URL}/${REPOSITORY}/actions/runs/${RUN_ID}" \
            "" \
            "## Pre-merge Checklist" \
            "- [ ] QA sign-off received" \
            "- [ ] Change ticket / CAB approved" \
            "- [ ] No unexpected config changes in this diff" \
            "- [ ] Runbook link added to change ticket" \
            "" \
            "## Post-merge" \
            "ArgoCD app \`pharma-prod/${SVC}\` is configured for **manual sync**." \
            "After merging, trigger sync in ArgoCD and monitor the rollout." \
            "" \
            "Merging this PR updates ${PROD_VALUES}." > /tmp/pr-body.md

          gh pr create \
            --repo "$GITOPS_REPO_NAME" \
            --title "promote(prod): ${SVC} → ${IMAGE_TAG}" \
            --body-file /tmp/pr-body.md \
            --base main \
            --head "$BRANCH"

          echo "::notice title=PROD PR opened::Review, get approvals, and merge in ${GITOPS_REPO_NAME}. Then manually sync ArgoCD pharma-prod/${SVC}."
```

**Walkthrough:**

1. **`workflow_dispatch` with `choice` input** — When you click "Run workflow" in the GitHub Actions UI, a dropdown appears with all 7 backend services. You pick the service you want to promote.

2. **Read QA image tag** — The workflow reads the current `.image.tag` from `envs/qa/values-<service>.yaml`. This is the image that has been running and tested in QA.

3. **Validate PROD values file** — Checks that `envs/prod/values-<service>.yaml` exists. If not, the workflow fails with a clear error message.

4. **Create branch and open PR** — Creates a branch like `promote/prod/api-gateway/sha-a1b2c3d`, updates the PROD values file, and opens a PR with a checklist including CAB approval and runbook links.

5. **Manual sync required** — PROD ArgoCD apps use manual sync (not auto-sync like DEV). After the PR is merged, someone must explicitly trigger the sync in ArgoCD.

> **Why one consolidated workflow instead of 8 separate ones?**
> The promotion logic is identical for all services — read QA tag, update PROD, open PR. Having 8 separate workflows would mean duplicating this logic 8 times. The `choice` input with a dropdown keeps it simple: one workflow, pick the service, promote. For the frontend, we used a separate workflow because the frontend repo only has one service.

### Step 3: Commit All Workflows

Now commit all workflow files from sections 7.3–7.5 as a single commit — reusable workflows, per-service CI/CD and PR check workflows, and the promotion workflow:

```bash
cd ~/devops/zenpharma/backend
git add .github/workflows/
git commit -m "ci: add all CI/CD pipelines and promotion workflow for backend services"
git push
```

> **Tag `backend` repo: `module-7.5-backend-workflows`**
>
> ```bash
> git tag -a module-7.5-backend-workflows -m "Module 7.5: Reusable workflows, per-service CI/CD, PR checks, and PROD promotion"
> git push origin module-7.5-backend-workflows
> ```

---

## 7.6 Add Backend Values Files and ArgoCD Apps to GitOps Repo

Now we add the Kubernetes configuration for each backend service to the GitOps repository.

> **Tip:** There are 8 values files + 8 ArgoCD app manifests to create. You can either create them manually (shown below) or copy them from the course reference materials:
> ```bash
> cp /path/to/course-materials/gitops/envs/dev/values-*.yaml ~/devops/zenpharma/gitops/envs/dev/
> cp /path/to/course-materials/gitops/argocd/apps/dev/*-app.yaml ~/devops/zenpharma/gitops/argocd/apps/dev/
> ```
> If you copy, make sure the `repoURL` in each ArgoCD app manifest points to your actual gitops repo URL (e.g., `https://github.com/zenpharma/gitops.git`). Also replace the `<AWS_ACCOUNT_ID>` in values files with your actual account ID.

### Step 1: Create Values Files for Each Backend Service

Switch to the GitOps repo:

```bash
cd ~/devops/zenpharma/gitops
```

Here is the complete `envs/dev/values-api-gateway.yaml` as the detailed reference:

```yaml
replicaCount: 1
fullnameOverride: api-gateway
image:
  repository: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/api-gateway
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
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/pharma-dev-eks-role
  name: api-gateway
```

**Key differences from pharma-ui values (Module 5):**

| Setting | pharma-ui (frontend) | Backend services |
|---------|---------------------|-----------------|
| **Port** | 80 (Nginx) | 8080 (Java) or 3000 (Node.js) |
| **Probe paths** | `/` (returns index.html) | `/actuator/health` and `/actuator/health/readiness` (Spring Boot Actuator) |
| **Probe initial delay** | 10s (Nginx starts instantly) | 30-60s (JVM startup time) |
| **Ingress path** | `/` (catch-all for SPA) | `/api` (only api-gateway has ingress enabled) |
| **configmap** | `REACT_APP_*` env vars | `SPRING_PROFILES_ACTIVE`, service URLs for inter-service communication |
| **envFrom** | None | `db-credentials` (database password), `jwt-secret` (JWT signing key) |
| **Memory** | 64Mi request / 128Mi limit | 256Mi request / 512Mi limit (JVM needs more memory) |

> **Why `/api` ingress path?**
> The api-gateway is the only backend service with `ingress.enabled: true`. It receives all `/api/*` requests from the ALB. The other 7 backend services have `ingress.enabled: false` — they are only reachable internally via Kubernetes Service DNS names (e.g., `http://auth-service:8081`). This means external traffic can only enter through the api-gateway, which enforces authentication and authorization.

> **Why `envFrom` with `secretRef`?**
> Database credentials and JWT secrets should never be in values files or configmaps (which are stored in plaintext in etcd). Instead, they are stored in Kubernetes Secrets (created by the ExternalSecrets operator from AWS Secrets Manager, set up in Module 3). The `envFrom` block injects these secrets as environment variables into the pod.

> **Why Spring Boot Actuator probe paths?**
> Spring Boot Actuator provides built-in health check endpoints. `/actuator/health` returns the overall health status (database connectivity, disk space, etc.). `/actuator/health/readiness` specifically indicates whether the service is ready to accept traffic. Kubernetes uses these to decide when to restart a pod (liveness) or stop sending traffic to it (readiness).

### Step 2: Create Values Files for Remaining Services

Create values files for each backend service. The pattern is the same — adjust `fullnameOverride`, `image.repository`, `service.port`, and `configmap` values:

```bash
cd ~/devops/zenpharma/gitops
```

You need a values file for each service:

```
envs/dev/values-api-gateway.yaml
envs/dev/values-auth-service.yaml
envs/dev/values-catalog-service.yaml
envs/dev/values-inventory-service.yaml
envs/dev/values-manufacturing-service.yaml
envs/dev/values-supplier-service.yaml
envs/dev/values-qc-service.yaml
envs/dev/values-notification-service.yaml
```

Here is the **notification-service** values file to show the Node.js differences:

```yaml
replicaCount: 1
fullnameOverride: notification-service
image:
  repository: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/notification-service
  tag: sha-f3307fa
  pullPolicy: Always
service:
  type: ClusterIP
  port: 3000
  targetPort: 3000
ingress:
  enabled: false
  className: alb
  host: dev.pharma.internal
  path: /notifications
  pathType: Prefix
resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70
livenessProbe:
  path: /actuator/health
  port: 3000
  initialDelaySeconds: 20
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
readinessProbe:
  path: /actuator/health/readiness
  port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1
configmap:
  NODE_ENV: development
  PORT: "3000"
  LOG_LEVEL: debug
  SMTP_HOST: "smtp.example.com"
  SMTP_PORT: "587"
  EMAIL_FROM: "noreply-dev@example.com"
envFrom:
  - secretRef:
      name: db-credentials
serviceAccount:
  create: true
  annotations: {}
  name: notification-service
```

**Key differences for notification-service:**

| Setting | Java services | notification-service |
|---------|--------------|---------------------|
| Port | 8080 | 3000 |
| `configmap` | `SPRING_PROFILES_ACTIVE`, service URLs | `NODE_ENV`, `PORT`, SMTP settings |
| `envFrom` | `db-credentials` + `jwt-secret` | `db-credentials` only |
| Memory requests | 256Mi | 128Mi (Node.js is lighter than JVM) |
| Probe initial delay | 30-60s | 10-20s (Node.js starts faster) |

### Step 3: Create ArgoCD Application Manifests

For each backend service, create an ArgoCD Application manifest in `argocd/apps/dev/`:

Here is `argocd/apps/dev/api-gateway-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-gateway-dev
  namespace: argocd
  labels:
    env: dev
    app: api-gateway
    managed-by: terraform
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: pharma

  source:
    repoURL: https://github.com/zenpharma/gitops.git
    targetRevision: HEAD
    path: helm-charts
    helm:
      valueFiles:
        - ../envs/dev/values-api-gateway.yaml

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

**The pattern is the same as pharma-ui-app.yaml** (Module 5) — the only differences are:
- `metadata.name`: `<service>-dev` (e.g., `api-gateway-dev`, `auth-service-dev`)
- `metadata.labels.app`: the service name
- `helm.valueFiles`: points to the corresponding values file (e.g., `../envs/dev/values-api-gateway.yaml`)

Create an ArgoCD Application for each service:

```
argocd/apps/dev/api-gateway-app.yaml
argocd/apps/dev/auth-service-app.yaml
argocd/apps/dev/catalog-service-app.yaml
argocd/apps/dev/inventory-service-app.yaml
argocd/apps/dev/manufacturing-service-app.yaml
argocd/apps/dev/supplier-service-app.yaml
argocd/apps/dev/qc-service-app.yaml
argocd/apps/dev/notification-service-app.yaml
```

> **Why `automated: prune: true, selfHeal: true`?**
> - **prune**: If we remove a resource from the Helm chart, ArgoCD deletes it from the cluster (instead of leaving orphaned resources)
> - **selfHeal**: If someone manually edits a resource in the cluster (e.g., `kubectl edit`), ArgoCD reverts it to match the Git state. This enforces "Git is the single source of truth."

### Step 4: Commit and Push

```bash
cd ~/devops/zenpharma/gitops
git add envs/dev/values-*.yaml argocd/apps/dev/*-app.yaml
git commit -m "feat: add backend service values files and ArgoCD applications for dev"
git push
```

> **Tag `gitops` repo: `module-7.6-backend-values`**
>
> ```bash
> git tag -a module-7.6-backend-values -m "Module 7.6: Backend values files and ArgoCD applications for dev environment"
> git push origin module-7.6-backend-values
> ```

---

## 7.7 Add Secrets, Variables, and SonarCloud to Backend Repo

The backend CI/CD workflows need the same secrets and variables as the frontend (Module 4). We also need to add the backend repo as a new project in SonarCloud.

### Step 1: Add Backend Project to SonarCloud

In Module 4 you added the **frontend** repo to SonarCloud. Now add the **backend** repo:

1. Go to https://sonarcloud.io and log in with your GitHub account
2. Click **Analyze new project** → select your `backend` repository → click **Set Up**
3. SonarCloud creates the project

**Disable Automatic Analysis (important):**

1. Go to your **backend project page** on SonarCloud
2. Click **Administration** (bottom of the left sidebar)
3. Click **Analysis Method**
4. Find the **Automatic Analysis** toggle and turn it **OFF**

> **Why?** Just like the frontend, you cannot run both CI-based analysis and Automatic Analysis at the same time. The pipeline will fail with: `You are running CI analysis while Automatic Analysis is enabled`.

**Find the Project Key:**

1. Go to your backend project page on SonarCloud
2. Click **Project Information** (bottom of the left sidebar)
3. Copy the **Project Key** (e.g., `zenpharma_backend`)
4. The **Organization Key** is the same one you used for frontend (e.g., `zenpharma`)

> **Note:** You can reuse the same `SONAR_TOKEN` from Module 4 — it is a user-level token, not project-specific. One token works for all projects in your organization.

### Step 2: Add Repository Secrets

Go to your **backend** repository on GitHub:

1. Click **Settings** > **Secrets and variables** > **Actions**
2. Add the following **Repository secrets**:

| Secret | Value | Purpose |
|--------|-------|---------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID | Used by OIDC role assumption for ECR access |
| `GITOPS_TOKEN` | The fine-grained PAT created in Module 4 | Push commits and open PRs in gitops repo |
| `SONAR_TOKEN` | Same token from Module 4 (or generate a new one) | Authenticates SonarCloud SAST and code-quality scans |

### Step 3: Add Repository Variables

In the same **Secrets and variables** > **Actions** page:

1. Click the **Variables** tab
2. Click **New repository variable**
3. Add:

| Variable | Value | How to Find |
|----------|-------|-------------|
| `GITOPS_REPO` | `<your-github-org>/gitops` | Your gitops repo in `owner/repo` format |
| `SONAR_ORG` | Your SonarCloud organization key | Same as frontend — SonarCloud → **Project Information** |
| `SONAR_PROJECT_KEY_BACKEND` | Project key for backend | SonarCloud → backend project → **Project Information** |

### Step 4: Protect Main Branch

1. Go to **Settings** > **Branches**
2. Click **Add branch protection rule** (or **Add classic branch protection rule**)
3. Branch name pattern: `main`
4. Enable:
   - **Require a pull request before merging**
     - Set **Required number of approvals** to **0** (same as frontend — solo-friendly, PR still required)
5. **Do NOT enable** "Require status checks to pass before merging"
6. Click **Create**

### Step 5: Create Develop Branch

```bash
cd ~/devops/zenpharma/backend
git checkout -b develop
git push -u origin develop
```

All CI/CD workflows trigger on pushes to `develop` — this branch is where active development happens. Feature branches merge into `develop` via PRs.

---

## 7.8 Build and Deploy All Backend Services

### Option A: Automated (using course scripts)

The course scripts can trigger all 8 pipelines and deploy all services automatically:

```bash
cd ~/devops/zenpharma
```

**Trigger pipelines:**

```bash
python3 04_run_pipeline.py
```

Select **Option B** (Backend — all services) or **Option A** (All services — frontend + backend). This pushes a trigger commit to the `develop` branch, which activates all 8 per-service CI workflows via the `paths` filter.

**Deploy services:**

```bash
python3 05_deploy_services.py
```

Select **Option A** (Deploy all) to `kubectl apply` all ArgoCD Application manifests.

### Option B: Manual

**Trigger pipelines by pushing to develop:**

```bash
cd ~/devops/zenpharma/backend
git checkout develop
# Make a small change (e.g., update a comment) in each service directory
# Or use workflow_dispatch from the GitHub Actions UI
git push
```

Since all 8 service directories have changes, all 8 CI pipelines will trigger simultaneously.

**Deploy by applying ArgoCD manifests:**

```bash
cd ~/devops/zenpharma/gitops

# Apply each ArgoCD Application manifest
for app in argocd/apps/dev/*-app.yaml; do
  kubectl apply -f "$app"
  echo "Applied: $app"
done
```

**Expected output:**
```
application.argoproj.io/api-gateway-dev created
application.argoproj.io/auth-service-dev created
application.argoproj.io/catalog-service-dev created
application.argoproj.io/inventory-service-dev created
application.argoproj.io/manufacturing-service-dev created
application.argoproj.io/notification-service-dev created
application.argoproj.io/qc-service-dev created
application.argoproj.io/supplier-service-dev created
```

> **Why does this take 10-15 minutes?**
> Each Java service pipeline runs Maven compile + tests + SonarCloud + Docker build + Trivy + ECR push + Cosign signing. While all 8 run in parallel (each triggered by its own `paths` filter), each individual pipeline takes ~10-15 minutes. The notification-service pipeline is slightly faster since Node.js builds are quicker than Maven.

### Monitor Build Progress

Watch the GitHub Actions tab of your backend repository. You should see 8 workflows running:

```
CI/CD — api-gateway           ● Running
CI/CD — auth-service          ● Running
CI/CD — drug-catalog          ● Running
CI/CD — inventory-service     ● Running
CI/CD — manufacturing-service ● Running
CI/CD — supplier-service      ● Running
CI/CD — qc-service            ● Running
CI/CD — notification-service  ● Running
```

Each workflow will progress through: Build & Security Gates → Deploy DEV.

---

## 7.9 End-to-End Validation

### Step 1: Run Verification Script

```bash
cd ~/devops/zenpharma
python3 06_verify_deployment.py
```

This script checks:
- All ArgoCD applications are `Synced` and `Healthy`
- All pods are `Running` with `1/1` containers ready
- Health check endpoints respond

### Step 2: Check ArgoCD UI

1. Open the ArgoCD dashboard (URL from Module 3)
2. You should see **9 applications** in the DEV environment:
   - `pharma-ui-dev` (from Module 5/6)
   - `api-gateway-dev`
   - `auth-service-dev`
   - `catalog-service-dev`
   - `inventory-service-dev`
   - `manufacturing-service-dev`
   - `notification-service-dev`
   - `qc-service-dev`
   - `supplier-service-dev`

All should show **Synced** (green) and **Healthy** (green heart icon).

### Step 3: Verify Pods are Running

```bash
kubectl get pods -n dev
```

**Expected output:**
```
NAME                                      READY   STATUS    RESTARTS   AGE
api-gateway-xxxxxxxxx-xxxxx               1/1     Running   0          5m
auth-service-xxxxxxxxx-xxxxx              1/1     Running   0          5m
catalog-service-xxxxxxxxx-xxxxx           1/1     Running   0          5m
inventory-service-xxxxxxxxx-xxxxx         1/1     Running   0          5m
manufacturing-service-xxxxxxxxx-xxxxx     1/1     Running   0          5m
notification-service-xxxxxxxxx-xxxxx      1/1     Running   0          5m
pharma-ui-xxxxxxxxx-xxxxx                 1/1     Running   0          30m
qc-service-xxxxxxxxx-xxxxx               1/1     Running   0          5m
supplier-service-xxxxxxxxx-xxxxx          1/1     Running   0          5m
```

### Step 4: Test API Endpoints

Check the api-gateway health endpoint through the ALB:

```bash
# Get the ALB URL
ALB_URL=$(kubectl get ingress -n dev -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')
echo "ALB URL: http://${ALB_URL}"

# Test api-gateway health
curl -s "http://${ALB_URL}/api/actuator/health" | jq .
```

**Expected output:**
```json
{
  "status": "UP"
}
```

### Step 5: Access the Full Application

Open the ALB URL in your browser. The pharma-ui frontend should now be fully functional:

1. Navigate to `http://<ALB_URL>/`
2. You should see the login page
3. Log in with the default credentials: `admin` / `changeme`
4. Navigate through the application — Drug Catalog, Inventory, Suppliers, etc.

> **Why does it work now?**
> In Module 6, the frontend loaded but API calls failed. Now with all 8 backend services running:
> - The ALB routes `/` to pharma-ui (frontend)
> - The ALB routes `/api/*` to api-gateway
> - api-gateway routes to the appropriate backend service
> - Backend services connect to RDS PostgreSQL for data

### Step 6: Debugging Common Issues

If pods are not starting or services are unhealthy, use these commands:

```bash
# Check pod events (CrashLoopBackOff, ImagePullBackOff, etc.)
kubectl describe pod <pod-name> -n dev

# Check application logs
kubectl logs <pod-name> -n dev

# Check if secrets exist (required by envFrom)
kubectl get secrets -n dev

# Check ArgoCD sync status
kubectl get applications -n argocd

# Check if ECR images exist
aws ecr describe-images --repository-name api-gateway --region us-east-1 --query 'imageDetails[*].imageTags' --output table
```

**Common issues and solutions:**

| Issue | Symptom | Solution |
|-------|---------|----------|
| `ImagePullBackOff` | Pod cannot pull image from ECR | Check that the ServiceAccount has the correct IAM role annotation and that the ECR repository exists |
| `CrashLoopBackOff` | Pod starts then crashes | Check logs with `kubectl logs`. Usually a missing environment variable or database connection issue |
| `0/1 Running` | Readiness probe failing | The service is starting but not ready. Check `initialDelaySeconds` — Java services need 30-60s to start |
| ArgoCD `OutOfSync` | App shows yellow status | Click "Sync" in ArgoCD UI or run `argocd app sync <app-name>` |
| `db-credentials` missing | Pod fails to start | Ensure ExternalSecrets operator is running and the SecretStore is configured (Module 3) |

### Step 7: Tag All Repos

---

## Module 7 Summary

| What We Built | Details |
|--------------|---------|
| **Dockerfiles** | 7 identical Java Dockerfiles (eclipse-temurin:17-jre, non-root user) + 1 Node.js multi-stage Dockerfile |
| **Reusable Workflows** | `_java-build.yml` (7-stage pipeline), `_node-build.yml` (8-stage pipeline), `_java-pr-check.yml`, `_node-pr-check.yml` |
| **Per-Service CI/CD** | 16 workflow files (8 full pipelines + 8 PR checks) with monorepo path filtering |
| **PROD Promotion** | Single consolidated `promote-prod.yml` with service dropdown for all backend services |
| **GitOps Values** | 8 values files with Spring Boot Actuator probes, inter-service URLs, secret references |
| **ArgoCD Apps** | 8 Application manifests with automated sync, self-heal, and pruning |
| **Security Pipeline** | SonarCloud + OWASP Dependency Check + Trivy image scan + Cosign keyless signing |
| **Full Stack Deploy** | 9 services running in DEV (1 frontend + 8 backend) with end-to-end connectivity |

| Tag | Repos |
|-----|-------|
| `module-7.2-dockerfiles` | backend |
| `module-7.5-backend-workflows` | backend |
| `module-7.6-backend-values` | gitops |

> **Next:** [Module 8 — Environment Promotion](MODULE-8-ENVIRONMENT-PROMOTION.md)
