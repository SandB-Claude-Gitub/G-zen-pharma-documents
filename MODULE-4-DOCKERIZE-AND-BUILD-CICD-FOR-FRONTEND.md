# Module 4 — Dockerize & Build CI/CD for Frontend

> Dockerize the pharma-ui frontend, set up branch protection and a branching strategy, configure secrets, and build GitHub Actions CI/CD pipelines that lint, test, scan, build, and deploy through DEV and QA environments.
> Estimated time: 2-3 hours.

---

## 4.1 Create Dockerfile for Pharma-UI

We will create a multi-stage Dockerfile that builds the React application and serves it with Nginx.

### Step 1: Create the Dockerfile

In your `frontend` repo, create the `Dockerfile`:

```bash
cd ~/devops/zenpharma/frontend
```

Create `Dockerfile` with the following content:

```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY public ./public
COPY src ./src
COPY .env.production ./
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:1.25-alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
RUN addgroup -S pharma && adduser -S pharma -G pharma
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Let's walk through every line:

**Stage 1 — Build:**

| Line | Explanation |
|------|-------------|
| `FROM node:22-alpine AS builder` | Uses the Alpine variant of Node 22 as the build environment. Alpine images are ~5x smaller than full Debian-based images (~180 MB vs ~900 MB). The `AS builder` label lets us reference this stage later. |
| `WORKDIR /app` | Sets the working directory inside the container. All subsequent commands run from `/app`. |
| `COPY package*.json ./` | Copies both `package.json` and `package-lock.json`. We copy these **first** so Docker can cache the dependency install layer separately from source code changes. |
| `RUN npm ci` | Installs dependencies from the lockfile. `npm ci` (clean install) is preferred over `npm install` in CI/Docker because it is **deterministic** — it installs exactly what is in `package-lock.json`, never modifying it. It also removes `node_modules` first, ensuring a clean state. |
| `COPY public ./public` | Copies static assets (favicon, manifest, index.html template). |
| `COPY src ./src` | Copies application source code (React components, styles, tests). |
| `COPY .env.production ./` | Copies the production environment file. Our `.env.production` contains `REACT_APP_API_URL=/api`, which tells React to use relative `/api` paths (the Nginx proxy handles routing these to the backend). |
| `RUN npm run build` | Runs `react-scripts build`, which creates an optimized production bundle in the `build/` directory with minified JS, CSS, and hashed filenames for cache-busting. |

**Stage 2 — Serve:**

| Line | Explanation |
|------|-------------|
| `FROM nginx:1.25-alpine` | Starts a fresh stage with Nginx on Alpine. The final image contains **only Nginx and the build output** — no Node.js, no `node_modules`, no source code. This drastically reduces the image size (from ~500 MB to ~25 MB) and the attack surface. |
| `COPY --from=builder /app/build /usr/share/nginx/html` | Copies the production build artifacts from Stage 1 into Nginx's default web root. This is the magic of multi-stage builds — we get the output without carrying the build tools. |
| `COPY nginx.conf /etc/nginx/conf.d/default.conf` | Replaces the default Nginx config with our custom configuration (created in the next step). |
| `RUN addgroup -S pharma && adduser -S pharma -G pharma` | Creates a non-root system user and group named `pharma`. This follows the security best practice of not running processes as root, even inside a container. |
| `EXPOSE 80` | Documents that the container listens on port 80 (Nginx's default HTTP port). This is informational for other developers and tools; it does not actually publish the port. |
| `CMD ["nginx", "-g", "daemon off;"]` | Starts Nginx in the foreground. By default Nginx daemonizes itself, but containers need the main process to stay in the foreground — if the process backgrounds itself, Docker thinks the container exited. |

> **Why multi-stage builds?**
> - The build stage needs Node.js, npm, and all dev dependencies (~500 MB)
> - The serve stage only needs Nginx and the static HTML/JS/CSS output (~25 MB)
> - Without multi-stage, the final image would contain all of Node.js and `node_modules` — wasted space, slower pulls, and a larger attack surface

### Step 2: Create the Nginx Configuration

Create `nginx.conf` in the same directory:

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    

    location /api/ {
        proxy_pass http://api-gateway:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    gzip on;
    gzip_types text/plain application/javascript text/css application/json;
    gzip_min_length 1000;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
}
```

Let's break down each section:

**Basic server block:**

| Directive | Explanation |
|-----------|-------------|
| `listen 80` | Listen on port 80 (HTTP). In Kubernetes, TLS termination happens at the ALB/Ingress level, so the container only needs HTTP. |
| `server_name _` | Matches any hostname. The underscore is a catch-all — we don't hardcode a domain because the same image runs in DEV, QA, and PROD with different domain names. |
| `root /usr/share/nginx/html` | Serves files from the directory where we copied the React build output. |
| `index index.html` | Default file to serve when a directory is requested. |

**SPA routing:**

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

> **Why `try_files` with `/index.html` fallback?**
> React Router handles routing on the client side. When a user navigates to `/inventory/items` and refreshes the browser, Nginx receives a request for `/inventory/items` — but that file does not exist on disk. Without `try_files`, Nginx would return a 404. With this directive, Nginx first checks if the exact file exists (`$uri`), then checks for a directory (`$uri/`), and finally falls back to serving `index.html`, which loads the React app and lets React Router handle the route.

