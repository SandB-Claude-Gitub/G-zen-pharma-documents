# Module 2 — Automating Terraform with GitHub Actions

> Automate Terraform plan, apply, and destroy through GitHub Actions with approval gates, concurrency controls, and branch protection.
> Estimated time: 1.5–2 hours.

---

## 2.1 Introduction to GitHub Actions

Before we create our Terraform workflow, let's understand how GitHub Actions works.

### What Are GitHub Actions?

GitHub Actions is a CI/CD platform built directly into GitHub. Instead of setting up a separate Jenkins server or CircleCI account, you define automation workflows as YAML files inside your repository. When something happens in your repo (a push, a PR, a manual trigger), GitHub spins up a virtual machine, runs your steps, and reports the results — all without any infrastructure to manage.

### Core Concepts

> **Events** — Things that happen in your repository that can trigger a workflow: a push to a branch, a pull request being opened, a manual button click, or a cron schedule.
>
> **Workflows** — YAML files stored in `.github/workflows/` that define what to do when an event occurs. One repo can have multiple workflows.
>
> **Jobs** — A workflow contains one or more jobs. Each job runs on its own fresh virtual machine (runner). By default, jobs run in parallel. You use `needs:` to make one job wait for another.
>
> **Steps** — A job contains a sequence of steps. Each step either runs a shell command (`run:`) or uses a pre-built action (`uses:`). Steps within a job run sequentially on the same runner.
>
> **Runners** — The virtual machines that execute your jobs. GitHub provides free hosted runners (Ubuntu, macOS, Windows). You can also self-host runners on your own machines.

### Workflow YAML Syntax

Every workflow file follows this structure:

```yaml
name: My Workflow              # Display name in the GitHub Actions UI

on:                            # WHEN to run
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:                          # WHAT to run
  build:
    runs-on: ubuntu-latest     # WHERE to run
    steps:
      - uses: actions/checkout@v5    # Pre-built action
      - run: echo "Hello World"      # Shell command
```

### Understanding Triggers

GitHub Actions supports many trigger types. Our Terraform workflow uses three:

**1. `push` — Triggered when code is pushed to a branch**
```yaml
on:
  push:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
```
This fires when someone pushes (or merges a PR) to `main`, but **only** if the changed files are under `envs/dev/` or `modules/`. Changing the README won't trigger an expensive Terraform run.

> **Why `paths` filtering?** Without it, every push to `main` — even editing a README — would trigger Terraform plan and apply. Path filtering saves CI minutes and avoids unnecessary plan/apply cycles. Only changes to Terraform code should trigger the Terraform workflow.

**2. `pull_request` — Triggered when a PR is opened or updated against a branch**
```yaml
on:
  pull_request:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
```
This fires when a PR targeting `main` is opened, updated, or reopened. We use this to run `terraform plan` so reviewers can see what will change before merging.

**3. `workflow_dispatch` — Triggered manually via the GitHub UI**
```yaml
on:
  workflow_dispatch:
    inputs:
      action:
        description: 'Terraform action'
        type: choice
        options: [plan, apply, destroy]
```
This adds a "Run workflow" button to the Actions tab. We use this for manual operations like destroying infrastructure. The `inputs` block creates form fields in the UI.

### Expressions and Contexts

GitHub Actions provides context objects you can reference in your workflow:

| Context | Example | What It Contains |
|---------|---------|-----------------|
| `github.ref` | `refs/heads/main` | The branch or tag that triggered the workflow |
| `github.event_name` | `push` | What triggered the workflow |
| `github.event.inputs.action` | `apply` | Values from `workflow_dispatch` form inputs |
| `secrets.AWS_ACCESS_KEY_ID` | (hidden) | Repository secrets |
| `vars.GH_ORG` | `zenpharma` | Repository variables |
| `steps.<id>.outcome` | `success` | Result of a previous step |

> **No tag needed** — this section is conceptual background, no code changes.

---

## 2.2 Create Terraform GitHub Actions Workflow

Let's start by creating the workflow file. Once we understand what the workflow does, we'll add the secrets and variables it needs in the next section.

### Step 1: Create the Workflow Directory

```bash
cd ~/devops/zenpharma/infra
mkdir -p .github/workflows
```

### Step 2: Create the Workflow File

Create the file `.github/workflows/terraform.yml`:

```bash
code ~/devops/zenpharma/infra/.github/workflows/terraform.yml
```