**API proxy:**

```nginx
location /api/ {
    proxy_pass http://api-gateway:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

This block forwards all `/api/` requests to the backend API Gateway service. In Kubernetes, `api-gateway` resolves to the API Gateway service via DNS. The `proxy_set_header` directives preserve the original client's hostname and IP address so the backend knows who actually made the request (not just that it came from Nginx).

**Gzip compression:**

```nginx
gzip on;
gzip_types text/plain application/javascript text/css application/json;
gzip_min_length 1000;
```

Enables on-the-fly compression for text-based assets. JavaScript and CSS files compress very well (typically 60-80% size reduction). The `gzip_min_length 1000` prevents compressing tiny files where the overhead of compression outweighs the benefit.

**Security headers:**

```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
```

| Header | Purpose |
|--------|---------|
| `X-Frame-Options: SAMEORIGIN` | Prevents the page from being embedded in iframes on other domains (clickjacking protection). |
| `X-Content-Type-Options: nosniff` | Prevents browsers from MIME-type sniffing — forces them to respect the declared `Content-Type`. Stops attackers from tricking the browser into executing a malicious file as JavaScript. |
| `X-XSS-Protection: 1; mode=block` | Enables the browser's built-in XSS filter. If a reflected XSS attack is detected, the browser blocks the page rather than trying to sanitize it. |

### Step 3: Build and Test Locally

```bash
docker build -t pharma-ui .
```

**Expected output (last few lines):**
```
 => [builder 7/7] RUN npm run build
 => [stage-1 1/4] FROM nginx:1.25-alpine
 => [stage-1 2/4] COPY --from=builder /app/build /usr/share/nginx/html
 => [stage-1 3/4] COPY nginx.conf /etc/nginx/conf.d/default.conf
 => [stage-1 4/4] RUN addgroup -S pharma && adduser -S pharma -G pharma
 => exporting to image
Successfully tagged pharma-ui:latest
```

Run the container:

```bash
docker run -p 80:80 pharma-ui
```

Open http://localhost in your browser. You should see the ZenPharma login page. The API calls will fail (no backend running), but the frontend should load and render correctly.

Press `Ctrl+C` to stop the container.

### Step 4: Commit and Push

```bash
cd ~/devops/zenpharma/frontend
git add Dockerfile nginx.conf .env.production
git commit -m "feat: add Dockerfile and nginx config for pharma-ui"
git push
```

> **Tag `frontend` repo: `module-4.1-dockerfile`**
>
> ```bash
> git tag -a module-4.1-dockerfile -m "Module 4.1: Multi-stage Dockerfile and Nginx config for pharma-ui"
> git push origin module-4.1-dockerfile
> ```

---

## 4.2 Protect Main Branch and Create Develop Branch

Before adding CI/CD workflows, we need a branching strategy. We will protect the `main` branch to prevent direct pushes and create a `develop` branch for active development.

### Step 1: Enable Branch Protection on `main`

1. Go to your frontend repository on GitHub
2. Click **Settings** (top menu)
3. In the left sidebar, click **Branches** (under "Code and automation")
4. Click **Add branch protection rule** (or **Add classic branch protection rule**)
5. Under **Branch name pattern**, enter: `main`
6. Enable the following settings:
   - **Require a pull request before merging**
     - Set **Required number of approvals** to **0**
7. **Do NOT enable** "Require status checks to pass before merging" — leave it unchecked
8. Click **Create** (or **Save changes**)

> **Why 0 approvals and no status checks?** Same reasoning as Module 2:
> - **0 approvals** — You're working solo. A PR is still required (no direct pushes), but you can merge it yourself. In a real team, set this to 1+.
> - **No required status checks** — The CI pipeline runs on every PR and you can review the results in the Actions tab. But configuring required checks by name is error-prone (exact name matching) and blocks the merge button if misconfigured. The CI still runs — you just aren't forced to wait for it.

> **Why branch protection at all?**
> - Prevents accidental pushes directly to `main` — all changes must go through a pull request
> - Creates an audit trail of who merged what
> - CI runs automatically on PRs so you can review results before merging

### Step 2: Create the `develop` Branch

```bash
cd ~/devops/zenpharma/frontend
git checkout -b develop
git push -u origin develop
```

### Step 3: Understand the Branching Strategy

Here is how branches flow through the pipeline:

```
developer creates feature branch from develop
    ↓
developer opens PR → develop
    ↓
CI runs lint, test, security (PR checks)
    ↓
reviewer approves, PR merges to develop
    ↓
CI runs full pipeline: lint → test → security → build → docker → push → deploy DEV
    ↓
Developer validates DEV deployment
    ↓
Manual trigger: promote-qa workflow → QA promotion PR
    ↓
QA team reviews and merges → ArgoCD deploys to QA
    ↓
Manual trigger: promote-prod workflow → PROD promotion PR
    ↓