```yaml
name: Terraform Infrastructure

# PR to main        → plan only
# Push/merge to main → plan (auto) → approval gate → apply
# Manual dispatch    → plan, apply, or destroy dev environment

on:
  push:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
  pull_request:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
  workflow_dispatch:
    inputs:
      action:
        description: 'Terraform action'
        required: true
        default: 'plan'
        type: choice
        options: [plan, apply, destroy]
      confirm_destroy:
        description: 'Type "destroy" to confirm destruction (required when action=destroy)'
        required: false
        default: ''

permissions:
  contents: read

env:
  TF_VERSION: '1.15.6'
  AWS_REGION: us-east-1

# Prevent parallel runs on the same branch — avoids state conflicts
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false   # wait, don't cancel — safer for Terraform

jobs:
  # ── Plan ──────────────────────────────────────────────────────────────────
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: envs/dev

    steps:
      - uses: actions/checkout@v5

      - name: Setup Terraform ${{ env.TF_VERSION }}
        uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive
        continue-on-error: true

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -var="db_password=${{ secrets.DEV_DB_PASSWORD }}" \
            -var="jwt_secret=${{ secrets.DEV_JWT_SECRET }}" \
            -var="github_org=${{ vars.GH_ORG }}" \
            -out=tfplan \
            -no-color
        continue-on-error: true

      - name: Check plan status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      - name: Release state lock on failure
        if: failure() || cancelled()
        run: |
          aws s3 rm s3://${{ vars.TF_STATE_BUCKET }}/envs/dev/terraform.tfstate.tflock || true

      - name: Upload plan file
        if: steps.plan.outcome == 'success'
        uses: actions/upload-artifact@v6
        with:
          name: tfplan
          path: envs/dev/tfplan
          retention-days: 1

  # ── Apply ─────────────────────────────────────────────────────────────────
  apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: plan
    if: |
      (github.ref == 'refs/heads/main' && github.event_name == 'push') ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.action == 'apply')
    environment: dev   # ← pauses here and waits for manual approval
    defaults:
      run:
        working-directory: envs/dev

    steps:
      - uses: actions/checkout@v5

      - name: Setup Terraform ${{ env.TF_VERSION }}
        uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Download plan file
        uses: actions/download-artifact@v7
        with:
          name: tfplan
          path: envs/dev

      - name: Terraform Apply
        run: |
          terraform apply \
            -auto-approve \
            -no-color \
            tfplan

      - name: Release state lock on failure
        if: failure() || cancelled()
        run: |
          aws s3 rm s3://${{ vars.TF_STATE_BUCKET }}/envs/dev/terraform.tfstate.tflock || true

  # ── Destroy ───────────────────────────────────────────────────────────────
  destroy:
    name: Terraform Destroy
    runs-on: ubuntu-latest
    if: |
      github.event_name == 'workflow_dispatch' &&
      github.event.inputs.action == 'destroy' &&
      github.event.inputs.confirm_destroy == 'destroy'
    environment: dev   # ← approval gate for destroy too
    defaults:
      run:
        working-directory: envs/dev

    steps:
      - uses: actions/checkout@v5

      - name: Setup Terraform ${{ env.TF_VERSION }}
        uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Destroy
        run: |
          terraform destroy \
            -var="db_password=${{ secrets.DEV_DB_PASSWORD }}" \
            -var="jwt_secret=${{ secrets.DEV_JWT_SECRET }}" \
            -var="github_org=${{ vars.GH_ORG }}" \
            -auto-approve \
            -no-color

      - name: Release state lock on failure
        if: failure() || cancelled()
        run: |
          aws s3 rm s3://${{ vars.TF_STATE_BUCKET }}/envs/dev/terraform.tfstate.tflock || true
```

### Step 3: Understanding the Workflow — Section by Section

Let's walk through every important design decision in this workflow.

#### Triggers and Path Filtering

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
  pull_request:
    branches: [main]
    paths:
      - 'envs/dev/**'
      - 'modules/**'
```

The workflow triggers on three events, but `push` and `pull_request` use **path filtering**. The workflow only runs when files under `envs/dev/` or `modules/` change. This means:
- Editing the README or `.gitignore` does **not** trigger the workflow
- Adding a new module under `modules/` **does** trigger it
- Changing `envs/qa/` does **not** trigger it (we'd create a separate workflow for each environment)

> **Why not a single `'**/*.tf'` filter?** Being explicit about `envs/dev/**` and `modules/**` prevents accidentally triggering dev deployments when working on other environments. Each environment should have its own workflow (or its own path filter).

#### Permissions

```yaml
permissions:
  contents: read
```

> **Principle of least privilege.** By default, GitHub Actions tokens have write access to the repository. We explicitly restrict it to read-only since our workflow never needs to push commits or create releases. This limits the blast radius if someone compromises the workflow.

#### Concurrency Groups

```yaml
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false
```

> **Why concurrency control?** Terraform modifies real infrastructure and uses a state file. If two workflows run `terraform apply` at the same time, they can corrupt the state file and create orphaned resources. The concurrency group ensures only one Terraform workflow runs per branch at a time.
>
> **Why `cancel-in-progress: false`?** Unlike build workflows where cancelling is fine, cancelling a running `terraform apply` mid-execution can leave infrastructure in a broken half-applied state. We set this to `false` so a new run **waits** for the previous one to finish instead of cancelling it.

#### Plan Job — continue-on-error Pattern

```yaml
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -var="db_password=${{ secrets.DEV_DB_PASSWORD }}" \
            ...
            -out=tfplan \
            -no-color
        continue-on-error: true

      - name: Check plan status
        if: steps.plan.outcome == 'failure'
        run: exit 1
```

> **Why `continue-on-error: true` on plan, then check status separately?** This is a common pattern in GitHub Actions. If `terraform plan` fails (syntax error, missing variable, provider issue), we still want the subsequent steps to have a chance to run — specifically the "Release state lock on failure" step. If we let the plan step fail normally, GitHub Actions would skip all remaining steps, including the lock cleanup. By using `continue-on-error: true`, the job continues even if plan fails. We then explicitly check the plan outcome and fail the job ourselves.

#### State Lock Release on Failure

```yaml
      - name: Release state lock on failure
        if: failure() || cancelled()
        run: |
          aws s3 rm s3://${{ vars.TF_STATE_BUCKET }}/envs/dev/terraform.tfstate.tflock || true
```

> **Why release the lock manually?** When Terraform uses S3 native locking (`use_lockfile = true`), it creates a `.tflock` file in S3 before modifying state. If the CI runner crashes, gets cancelled, or the apply step fails mid-execution, Terraform might not get a chance to release the lock. This step ensures the lock is always cleaned up on failure or cancellation, so subsequent runs are not blocked.
>
> **Why `|| true` at the end?** If the lock file doesn't exist (because the failure happened before Terraform acquired the lock), the `aws s3 rm` command would fail. `|| true` prevents that failure from making the cleanup step itself fail.

#### Artifact Passing Between Jobs

```yaml
      # In the plan job:
      - name: Upload plan file
        uses: actions/upload-artifact@v6
        with:
          name: tfplan
          path: envs/dev/tfplan
          retention-days: 1

      # In the apply job:
      - name: Download plan file
        uses: actions/download-artifact@v7
        with:
          name: tfplan
          path: envs/dev
```

> **Why upload/download the plan file?** Each job runs on a separate virtual machine. The plan job generates a `tfplan` binary file that captures exactly what changes Terraform will make. The apply job needs this file to apply those exact changes — not re-plan and potentially apply different changes. GitHub Actions artifacts are the mechanism to pass files between jobs.
>
> **Why `retention-days: 1`?** Plan files are only valid for the current state of the infrastructure. A plan file from yesterday is dangerous — the infrastructure may have changed since then. We keep it for just 1 day to save storage.

#### Apply Job — Conditional Execution and Auto-Approve

```yaml
  apply:
    needs: plan
    if: |
      (github.ref == 'refs/heads/main' && github.event_name == 'push') ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.action == 'apply')
    environment: dev
```

The apply job has three important properties:

1. **`needs: plan`** — Only runs after the plan job succeeds
2. **`if:` condition** — Only runs on pushes to `main` (merge) or manual dispatch with `action: apply`. This prevents apply from running on pull requests — PRs only get a plan
3. **`environment: dev`** — This is the approval gate. GitHub pauses the workflow and waits for a designated reviewer to approve before proceeding (we'll configure this in Section 2.4)

```yaml
      - name: Terraform Apply
        run: |
          terraform apply \
            -auto-approve \
            -no-color \
            tfplan
```

> **Why `-auto-approve`?** Normally `terraform apply` asks "Do you want to perform these actions?" and waits for you to type `yes`. In CI, there's no one to type `yes`. We use `-auto-approve` to skip the prompt. This is safe because:
> 1. We're applying a **saved plan file** (`tfplan`), not re-planning — so the changes are exactly what was reviewed
> 2. A human already approved the PR (code review) and the environment gate (approval in GitHub)
> 3. The plan output was visible in the plan job's logs for review

#### Destroy Job — Safety Gates

```yaml
  destroy:
    if: |
      github.event_name == 'workflow_dispatch' &&
      github.event.inputs.action == 'destroy' &&
      github.event.inputs.confirm_destroy == 'destroy'
    environment: dev
```

> **Why so many safety gates for destroy?** Destroying infrastructure is irreversible. This job requires three things to line up:
> 1. It must be triggered **manually** (`workflow_dispatch`) — no automated destroys
> 2. The action input must be set to **`destroy`**
> 3. The user must type the word **`destroy`** in the confirmation field — this prevents accidental clicks
> 4. The `environment: dev` gate requires a reviewer to approve even after all that
>
> **Why does destroy NOT use a saved plan file?** Unlike apply, the destroy job does not depend on the plan job (`needs:` is absent). It runs `terraform destroy` directly, which computes its own destruction plan. This is intentional — you might want to destroy infrastructure even when the plan job would fail (e.g., if your Terraform code has errors but you need to tear down what was previously created).

### Step 4: Verify Your File Structure

```bash
find ~/devops/zenpharma/infra/.github -type f | sort
```

**Expected output:**
```
.github/workflows/terraform.yml
```

### Step 5: Commit and Push

```bash
cd ~/devops/zenpharma/infra
git add .github/workflows/terraform.yml
git commit -m "ci: add terraform GitHub Actions workflow"
git push origin main
```

> **Note:** Pushing this commit will NOT trigger the workflow — the `paths` filter only watches `envs/dev/**` and `modules/**`, and we only changed `.github/workflows/`. GitHub will register the workflow (it appears in the Actions tab), but it won't run until a commit modifies actual Terraform code. You can also trigger it manually using the **Run workflow** button (via `workflow_dispatch`) once it appears in the Actions tab.

> **Tag `infra` repo: `module-2.2-terraform-workflow`**
> ```bash
> cd ~/devops/zenpharma/infra
> git tag -a module-2.2-terraform-workflow -m "Module 2.2: Terraform GitHub Actions workflow"
> git push origin module-2.2-terraform-workflow
> ```

---

## 2.3 Adding Secrets and Variables to GitHub

Now that you've seen the workflow, you'll notice it references secrets like `${{ secrets.AWS_ACCESS_KEY_ID }}` and variables like `${{ vars.GH_ORG }}`. These need to be configured in GitHub before the workflow can run. GitHub provides two mechanisms: **Secrets** (encrypted, hidden in logs) and **Variables** (plaintext, visible in logs).

### Understanding Secrets vs. Variables

> **Secrets** are for sensitive values like passwords and access keys. They are encrypted at rest, never shown in workflow logs (GitHub automatically masks them), and cannot be read back after being set — only overwritten.
>
> **Variables** are for non-sensitive configuration like region names, bucket names, and organization names. They are stored in plaintext, visible in logs, and can be read back in the GitHub UI.
>
> **Rule of thumb:** If you would be uncomfortable seeing the value in a public build log, use a secret. Otherwise, use a variable.

### Step 1: Add Repository Secrets

1. Go to your infra repository: `https://github.com/zenpharma/infra`
2. Click **Settings** (you need admin access)
3. In the left sidebar, expand **Secrets and variables** and click **Actions**
4. You'll see two tabs at the top: **Secrets** and **Variables**
5. On the **Secrets** tab, click **New repository secret**

**Secret 1: AWS_ACCESS_KEY_ID**
- Name: `AWS_ACCESS_KEY_ID`
- Secret: Paste your IAM user's Access Key ID (from Module 1.1)
- Click **Add secret**

**Secret 2: AWS_SECRET_ACCESS_KEY**
- Name: `AWS_SECRET_ACCESS_KEY`
- Secret: Paste your IAM user's Secret Access Key
- Click **Add secret**

**Secret 3: DEV_DB_PASSWORD**
- Name: `DEV_DB_PASSWORD`
- Secret: A strong password for your dev database (e.g., `MyDevDb#2025!Secure`)
- Click **Add secret**

> **Important:** Use a strong password with uppercase, lowercase, numbers, and special characters. AWS RDS rejects weak passwords. Do NOT use quotes or backslashes in the password — they can cause escaping issues in shell commands.

**Secret 4: DEV_JWT_SECRET**
- Name: `DEV_JWT_SECRET`
- Secret: A random string for JWT signing (e.g., `dev-jwt-secret-zenpharma-2025`)
- Click **Add secret**

You should now see 4 secrets listed:

```
AWS_ACCESS_KEY_ID       Updated just now
AWS_SECRET_ACCESS_KEY   Updated just now
DEV_DB_PASSWORD         Updated just now
DEV_JWT_SECRET          Updated just now
```

> **Why static access keys instead of OIDC?** The ideal approach for GitHub Actions to access AWS is through OIDC federation — no static credentials to rotate. However, OIDC federation requires a GitHub Actions OIDC provider configured in IAM. Our Module 1 IAM module created this provider, but to use it we'd need to configure `aws-actions/configure-aws-credentials` with a role ARN and `role-to-assume`. For simplicity in this course, we use static access keys. In a production environment, you would use OIDC.

### Step 2: Add Repository Variables

1. On the same page (**Settings** > **Secrets and variables** > **Actions**), click the **Variables** tab
2. Click **New repository variable**

**Variable 1: GH_ORG**
- Name: `GH_ORG`
- Value: Your GitHub username or organization name (e.g., `zenpharma`)
- Click **Add variable**

**Variable 2: TF_STATE_BUCKET**
- Name: `TF_STATE_BUCKET`
- Value: Your S3 bucket name from Module 1.4 (e.g., `zen-pharma-terraform-state-<your-name>`)
- Click **Add variable**

You should now see 2 variables listed:

```
GH_ORG              zenpharma
TF_STATE_BUCKET     zen-pharma-terraform-state-<your-name>
```

### Step 3: Understand Environment-Scoped Secrets

GitHub also supports **environment-scoped secrets**. These are secrets that only apply to a specific deployment environment (like `dev`, `staging`, `prod`).

> **When to use environment secrets:** If you have different AWS accounts or different database passwords per environment, you would create a GitHub Environment for each (dev, staging, prod) and store credentials inside each environment. Workflows reference them via `environment: dev`, and only the secrets for that environment are available.
>
> **In our setup:** We use repository-level secrets because we have a single AWS account and a single `dev` environment. The `DEV_` prefix on `DEV_DB_PASSWORD` and `DEV_JWT_SECRET` makes it clear which environment they belong to. If we later add `qa` and `prod` environments, we would create environment-scoped secrets.

### Step 4: Verify Secrets Are Set

There is no way to view secret values after creation — this is by design. To verify they work, we'll run the workflow in Section 2.5. For now, double-check that you see all 4 secrets and 2 variables listed correctly.

---

## 2.4 Enable Branch Protection and Approval Process

Right now, anyone can push directly to `main`, which triggers an immediate apply. That's dangerous — a typo could destroy production infrastructure. We need two safety layers:

1. **Branch protection** — Prevent direct pushes to `main`; require pull requests with passing checks
2. **Environment approval** — Require a human to approve before `terraform apply` runs

### Step 1: Protect the Main Branch

1. Go to your infra repository: `https://github.com/<your-username>/infra`
2. Click **Settings**
3. In the left sidebar, click **Branches** (under "Code and automation")
4. Click **Add branch ruleset** (or **Add classic branch protection rule** if rulesets aren't available)

> **Note:** GitHub recently introduced **Branch rulesets** as the modern replacement for classic branch protection rules. Both work. The instructions below cover the classic approach, which is available on all plan types.

**If using classic branch protection rules:**

4. Click **Add classic branch protection rule**
5. Branch name pattern: `main`
6. Check the following options:

   - **Require a pull request before merging**
     - Set **Required number of approvals** to **0**
     - This forces all changes to go through a PR (so you get the plan review), but does not require someone else to approve it

7. **Do NOT enable** "Require status checks to pass before merging" — leave it unchecked
8. Click **Create** (or **Save changes**)

> **Why set required approvals to 0?**
> We want learners to experience the PR workflow — create a branch, open a PR, see the plan run, review it, then merge. But since you're working solo, requiring an approval from another person would block you. Setting it to 0 means: a PR is required (no direct pushes to `main`), but you can merge it yourself without waiting for a reviewer.
>
> In a real team, you would set this to 1 or 2 so that a colleague reviews every infrastructure change before it goes to `main`.

> **Why NOT require status checks?**
> GitHub matches status checks by their exact name (e.g., `Terraform Infrastructure / Terraform Plan`). If you type the name slightly wrong — different capitalization, missing the workflow prefix, extra spaces — the merge button stays blocked forever, waiting for a check that will never match. This is a common source of confusion. The plan still runs on every PR and you can see the result in the Actions tab before merging. The real safety gate is the **environment approval** on the apply job (next step), which is much harder to misconfigure.

**If using branch rulesets:**

4. Click **Add branch ruleset**
5. Ruleset name: `Protect main`
6. Enforcement status: **Active**
7. Under **Target branches**, click **Add target** > **Include by pattern** > type `main`
8. Under **Rules**, enable:
   - **Require a pull request before merging** (set required approvals to **0**)
9. Click **Create**

### Step 2: Create the GitHub Environment

GitHub Environments provide **deployment protection rules** — most importantly, required reviewers. When a workflow job references an environment, it pauses and waits for an approved reviewer to click "Approve" before continuing.

1. Go to your infra repository: `https://github.com/zenpharma/infra`
2. Click **Settings**
3. In the left sidebar, click **Environments** (under "Code and automation")
4. Click **New environment**
5. Name: `dev`
6. Click **Configure environment**
7. Under **Deployment protection rules**, check **Required reviewers**
8. In the search box, add yourself (your GitHub username) as a reviewer
9. Click **Save protection rules**

> **Why a `dev` environment with approval?** Our workflow's apply job has `environment: dev`. When this job runs, GitHub sees the environment protection rules and pauses the workflow. A notification is sent to the required reviewers. Only after a reviewer clicks "Approve and deploy" does the apply job proceed. This gives you a final checkpoint to review the plan output before applying changes to real infrastructure.
>
> **In a real team setup**, you would add senior engineers or a platform team as reviewers. For this course, you are both the author and the reviewer.

### Step 3: Understand the Full Workflow

Now that branch protection and environment approval are configured, here is the complete flow for making an infrastructure change:

```
1. Create a feature branch
   └── git checkout -b feat/add-new-module

2. Make changes and push
   └── git push origin feat/add-new-module

3. Open a Pull Request (PR) against main
   └── GitHub triggers: pull_request event
   └── Workflow runs: Plan job executes terraform plan
   └── PR shows: ✅ Terraform Plan — passed (or ❌ if plan fails)

4. Code Review
   └── Reviewer reads the PR diff (Terraform code changes)
   └── Reviewer checks the plan output in the Actions tab
   └── Reviewer approves the PR

5. Merge the PR
   └── GitHub triggers: push event (merge commit on main)
   └── Workflow runs: Plan job re-runs → Apply job starts
   └── Apply job pauses: "Waiting for review — dev environment"

6. Environment Approval
   └── Reviewer sees notification
   └── Reviews the plan output one final time
   └── Clicks "Approve and deploy"

7. Apply Executes
   └── terraform apply runs with the saved plan file
   └── Infrastructure is created/updated in AWS
```

> **Two levels of approval:**
> 1. **PR approval** — Reviews the code changes (what Terraform code was added/modified)
> 2. **Environment approval** — Reviews the plan output (what AWS resources will actually be created/changed/destroyed)
>
> This separation is important because the same Terraform code can produce different plans depending on the current state of the infrastructure.

---

## 2.5 Run Terraform Through GitHub Actions

Now let's test the full workflow by pushing a change through the CI pipeline.

### Step 1: Create a Feature Branch

Since we enabled branch protection, we can no longer push directly to `main`. All changes must go through a pull request.

```bash
cd ~/devops/zenpharma/infra
git checkout -b feat/trigger-ci
```

### Step 2: Make a Small Change

We need to change a file under `envs/dev/` or `modules/` to trigger the workflow (path filtering). Let's add a comment to the main Terraform configuration:

```bash
code ~/devops/zenpharma/infra/envs/dev/main.tf
```

Add a comment at the top of the file:

```hcl
# ZenPharma Dev Environment — managed via GitHub Actions CI/CD
locals {
  project = "pharma"
  env     = "dev"
  region  = "us-east-1"
}
...
```

### Step 3: Commit and Push the Feature Branch

```bash
cd ~/devops/zenpharma/infra
git add envs/dev/main.tf
git commit -m "ci: trigger initial CI pipeline run"
git push origin feat/trigger-ci
```

### Step 4: Create a Pull Request

**Option A — GitHub CLI:**

```bash
gh pr create \
  --title "ci: trigger initial terraform CI pipeline" \
  --body "Trigger the Terraform GitHub Actions workflow to verify the pipeline works end-to-end." \
  --base main
```

**Option B — GitHub Web UI:**

1. Go to `https://github.com/zenpharma/infra`
2. You'll see a yellow banner: "feat/trigger-ci had recent pushes — Compare & pull request"
3. Click **Compare & pull request**
4. Title: `ci: trigger initial terraform CI pipeline`
5. Description: `Trigger the Terraform GitHub Actions workflow to verify the pipeline works end-to-end.`
6. Click **Create pull request**

### Step 5: Watch the Plan Run

1. On the PR page, scroll down to the **Checks** section
2. You should see **Terraform Plan** running (or queued)
3. Click **Details** to see the workflow execution
4. Click on the **Terraform Plan** job in the left sidebar
5. Expand each step to see the output:
   - **Terraform Format Check** — Verifies code formatting
   - **Terraform Init** — Downloads providers and initializes backend
   - **Terraform Validate** — Checks syntax
   - **Terraform Plan** — Shows what will be created/changed/destroyed

**Expected plan output (summary line):**

```
Plan: XX to add, 0 to change, 0 to destroy.
```

> **Note:** The apply job will NOT run at this point — it only runs on pushes to `main`, not on pull requests. The PR only triggers the plan.

6. Once the plan succeeds, the PR check shows: **Terraform Plan — passed**

### Step 6: Review and Merge the PR

1. On the PR page, click **Files changed** to review the code diff
2. Add an approval (if you're the only reviewer, you may need to approve your own PR — or temporarily disable the approval requirement in branch protection settings)
3. Click **Merge pull request**
4. Click **Confirm merge**
5. Optionally delete the feature branch

### Step 7: Watch Plan and Apply on Main

After merging, the push to `main` triggers the workflow again:

1. Go to the **Actions** tab in your repository
2. You should see a new workflow run triggered by the merge
3. Click on the run to see both jobs:
   - **Terraform Plan** — Runs first (should succeed)
   - **Terraform Apply** — Waiting for environment approval

4. The apply job shows: **Waiting for review — dev**

### Step 8: Approve and Apply

1. Click on the **Terraform Apply** job
2. You'll see a banner: **Review deployments**
3. Click **Review deployments**
4. Check the **dev** environment checkbox
5. Optionally add a comment (e.g., "Plan looks good, applying")
6. Click **Approve and deploy**

The apply job now runs:
- Downloads the plan file from the plan job
- Runs `terraform init`
- Runs `terraform apply -auto-approve tfplan`

**Expected output:**
```
Apply complete! Resources: XX added, 0 changed, 0 destroyed.
```

### Step 9: Verify Infrastructure in AWS

1. Go to the **AWS Management Console**
2. Check the following resources exist:

**VPC:**
- Go to https://console.aws.amazon.com/vpc/
- You should see `pharma-dev-vpc`

**EKS (if included in your Terraform):**
- Go to https://console.aws.amazon.com/eks/
- You should see `pharma-dev-cluster`

**RDS (if included):**
- Go to https://console.aws.amazon.com/rds/
- You should see `pharma-dev-db`

**ECR (if included):**
- Go to https://console.aws.amazon.com/ecr/
- You should see your container repositories

3. Check the **Terraform state file** in S3:
   - Go to https://console.aws.amazon.com/s3/
   - Open your state bucket
   - Navigate to `envs/dev/terraform.tfstate`
   - The file should have been updated recently

> **Congratulations!** You've successfully run Terraform through a CI/CD pipeline. No one needs local AWS credentials to modify infrastructure anymore — all changes go through code review, automated planning, and approval gates.

---

## 2.6 Destroy Infrastructure via GitHub Actions

When you're done working or want to save costs, you can tear down all infrastructure. At this point in the course, only Terraform-managed resources exist (VPC, EKS, RDS, ECR, IAM, Secrets Manager) — no Helm charts or ArgoCD apps have been deployed yet, so a simple `terraform destroy` is all that's needed.

> **Note:** In later modules (after Helm charts and ArgoCD are installed), you will need to clean up those resources first before running destroy. That process is covered when we reach those modules.

### Step 1: Navigate to the Workflow

1. Go to your infra repository: `https://github.com/zenpharma/infra`
2. Click the **Actions** tab
3. In the left sidebar, click **Terraform Infrastructure** (your workflow name)

### Step 2: Trigger the Destroy

1. Click the **Run workflow** dropdown (top right area)
2. You'll see a form with the following fields:
   - **Branch:** `main` (leave as default)
   - **Terraform action:** Select `destroy` from the dropdown
   - **Type "destroy" to confirm destruction:** Type `destroy` in this field
3. Click **Run workflow**

### Step 3: Approve the Destroy

The destroy job has `environment: dev`, so it pauses for approval just like apply:

1. Click on the running workflow
2. The **Terraform Destroy** job shows: **Waiting for review — dev**
3. Click **Review deployments**
4. Check the **dev** environment checkbox
5. Add a comment: "Confirmed destroy for cost savings"
6. Click **Approve and deploy**

### Step 4: Monitor the Destroy

The destroy job runs `terraform destroy -auto-approve`. This will take **10-15 minutes** depending on the resources (EKS clusters and NAT Gateways are slow to delete).

**Expected output (last line):**
```
Destroy complete! Resources: XX destroyed.
```

### Step 5: Verify Cleanup

After the destroy completes, verify in the AWS Console:

```bash
# Check no EKS clusters remain
aws eks list-clusters --region us-east-1

# Check no RDS instances remain
aws rds describe-db-instances --region us-east-1 --query 'DBInstances[].DBInstanceIdentifier'

# Check no VPCs remain (besides the default VPC)
aws ec2 describe-vpcs --region us-east-1 --query 'Vpcs[?Tags]'
```

All should return empty results (except the default VPC which always exists).

### Understanding the Destroy Safety Gates

Let's recap all the protections preventing accidental destruction:

| Safety Gate | How It Protects |
|-------------|----------------|
| **Manual trigger only** | Destroy cannot be triggered by a push or PR — only by manually clicking "Run workflow" |
| **Action selection** | You must explicitly choose "destroy" from the dropdown — "plan" is the default |
| **Confirmation text** | You must type the word "destroy" in a separate field — prevents accidental clicks |
| **`if:` condition** | The workflow checks all three conditions: `workflow_dispatch` AND `action == 'destroy'` AND `confirm_destroy == 'destroy'` |
| **Environment approval** | Even after all the above, a reviewer must click "Approve and deploy" |

> **Five layers of protection.** An accidental destroy requires someone to: (1) go to the Actions tab, (2) click Run workflow, (3) select "destroy" from the dropdown, (4) type the word "destroy" in the confirmation field, (5) approve the environment deployment. This level of paranoia is appropriate — a single `terraform destroy` can delete databases, clusters, and all the data in them.

> **End of Module 2.** Infrastructure can now be planned, applied, and destroyed entirely through GitHub Actions. No team member needs local AWS credentials for day-to-day infrastructure changes — everything goes through code review, automated planning, and approval gates.

---

## Module 2 Summary

| What We Configured | Details |
|-------------------|---------|
| **GitHub Secrets** | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, DEV_DB_PASSWORD, DEV_JWT_SECRET |
| **GitHub Variables** | GH_ORG (org name), TF_STATE_BUCKET (S3 bucket name) |
| **Terraform Workflow** | 3 jobs: Plan (on every push/PR), Apply (on merge to main + approval), Destroy (manual + confirmation) |
| **Concurrency Control** | One Terraform run per branch at a time, no cancellation |
| **Artifact Passing** | Plan file uploaded by plan job, downloaded by apply job |
| **Branch Protection** | Required PRs (0 approvals — solo-friendly), no direct pushes to main |
| **Environment Approval** | `dev` environment with required reviewer before apply/destroy (the real safety gate) |
| **State Lock Cleanup** | Automatic lock release on failure or cancellation |

| Tag | Repos |
|-----|-------|
| `module-2.2-terraform-workflow` | infra |

> **Next:** [Module 3 — Kubernetes Cluster Bootstrap](MODULE-3-KUBERNETES-CLUSTER-BOOTSTRAP.md)