Change board approves, PR merges → ArgoCD manual sync → PROD
```

**Branch types:**

| Branch | Purpose | CI Behavior |
|--------|---------|-------------|
| `develop` | Active development. All feature branches merge here. | Full pipeline: build Docker image, push to ECR, deploy to DEV |
| `main` | Stable releases. Protected — requires PR and approvals. | No direct CI triggers (promotion to PROD is manual via workflow_dispatch) |
| `feature/*` | Individual features or bug fixes. Created from `develop`. | PR checks only: lint, test, security (no Docker build) |
| `release/*` | Optional release candidates. Created from `develop`. | Full pipeline (same as `develop` — builds and deploys) |

---

## 4.3 Add GitOps Token and Secrets to GitHub

The CI pipeline needs two kinds of access:
1. **Push Docker images to ECR** — handled via OIDC federation (no static keys needed; the IAM role was created in Module 1.7)
2. **Update image tags in the gitops repo** — needs a GitHub PAT with write access

### Step 1: Create a GitHub Fine-Grained PAT

1. Go to https://github.com/settings/tokens (or: profile icon → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**)
2. Click **Generate new token**
3. Set the following:
   - **Token name:** `zenpharma-gitops-write`
   - **Expiration:** 90 days (or your preference)
   - **Resource owner:** Select your `zenpharma` organization
   - **Repository access:** Select **Only select repositories** → choose your `gitops` repository
   - **Permissions:**
     - **Contents:** Read and write (to push commits and create branches)
     - **Pull requests:** Read and write (to open PRs for QA/PROD promotion)
4. Click **Generate token**
5. Copy the token — you will not see it again

> **Why a fine-grained PAT instead of a classic PAT?**
> - Scoped to a single repository (gitops only) — least privilege
> - Specific permissions (contents + pull requests only) — a classic PAT grants access to all repositories you own
> - Has an expiration date — forces periodic rotation

### Step 2: Add Secrets to the Frontend Repository

1. Go to your frontend repository on GitHub
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Under **Repository secrets**, click **New repository secret** for each:

| Secret Name | Value | Purpose |
|-------------|-------|---------|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account number (e.g., `123456789012`) | Used to construct the IAM role ARN and ECR registry URL |
| `GITOPS_TOKEN` | The PAT you created above | Write access to the gitops repo for updating image tags and opening PRs |
| `SONAR_TOKEN` | Your SonarCloud token (see setup instructions below) | Authenticates SonarCloud SAST and code-quality scans |

4. Under **Repository variables** (same page, switch to the **Variables** tab), click **New repository variable**:

| Variable Name | Value | Purpose |
|---------------|-------|---------|
| `GITOPS_REPO` | `zenpharma/gitops` (your org/repo) | Tells the CI workflow which repo to update with new image tags |
| `SONAR_ORG` | Your SonarCloud organization key | Identifies your SonarCloud organization |
| `SONAR_PROJECT_KEY_FRONTEND` | Project key for pharma-ui in SonarCloud | Identifies this project within your SonarCloud organization |

**SonarCloud setup:**

1. Go to https://sonarcloud.io and click **Log in** → sign in with your **GitHub account**
2. Click **Import an organization** → select your GitHub organization (e.g., `zenpharma`)
3. Choose the **Free plan** → click **Create Organization**
4. Click **Analyze new project** → select your `frontend` repository → click **Set Up**
5. SonarCloud creates the project

**Disable Automatic Analysis (important):**

SonarCloud enables **Automatic Analysis** by default on new projects. This conflicts with our CI-based analysis from GitHub Actions — you cannot run both at the same time. If you skip this step, the pipeline will fail with: `You are running CI analysis while Automatic Analysis is enabled`.

1. Go to your **project page** on SonarCloud (not the organization page)
2. Click **Administration** (bottom of the left sidebar)
3. Click **Analysis Method**
4. Find the **Automatic Analysis** toggle and turn it **OFF**
5. The page should now show CI-based analysis as the active method

> **Note:** The toggle is on the **project** settings page, not the organization settings page. If you go to Organization Settings → Analysis, you'll only see a description — the actual on/off toggle is per-project under Administration → Analysis Method.

**How to find the Organization Key and Project Key:**

1. Go to your project page on SonarCloud
2. Click **Project Information** (bottom of the left sidebar)
3. You will see both values:
   - **Organization Key** — typically your GitHub org name in lowercase (e.g., `zenpharma`)
   - **Project Key** — typically in the format `<org>_<repo>` (e.g., `zenpharma_frontend`)

You can also find them in the URL: `https://sonarcloud.io/project/overview?id=<project-key>`

**Generate a SonarCloud token:**

1. Click your **profile icon** (top right) → **My Account**
2. Go to the **Security** tab
3. Enter a token name (e.g., `github-actions`) → click **Generate**
4. Copy the token and save it as the `SONAR_TOKEN` secret above

> **Why SonarCloud?**
> SonarCloud provides SAST (static application security testing), code-quality analysis, code-smell detection, and coverage tracking in a single managed service. It integrates natively with GitHub pull requests, posting inline comments on new issues. Unlike self-hosted tools, SonarCloud requires zero infrastructure — just a token and a GitHub Action.

> **Why no AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY?**
> The CI pipeline authenticates to AWS using **OIDC federation** — not static credentials. When the workflow runs, GitHub mints a short-lived OIDC token. AWS validates this token against the trust policy on `pharma-dev-github-actions-role` (created by the IAM module in Module 1.7 with `github-actions-oidc.tf`). The role grants permissions to push to ECR. This approach is more secure because there are no long-lived secrets to rotate or leak.

> **No tag needed** — this step is GitHub configuration only, no code changes.

---

## 4.4 Create GitHub Workflows for Pharma-UI

This is the core of Module 4. We will create a single CI/CD workflow file that handles both PR checks and the full deployment pipeline.

### Understanding the Workflow Design

**Two scenarios, one workflow file:**

| Trigger | What Runs | Why |
|---------|-----------|-----|
| `pull_request` to `main` or `develop` | Lint, Test, SonarCloud, Build | Full quality checks including build verification. No Docker build, no ECR push, no deployment. PRs should never produce deployable artifacts. |
| `push` to `develop` or `release/*` | Lint, Test, SonarCloud, Build, Docker Build, Trivy Scan, ECR Push, Deploy to DEV | A merge to `develop` means the code is reviewed and approved. Now we build, scan, push, and deploy. |

We use `if:` conditions on the Docker/deploy jobs to control which jobs run for each trigger type.

**Pipeline flow:**

```
Developer pushes to develop
    |
    v
  Lint
    |
    v
  Test
    |
    +---> SonarCloud ---+
    |                   |
    +-------------------+
                        |
                        v
                      Build
                        |
                        v
              Docker Build & Push
          (only on push, not on PRs)
                |               |
                v               v
           Trivy Scan     Push to ECR
                                |
                                v
                          Deploy DEV
                       (direct push to
                        gitops repo)
                                |
                                v
                    ArgoCD syncs DEV deployment
```

Test runs after Lint, and SonarCloud runs after Test (it needs the coverage report). This sequential flow ensures that the coverage data is available for SonarCloud analysis.

### Step 1: Create the Workflow File

Make sure you are on the `develop` branch:

```bash
cd ~/devops/zenpharma/frontend
git checkout develop
mkdir -p .github/workflows
```

Create `.github/workflows/ci-pharma-ui.yml`:

```yaml
name: CI/CD — pharma-ui

# Triggers on push to develop / release branches only.
# PRs → lint, test, security run but no Docker build.
# PROD promotion → promote-prod-pharma-ui.yml (workflow_dispatch)

on:
  push:
    branches: [develop, 'release/**']
  pull_request:
    branches: [main, develop]
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
  pull-requests: read

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: pharma-ui
  GITOPS_REPO: ${{ vars.GITOPS_REPO }}

jobs:
  # ── 1. Lint ───────────────────────────────────────────────────────────────
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: ESLint
        run: ./node_modules/.bin/eslint src/ --ext .js,.jsx

  # ── 2. Unit Tests with Coverage ──────────────────────────────────────────
  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm test -- --ci
        env:
          CI: true

      - name: Upload coverage report
        uses: actions/upload-artifact@v6
        with:
          name: coverage-report
          path: coverage/
          retention-days: 3

  # ── 3. Code Quality & Security (SonarCloud + npm audit) ──────────────────
  sonar:
    name: Code Quality & Security
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0

      - name: Download coverage report
        uses: actions/download-artifact@v7
        with:
          name: coverage-report
          path: coverage/

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@v3
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        with:
          args: >
            -Dsonar.organization=${{ vars.SONAR_ORG }}
            -Dsonar.projectKey=${{ vars.SONAR_PROJECT_KEY_FRONTEND }}
            -Dsonar.sources=src
            -Dsonar.tests=src
            -Dsonar.test.inclusions=**/*.test.js,**/*.test.jsx
            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info

      - name: npm audit (HIGH/CRITICAL — fail)
        run: npm audit --audit-level=high
        continue-on-error: true

  # ── 4. Build React App ───────────────────────────────────────────────────
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [test, sonar]
    steps:
      - uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build production bundle
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v6
        with:
          name: build-artifact
          path: build/

  # ── 5. Docker Build, Trivy Scan & Push to ECR ────────────────────────────
  docker-build-push:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
    outputs:
      image_tag: ${{ steps.tag.outputs.image_tag }}
    steps:
      - uses: actions/checkout@v5

      - name: Set short image tag
        id: tag
        run: echo "image_tag=sha-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/pharma-dev-github-actions-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.image_tag }} \
            -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest .

      - name: Trivy — scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.image_tag }}
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: '0'

      - name: Push image to ECR
        run: |
          docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.image_tag }}
          docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:latest

  # ── 6. Deploy — DEV (direct push to gitops) ──────────────────────────────
  deploy-dev:
    name: Deploy — DEV
    needs: docker-build-push
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

      - name: Install yq
        run: |
          wget -qO /usr/local/bin/yq \
            https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          chmod +x /usr/local/bin/yq

      - name: Update image tag — DEV
        env:
          IMAGE_TAG: ${{ needs.docker-build-push.outputs.image_tag }}
        run: |
          yq e ".image.tag = \"${IMAGE_TAG}\"" -i \
            "_gitops/envs/dev/values-pharma-ui.yaml"
          cd _gitops
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          git diff --staged --quiet && echo "Already up to date" && exit 0
          git commit -m "ci(dev): update pharma-ui → ${IMAGE_TAG}"
          git push
```

### Step 2: Deep Dive — Job-by-Job Explanation

#### Job 1: `lint`

```yaml
lint:
  name: Lint
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v5
    - name: Setup Node.js
      uses: actions/setup-node@v5
      with:
        node-version: '22'
        cache: 'npm'
    - name: Install dependencies
      run: npm ci
    - name: ESLint
      run: ./node_modules/.bin/eslint src/ --ext .js,.jsx
```

> **Why lint first?**
> Lint is the fastest check (~10-15 seconds). If the code has formatting or style violations, there is no point running slower tests or security scans. Lint acts as a gate — everything else depends on it passing.

> **Why `cache: 'npm'`?**
> The `setup-node` action caches the npm cache directory between workflow runs. On subsequent runs, `npm ci` can pull packages from the cache instead of downloading them from the registry. This typically saves 20-40 seconds per job.

#### Job 2: `test` (needs: lint)

```yaml
test:
  name: Unit Tests
  needs: lint
  ...
    - name: Run tests with coverage
      run: npm test -- --ci
      env:
        CI: true
    - name: Upload coverage report
      uses: actions/upload-artifact@v6
      with:
        name: coverage-report
        path: coverage/
        retention-days: 3
```

> **Why `-- --ci`?**
> The `--` passes subsequent flags through `npm test` to the underlying test runner (Jest). The `--ci` flag tells Jest to run in CI mode: no interactive watch mode, fail if no tests are found, and use a single-run execution. Without `--ci`, Jest would start in watch mode and hang the pipeline forever.

> **Why `retention-days: 3`?**
> Coverage reports are only useful for the current PR review cycle. Keeping them for 3 days saves storage costs — GitHub Actions artifact storage is metered on paid plans.

> **Why does test depend on lint?**
> Fail fast. If the code has lint errors, running a 60-second test suite wastes CI minutes. Fix the lint errors first, then re-run.

#### Job 3: `sonar` (needs: test)

```yaml
sonar:
  name: Code Quality & Security
  needs: test
  ...
```

This job runs two security and quality tools:

| Tool | Type | What It Catches |
|------|------|-----------------|
| **SonarCloud** | SAST + code quality | Security vulnerabilities, code smells, bugs, duplications, and coverage analysis. Posts inline PR comments on new issues. |
| **npm audit** | Dependency scanning | Known CVEs in `node_modules` dependencies. Checks against the GitHub Advisory Database. |

> **Why does sonar need the coverage report?**
> The test job uploads a coverage report as an artifact. The sonar job downloads it and passes the `lcov.info` file to SonarCloud via `-Dsonar.javascript.lcov.reportPaths=coverage/lcov.info`. This lets SonarCloud display coverage metrics alongside code-quality findings in a single dashboard.

> **Why `fetch-depth: 0` on the checkout?**
> SonarCloud needs the full git history to accurately detect new code on pull requests and perform blame-based analysis. A shallow clone (the default `fetch-depth: 1`) only has the latest commit, which limits SonarCloud's ability to distinguish new issues from existing ones.

> **Why does sonar run after test?**
> The sonar job depends on `test` (via `needs: test`) because it needs the coverage report artifact that the test job uploads. The dependency graph is:
> ```
> lint → test → sonar → build
> ```

#### Job 4: `build` (needs: [test, sonar])

```yaml
build:
  name: Build
  needs: [test, sonar]
  ...
    - name: Build production bundle
      run: npm run build
    - name: Upload build artifact
      uses: actions/upload-artifact@v6
      with:
        name: build-artifact
        path: build/
```

> **Why a separate build job instead of building inside Docker?**
> The Docker build (Job 5) also runs `npm run build` inside the container. This separate build job validates that the React build succeeds **before** incurring Docker overhead. If the React build fails (e.g., TypeScript errors, missing imports), you get a clear error from this job without waiting for Docker to pull images and set up layers. The uploaded artifact is also useful for debugging build issues.

#### Job 5: `docker-build-push` (needs: build, only on push/dispatch)

This is where the `if:` condition gates execution:

```yaml
docker-build-push:
  needs: build
  if: github.event_name == 'push' || github.event_name == 'workflow_dispatch'
```

> **Why skip Docker build on PRs?**
> PRs are for validating code quality (lint, test, security). Building and pushing a Docker image for every PR commit would:
> - Waste CI minutes (~2-3 minutes per build)
> - Push unused images to ECR (each PR commit would create an image that is never deployed)
> - Increase ECR storage costs
> Images should only be built when code is merged (via `push` event) or manually triggered.

**Image tag strategy:**

```yaml
- name: Set short image tag
  id: tag
  run: echo "image_tag=sha-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
```

The image tag is `sha-` followed by the first 7 characters of the git commit SHA (e.g., `sha-a1b2c3d`).

> **Why SHA-based tags?**
> - **Immutable** — every commit gets a unique tag. You can never accidentally overwrite an image.
> - **Traceable** — given an image tag, you can find the exact commit: `git log sha-a1b2c3d`
> - **Why NOT just `latest`?** — `latest` is mutable. If two pipelines run simultaneously, they overwrite each other's `latest`. In production, you cannot tell which commit is running.
> - **Why push BOTH `sha-xxxxx` AND `latest`?** — `sha-xxxxx` for traceability and immutability; `latest` for convenience (e.g., quickly pulling the most recent image for local testing).

**OIDC authentication to AWS:**

```yaml
- name: Configure AWS credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v5
  with:
    role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/pharma-dev-github-actions-role
    aws-region: ${{ env.AWS_REGION }}
```

> **Why OIDC instead of static AWS keys?**
> - No secrets to rotate — GitHub mints a short-lived OIDC token (valid for ~15 minutes) for each workflow run
> - No risk of key leaks — there are no long-lived access keys stored anywhere
> - The IAM role `pharma-dev-github-actions-role` was created by the IAM module in Module 1.7, with a trust policy that only allows GitHub Actions from your specific repository to assume it

**ECR login:**

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
```

This action uses the OIDC-assumed role to call `ecr:GetAuthorizationToken`, which returns a 12-hour Docker login token for your ECR registry.

**Trivy vulnerability scan:**

```yaml
- name: Trivy — scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.tag.outputs.image_tag }}
    format: sarif
    output: trivy-results.sarif
    severity: HIGH,CRITICAL
    exit-code: '0'
```

> **Why Trivy?**
> Trivy scans the Docker image's OS packages and application libraries for known vulnerabilities (CVEs). It catches issues that npm audit misses — for example, vulnerabilities in Alpine Linux packages, Nginx, or other system-level dependencies.

> **Why `exit-code: '0'`?**
> With `exit-code: '0'`, Trivy reports vulnerabilities but does not fail the pipeline. This is appropriate for initial setup — you can see what vulnerabilities exist without blocking deployments. Change to `exit-code: '1'` once your team has addressed existing findings and wants to enforce a vulnerability-free gate.

> **Why SARIF format?**
> SARIF (Static Analysis Results Interchange Format) is a standard format for security findings. Trivy outputs results in SARIF for structured reporting and integration with other tools.

#### Job 6: `deploy-dev` (needs: docker-build-push)

```yaml
deploy-dev:
  name: Deploy — DEV
  needs: docker-build-push
  runs-on: ubuntu-latest
  environment: dev
```

This job checks out the gitops repository, uses `yq` to update the image tag in `envs/dev/values-pharma-ui.yaml`, and pushes the change directly.

> **Why direct push (not a PR) for DEV?**
> DEV is the fast iteration environment. Developers need to see their changes running as quickly as possible after merging. Requiring a PR approval for every DEV deployment would slow down the feedback loop. The code has already been reviewed via the PR to `develop` — the DEV deployment is the automated consequence of that approval.

> **Why `environment: dev`?**
> GitHub Environments provide:
> - A deployment history visible in the repository's "Deployments" section
> - The ability to add environment-specific secrets and protection rules later
> - Links from the workflow run to the environment, making it easy to track what is deployed where

> **How does the deployment actually happen?**
> This job only updates a YAML file in the gitops repo. ArgoCD (installed in Module 3) watches the gitops repo. When ArgoCD detects the change to `envs/dev/values-pharma-ui.yaml`, it automatically syncs the DEV application — pulling the new image from ECR and rolling out the updated pods.

> **What about promoting to QA?** The CI pipeline only deploys to DEV. QA and PROD promotions are handled by separate manual workflows (covered in sections 4.5 and 4.6). This keeps the default pipeline fast and avoids cluttering the gitops repo with PRs for every dev build.

> **Don't commit yet** — we will commit this workflow along with the QA and PROD promotion workflows as a single commit in section 4.6.

---

## 4.5 Create QA Promotion Workflow

The CI pipeline deploys to DEV automatically on every merge to `develop`. But promoting to QA should be intentional — you want to validate the DEV deployment first, then explicitly choose to promote it.

> **Why not auto-promote to QA?** If the pipeline automatically opened a QA PR on every build, you'd get a new PR for every merge to `develop`. Most of those are work-in-progress — you don't want to promote every one. Making QA promotion manual keeps the gitops repo clean and ensures only validated builds reach QA. This also makes the promotion flow consistent: DEV is automatic, QA and PROD are both manual triggers.

### Step 1: Create the Workflow

Create `.github/workflows/promote-qa-pharma-ui.yml`:

```yaml
name: Promote pharma-ui to QA

# Triggered manually — picks up the image currently running in DEV and opens
# a PR in the GitOps repo to promote it to QA.

on:
  workflow_dispatch:

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

      - name: Validate DEV values file exists
        run: |
          DEV_VALUES="_gitops/envs/dev/values-pharma-ui.yaml"
          if [[ ! -f "$DEV_VALUES" ]]; then
            echo "::error::DEV values file not found: ${DEV_VALUES}"
            echo "::error::Cannot determine which image is running in DEV."
            exit 1
          fi
          echo "DEV_VALUES=${DEV_VALUES}" >> "$GITHUB_ENV"

      - name: Read image tag from DEV
        run: |
          IMAGE_TAG=$(yq e '.image.tag' "$DEV_VALUES")
          if [[ -z "$IMAGE_TAG" || "$IMAGE_TAG" == "null" ]]; then
            echo "::error::Could not read .image.tag from $DEV_VALUES"
            exit 1
          fi
          echo "IMAGE_TAG=${IMAGE_TAG}" >> "$GITHUB_ENV"
          echo "Promoting image: ${IMAGE_TAG}"

      - name: Create branch, commit, open PR
        env:
          GH_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          ACTOR: ${{ github.actor }}
          SERVER_URL: ${{ github.server_url }}
          REPOSITORY: ${{ github.repository }}
          RUN_ID: ${{ github.run_id }}
          GITOPS_REPO_NAME: ${{ env.GITOPS_REPO }}
        run: |
          VALUES="_gitops/envs/qa/values-pharma-ui.yaml"
          BRANCH="promote/qa/pharma-ui/${IMAGE_TAG}"

          if [[ ! -f "$VALUES" ]]; then
            echo "::warning::$VALUES not found — QA PR skipped."
            exit 0
          fi

          cd _gitops
          git checkout -b "$BRANCH"
          yq e ".image.tag = \"${IMAGE_TAG}\"" -i "envs/qa/values-pharma-ui.yaml"
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .
          if git diff --staged --quiet; then
            echo "QA already has this image — no PR needed"
            exit 0
          fi
          git commit -m "promote(qa): pharma-ui → ${IMAGE_TAG}"
          git push --force-with-lease origin "$BRANCH"

          printf '%s\n' \
            "## QA Promotion" \
            "" \
            "Service : pharma-ui" \
            "Image   : ${IMAGE_TAG}" \
            "Promoted by : ${ACTOR}" \
            "Build   : ${SERVER_URL}/${REPOSITORY}/actions/runs/${RUN_ID}" \
            "" \
            "## Checklist" \
            "- [ ] DEV smoke test passed" \
            "- [ ] No unexpected config changes in this diff" \
            "- [ ] QA team sign-off" \
            "" \
            "Merging this PR updates envs/qa/values-pharma-ui.yaml." \
            "ArgoCD (pharma-qa) auto-syncs on merge." > /tmp/pr-body.md

          PR_URL=$(gh pr view --repo "$GITOPS_REPO_NAME" --head "$BRANCH" --json url -q .url 2>/dev/null || true)
          if [[ -n "$PR_URL" ]]; then
            echo "::notice title=QA PR already open::${PR_URL}"
          else
            gh pr create \
              --repo "$GITOPS_REPO_NAME" \
              --title "promote(qa): pharma-ui → ${IMAGE_TAG}" \
              --body-file /tmp/pr-body.md \
              --base main \
              --head "$BRANCH"
            echo "::notice title=QA PR opened::Review and merge in ${GITOPS_REPO_NAME} to deploy to QA."
          fi
```

### Step 2: Key Design Decisions

> **Why read from DEV instead of specifying a tag?** The workflow reads the image tag from `envs/dev/values-pharma-ui.yaml` — whatever is currently running in DEV. This guarantees you promote exactly what you tested. You can't accidentally promote an untested image.

> **Consistent promotion pattern:** Both QA and PROD promotions now work the same way — manual trigger → read from lower environment → open PR → review → merge → ArgoCD syncs. The only difference is where they read from: QA reads from DEV, PROD reads from QA.

**The promotion flow is now:**
```
Developer pushes to develop
    ↓
CI: lint → test → scan → build → Docker → ECR → deploy DEV (automatic)
    ↓
Developer validates DEV deployment
    ↓
Manual trigger: "Promote pharma-ui to QA" workflow
    ↓
Opens PR in gitops repo → review → merge → ArgoCD syncs QA
    ↓
Manual trigger: "Promote pharma-ui to PROD" workflow
    ↓
Opens PR in gitops repo → review → merge → ArgoCD syncs PROD
```

> **Don't commit yet** — we will commit this workflow along with the CI and PROD promotion workflows as a single commit in section 4.6.

---

## 4.6 Create Production Promotion Workflow

The production promotion workflow is a separate file, triggered manually. It reads the image tag currently deployed in QA and opens a PR to deploy that same image to PROD.

### Step 1: Create the Workflow

Create `.github/workflows/promote-prod-pharma-ui.yml`:

```yaml
name: Promote pharma-ui to PROD

# Triggered manually — picks up the image currently running in QA and opens
# a PR in the GitOps repo to promote it to PROD.
# PROD ArgoCD app requires manual sync after merge.

on:
  workflow_dispatch:

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
        run: |
          QA_VALUES="_gitops/envs/qa/values-pharma-ui.yaml"
          if [[ ! -f "$QA_VALUES" ]]; then
            echo "::error::QA values file not found: ${QA_VALUES}"
            echo "::error::Cannot determine which image is running in QA."
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
        run: |
          PROD_VALUES="_gitops/envs/prod/values-pharma-ui.yaml"
          if [[ ! -f "$PROD_VALUES" ]]; then
            echo "::error::PROD values file not found: ${PROD_VALUES}"
            exit 1
          fi
          echo "PROD_VALUES=envs/prod/values-pharma-ui.yaml" >> "$GITHUB_ENV"

      - name: Create branch, patch values, open PR
        env:
          GH_TOKEN: ${{ secrets.GITOPS_TOKEN }}
          ACTOR: ${{ github.actor }}
          SERVER_URL: ${{ github.server_url }}
          REPOSITORY: ${{ github.repository }}
          RUN_ID: ${{ github.run_id }}
          GITOPS_REPO_NAME: ${{ env.GITOPS_REPO }}
        run: |
          BRANCH="promote/prod/pharma-ui/${IMAGE_TAG}"

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
          git commit -m "promote(prod): pharma-ui → ${IMAGE_TAG}"
          git push origin "$BRANCH"

          printf '%s\n' \
            "## PROD Promotion" \
            "" \
            "Service : pharma-ui" \
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
            "ArgoCD app \`pharma-prod/pharma-ui\` is configured for **manual sync**." \
            "After merging, trigger sync in ArgoCD and monitor the rollout." \
            "" \
            "Merging this PR updates ${PROD_VALUES}." > /tmp/pr-body.md

          gh pr create \
            --repo "$GITOPS_REPO_NAME" \
            --title "promote(prod): pharma-ui → ${IMAGE_TAG}" \
            --body-file /tmp/pr-body.md \
            --base main \
            --head "$BRANCH"

          echo "::notice title=PROD PR opened::Review, get approvals, and merge in ${GITOPS_REPO_NAME}. Then manually sync ArgoCD pharma-prod/pharma-ui."
```

### Step 2: Key Design Decisions

> **Why `workflow_dispatch` only (manual trigger)?**
> Production deployments should always be a deliberate human decision. Automatically deploying to PROD on every merge would be dangerous for a pharmaceutical application where compliance and change management are critical.

> **Why read the image tag from QA instead of specifying it manually?**
> This guarantees you promote exactly what was tested in QA. If a developer could type in an arbitrary image tag, they might accidentally promote an untested image. By reading from `envs/qa/values-pharma-ui.yaml`, the workflow enforces the promotion chain: DEV -> QA -> PROD.

> **Why does PROD ArgoCD require manual sync?**
> Even after the PR is merged, ArgoCD does not automatically roll out the new image in PROD. This is a safety net:
> 1. Merge the PR (updates the gitops repo)
> 2. Go to ArgoCD and click **Sync** on the `pharma-prod/pharma-ui` application
> 3. Monitor the rollout in ArgoCD
>
> This gives the team a chance to verify timing (deploy during a maintenance window), ensure monitoring is active, and have rollback procedures ready.

> **Why `environment: prod`?**
> GitHub Environments support **required reviewers**. You can configure the `prod` environment to require approval from specific team members before the workflow runs. This adds another gate: even clicking "Run workflow" is not enough — an approver must confirm.

**PROD promotion flow:**

```
Operator clicks "Run workflow" in GitHub Actions
    ↓
Workflow reads image tag from QA values file (e.g., sha-a1b2c3d)
    ↓
Creates branch: promote/prod/pharma-ui/sha-a1b2c3d
    ↓
Updates envs/prod/values-pharma-ui.yaml with the image tag
    ↓
Opens PR in gitops repo with pre-merge checklist
    ↓
Team reviews: QA sign-off ✓ | Change ticket ✓ | Runbook ✓
    ↓
PR merged → ArgoCD detects change
    ↓
Operator triggers manual sync in ArgoCD
    ↓
PROD deployment rolls out
```

### Step 3: Commit and Push All Workflows

Now commit all three workflow files (CI/CD, QA promotion, PROD promotion) as a single commit:

```bash
cd ~/devops/zenpharma/frontend
git checkout develop
git add .github/workflows/ci-pharma-ui.yml
git add .github/workflows/promote-qa-pharma-ui.yml
git add .github/workflows/promote-prod-pharma-ui.yml
git commit -m "ci: add CI/CD pipeline and promotion workflows for pharma-ui"
git push
```

> **Tag `frontend` repo: `module-4.4-ci-workflows`**
>
> ```bash
> git tag -a module-4.4-ci-workflows -m "Module 4.4: CI/CD pipeline, QA and PROD promotion workflows for pharma-ui"
> git push origin module-4.4-ci-workflows
> ```

---

## Module 4 Summary

| What We Built | Details |
|--------------|---------|
| **Dockerfile** | Multi-stage build: Node 22 Alpine (build) + Nginx 1.25 Alpine (serve), non-root user |
| **Nginx Config** | SPA routing, API proxy to backend, gzip compression, security headers |
| **Branch Protection** | `main` branch requires PR (0 approvals — solo-friendly), no direct pushes |
| **Branching Strategy** | `develop` for active work, `main` for stable releases, `feature/*` for individual changes |
| **GitHub Secrets** | `AWS_ACCOUNT_ID`, `GITOPS_TOKEN`, `SONAR_TOKEN` + `GITOPS_REPO`, `SONAR_ORG`, `SONAR_PROJECT_KEY_FRONTEND` variables |
| **CI/CD Workflow** | 6-job pipeline: Lint, Test, SonarCloud, Build, Docker Build & Push, Deploy DEV |
| **QA Promotion** | Manual workflow reads DEV image tag, opens PR in gitops repo for QA deployment |
| **PROD Promotion** | Manual workflow reads QA image tag, opens PR in gitops repo with approval checklist |

| Tag | Repo |
|-----|------|
| `module-4.1-dockerfile` | frontend |
| `module-4.4-ci-workflows` | frontend |

> **Next:** [Module 5 — GitOps Repository & Helm Charts](MODULE-5-GITOPS-REPOSITORY-AND-HELM-CHARTS.md)
