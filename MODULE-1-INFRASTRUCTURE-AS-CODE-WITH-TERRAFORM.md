# Module 1 — Infrastructure as Code with Terraform

> Set up AWS credentials, configure GitHub repositories, and provision the entire AWS infrastructure (VPC, EKS, RDS, ECR, IAM, Secrets Manager) using Terraform.
> Estimated time: 2–3 hours.

---

## 1.1 Create and Configure AWS Credentials

We need an IAM user with programmatic access so that both the AWS CLI and Terraform can interact with your AWS account.

### Step 1: Create an IAM User

1. Sign in to the **AWS Management Console** at https://console.aws.amazon.com/
2. In the top search bar, type **IAM** and click on the IAM service
3. In the left sidebar, click **Users**
4. Click **Create user**
5. Enter a username: `terraform-admin`
6. Click **Next**
7. Select **Attach policies directly**
8. Search for and check **AdministratorAccess**
   > **Note:** In production, you would use least-privilege policies. For this course, admin access simplifies setup.
9. Click **Next**, then **Create user**

### Step 2: Create Access Keys

1. Click on the user `terraform-admin` you just created
2. Go to the **Security credentials** tab
3. Scroll down to **Access keys** and click **Create access key**
4. Select **Command Line Interface (CLI)**
5. Check the acknowledgement box, click **Next**, then **Create access key**
6. **IMPORTANT:** Copy both the **Access Key ID** and **Secret Access Key** — you won't be able to see the secret key again

### Step 3: Configure AWS CLI

Open your terminal and run:

```bash
aws configure
```

You will be prompted for 4 values:

```
AWS Access Key ID [None]: <paste your Access Key ID>
AWS Secret Access Key [None]: <paste your Secret Access Key>
Default region name [None]: us-east-1
Default output format [None]: json
```

### Step 4: Verify

```bash
aws sts get-caller-identity
```

**Expected output:**
```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/terraform-admin"
}
```

Note down your **Account ID** (the 12-digit number) — you will need it later.

> **No tag needed** — this step has no code changes.

---

## 1.2 GitHub Organization and Repositories

The developers have already committed the source code for all microservices into a GitHub organization called **zenpharma**, with repositories called `frontend` and `backend`. You can find them here:

- **Frontend:** https://github.com/zenpharma/frontend
- **Backend:** https://github.com/zenpharma/backend

You have two options to get started:

### Option A: Fork the Repositories (Recommended)

Forking is the quickest way — you get your own copy of the code under your GitHub account.

1. Go to https://github.com/zenpharma/frontend
2. Click **Fork** (top right)
3. Select your GitHub account as the owner
4. Click **Create fork**
5. Repeat for https://github.com/zenpharma/backend

This gives you:
- `https://github.com/<your-username>/frontend`
- `https://github.com/<your-username>/backend`

### Option B: Create Your Own Organization (Optional)

If you want the full experience of setting up an organization from scratch:

1. Go to https://github.com/ and sign in
2. Click your **profile icon** (top right) → **Your organizations**
3. Click **New organization**
4. Select the **Free** plan
5. Enter organization name: `zenpharma` (or your preferred name — org names must be globally unique)
6. Enter your contact email
7. Select **My personal account** as the owner
8. Complete the verification and click **Create organization**

Then create the `frontend` and `backend` repos inside the organization and push the source code to them.

> **Note:** Throughout this course, replace `zenpharma` with your actual GitHub username or organization name wherever you see it in commands and configuration files.

### Step 1: Create the `infra` and `gitops` Repositories

Regardless of which option you chose above, you need to create two additional repositories for infrastructure and GitOps:

**Repository: infra**
1. Go to your GitHub account or organization page
2. Click **New** (or the green **Create a new repository** button)
3. Repository name: `infra`
4. Description: `Terraform infrastructure code for ZenPharma`
5. Select **Private**
6. Check **Add a README file**
7. Add `.gitignore`: select **Terraform**
8. Click **Create repository**

**Repository: gitops**
1. Click **New** again
2. Repository name: `gitops`
3. Description: `GitOps repository — Helm charts, ArgoCD apps, environment configs`
4. Select **Private**
5. Check **Add a README file**
6. Click **Create repository**

### Step 2: Clone All Repositories

```bash
cd ~/devops/zenpharma

# Replace <your-username> with your GitHub username or org name
git clone git@github.com:<your-username>/infra.git
git clone git@github.com:<your-username>/frontend.git
git clone git@github.com:<your-username>/backend.git
git clone git@github.com:<your-username>/gitops.git
```

> **Note:** If SSH is not yet configured (next step), use HTTPS temporarily:
> ```bash
> git clone https://github.com/<your-username>/infra.git
> ```

Your directory should now look like:
```
~/devops/zenpharma/
├── infra/          ← empty (we add Terraform code starting in step 1.5)
├── frontend/       ← contains React app source code (from fork or manual push)
├── backend/        ← contains 8 microservice directories (from fork or manual push)
└── gitops/         ← empty (we build this in Module 5)
```

> **Important:** At this stage, the frontend and backend repos contain **only source code** — no Dockerfiles, no `.github/workflows`. We add those in later modules.

> **Tag `infra` and `gitops` repos: `module-1.2-initial-code`**
> ```bash
> cd ~/devops/zenpharma/infra
> git tag -a module-1.2-initial-code -m "Module 1.2: Initial repo structure"
> git push origin module-1.2-initial-code
>
> cd ~/devops/zenpharma/gitops
> git tag -a module-1.2-initial-code -m "Module 1.2: Initial repo structure"
> git push origin module-1.2-initial-code
> ```

---

## 1.3 Configure Passwordless GitHub Authentication (SSH)

To manage these repositories (push, pull, clone), we need to add our SSH public key to our GitHub account. This lets Git authenticate without prompting for a username and password every time.

### Step 1: Check for Existing SSH Keys

```bash
ls -la ~/.ssh/
```

If you see files like `id_ed25519` and `id_ed25519.pub`, you already have an SSH key — skip to Step 3.

### Step 2: Generate a New SSH Key

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

When prompted:
- **File location:** Press Enter to accept the default (`~/.ssh/id_ed25519`)
- **Passphrase:** Press Enter for no passphrase (simplest option for this course)

This creates two files:
- `~/.ssh/id_ed25519` — your **private key** (never share this)
- `~/.ssh/id_ed25519.pub` — your **public key** (this is what we give to GitHub)

### Step 3: Copy the Public Key

**macOS:**
```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

**Linux:**
```bash
cat ~/.ssh/id_ed25519.pub
# Select and copy the entire output
```

**Windows (PowerShell):**
```powershell
Get-Content ~/.ssh/id_ed25519.pub | Set-Clipboard
```

### Step 4: Add the Public Key to GitHub

1. Go to https://github.com/settings/keys
2. Click **New SSH key**
3. Title: `My MacBook` (or any descriptive name for your machine)
4. Key type: **Authentication Key**
5. Paste the public key into the **Key** field
6. Click **Add SSH key**

> **How does this work?** When you `git push`, Git sends your private key's signature to GitHub. GitHub checks it against the public key you just added. If they match, you're authenticated — no password needed.

### Step 5: Test the Connection

```bash
ssh -T git@github.com
```

If this is your first time connecting, you'll see:
```
The authenticity of host 'github.com' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6...
Are you sure you want to continue connecting (yes/no)?
```

Type `yes` and press Enter.

**Expected output:**
```
Hi your-username! You've successfully authenticated, but GitHub does not provide shell access.
```

### Step 6: Update Repository Remotes to SSH (If Needed)

If you cloned repos with HTTPS earlier, update them to use SSH:

```bash
# Replace <your-username> with your GitHub username or org name
cd ~/devops/zenpharma/infra
git remote set-url origin git@github.com:<your-username>/infra.git

cd ~/devops/zenpharma/frontend
git remote set-url origin git@github.com:<your-username>/frontend.git

cd ~/devops/zenpharma/backend
git remote set-url origin git@github.com:<your-username>/backend.git

cd ~/devops/zenpharma/gitops
git remote set-url origin git@github.com:<your-username>/gitops.git
```

**Verify each remote:**
```bash
cd ~/devops/zenpharma/infra
git remote -v
# Expected:
# origin  git@github.com:<your-username>/infra.git (fetch)
# origin  git@github.com:<your-username>/infra.git (push)
```

> **No tag needed** — this step has no code changes.

---

## 1.4 Create S3 Bucket for Terraform State

Terraform stores the state of your infrastructure in a **state file**. By default, this is stored locally on your machine. For team collaboration and CI/CD, we store it in S3.

### Step 1: Create the S3 Bucket

Replace `<your-name>` with a unique identifier (S3 bucket names must be globally unique):

```bash
aws s3api create-bucket \
  --bucket zen-pharma-terraform-state-<your-name> \
  --region us-east-1
```

> **Note:** For `us-east-1`, you do NOT need the `--create-bucket-configuration` flag. For other regions, add:
> `--create-bucket-configuration LocationConstraint=<region>`

### Step 2: Enable Versioning

Versioning protects you from accidental state file corruption — you can always roll back to a previous version.

```bash
aws s3api put-bucket-versioning \
  --bucket zen-pharma-terraform-state-<your-name> \
  --versioning-configuration Status=Enabled
```

### Step 3: Enable Server-Side Encryption

Terraform state files contain sensitive data — resource IDs, RDS endpoints, IAM role ARNs. We enable default encryption so every object stored in this bucket is automatically encrypted at rest.

```bash
aws s3api put-bucket-encryption \
  --bucket zen-pharma-terraform-state-<your-name> \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'
```

> **Why AES256 (SSE-S3)?** AWS manages the encryption keys for you — no key management overhead. For stricter compliance needs, you could use `aws:kms` (SSE-KMS) with a customer-managed key, but SSE-S3 is sufficient for this course and has no additional cost.

### Step 4: Block Public Access

State files should never be publicly accessible. This is a safety net in case someone misconfigures a bucket policy later.

```bash
aws s3api put-public-access-block \
  --bucket zen-pharma-terraform-state-<your-name> \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Step 5: Verify

```bash
# Check the bucket exists
aws s3 ls | grep zen-pharma

# Check versioning is enabled
aws s3api get-bucket-versioning \
  --bucket zen-pharma-terraform-state-<your-name>
```

**Expected output:**
```json
{
    "Status": "Enabled"
}
```

```bash
# Check encryption is enabled
aws s3api get-bucket-encryption \
  --bucket zen-pharma-terraform-state-<your-name>
```

**Expected output:**
```json
{
    "ServerSideEncryptionConfiguration": {
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                },
                "BucketKeyEnabled": true
            }
        ]
    }
}
```

### Step 6: Verify in AWS Console (GUI)

1. Go to https://console.aws.amazon.com/s3/
2. You should see your bucket `zen-pharma-terraform-state-<your-name>`
3. Click on the bucket → **Properties** tab
4. Scroll to **Bucket Versioning** — it should say **Enabled**
5. Scroll to **Default encryption** — it should say **SSE-S3** with **Bucket Key enabled**
6. Click the **Permissions** tab
7. **Block public access** — all four settings should say **On**

> **Why S3 for state?**
> - **Team collaboration:** Multiple people can work on the same infrastructure
> - **State locking:** Prevents two people from running Terraform at the same time (we use S3 native locking with `use_lockfile = true`)
> - **Versioning:** Roll back if the state gets corrupted
> - **Encryption:** State files are encrypted at rest with AES-256
> - **Public access blocked:** State files are never accidentally exposed to the internet

> **No tag needed** — this is an AWS resource, no code change yet.

---

## 1.5 Add VPC Terraform Module

Now we start writing Terraform code. We'll create the folder structure and add the VPC module first.

### Step 1: Create the Folder Structure

```bash
cd ~/devops/zenpharma/infra

# Create module directories
mkdir -p modules/vpc

# Create environment directories
mkdir -p envs/dev
mkdir -p envs/qa
mkdir -p envs/prod
```

Your directory should now look like:
```
infra/
├── .gitignore
├── README.md
├── envs/
│   ├── dev/        ← we'll add files here
│   ├── qa/         ← empty for now
│   └── prod/       ← empty for now
└── modules/
    └── vpc/        ← we'll add files here
```

### Step 2: Create the .gitignore

Make sure your `.gitignore` prevents committing state files, lock files, and sensitive data. Open `~/devops/zenpharma/infra/.gitignore` in VS Code and replace its contents:

```bash
code ~/devops/zenpharma/infra/.gitignore
```

```gitignore
# Terraform state — NEVER commit state files
**/*.tfstate
**/*.tfstate.*
**/.terraform/
**/.terraform.lock.hcl

# Terraform plan output
*.tfplan
*.tfplan.json

# Sensitive variable files
*.tfvars
*.tfvars.json
!example.tfvars

# Crash logs
crash.log
crash.*.log

# Override files (local developer overrides)
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Terraform CLI config
.terraformrc
terraform.rc

# OS files
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
```

### Step 3: Create the VPC Module

**File: `modules/vpc/variables.tf`**

```bash
code ~/devops/zenpharma/infra/modules/vpc/variables.tf
```

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "env" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets (EKS)"
  type        = list(string)
}

variable "database_subnet_cidrs" {
  description = "CIDR blocks for database subnets (RDS)"
  type        = list(string)
}
```

> **What are CIDR blocks?**
> A CIDR block like `10.0.0.0/16` defines a range of IP addresses. The `/16` means the first 16 bits are fixed, giving us 65,536 addresses. We split this range into smaller subnets:
> - **Public subnets** (`10.0.1.0/24`, `10.0.2.0/24`) — for resources that need internet access (ALBs)
> - **Private subnets** (`10.0.3.0/24`, `10.0.4.0/24`) — for EKS nodes (internet access through NAT Gateway)
> - **Database subnets** (`10.0.5.0/24`, `10.0.6.0/24`) — for RDS (no internet access)

**File: `modules/vpc/main.tf`**

```bash
code ~/devops/zenpharma/infra/modules/vpc/main.tf
```

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "${var.project}-${var.env}-vpc"
  cidr = var.vpc_cidr

  azs              = ["${var.region}a", "${var.region}b"]
  public_subnets   = var.public_subnet_cidrs
  private_subnets  = var.private_subnet_cidrs
  database_subnets = var.database_subnet_cidrs

  enable_nat_gateway           = true
  single_nat_gateway           = true
  enable_dns_hostnames         = true
  enable_dns_support           = true
  create_database_subnet_group = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb"                          = "1"
    "kubernetes.io/cluster/${var.project}-${var.env}-cluster"  = "owned"
  }

  tags = {
    Project = var.project
    Env     = var.env
  }
}
```

> **Key decisions explained:**
> - We use the official **terraform-aws-modules/vpc** community module instead of writing raw `aws_vpc`, `aws_subnet`, etc. resources. This saves hundreds of lines of code.
> - **`single_nat_gateway = true`** — Uses one NAT Gateway instead of one per AZ. Saves ~$30/month but is a single point of failure. Fine for dev, not for production.
> - **Subnet tags** — The `kubernetes.io/role/elb` and `kubernetes.io/role/internal-elb` tags tell the AWS Load Balancer Controller which subnets to use when creating ALBs.
> - **2 AZs** (`us-east-1a`, `us-east-1b`) — Gives us high availability without the cost of 3 AZs.

**File: `modules/vpc/outputs.tf`**

```bash
code ~/devops/zenpharma/infra/modules/vpc/outputs.tf
```

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = module.vpc.vpc_id
}

output "public_subnets" {
  description = "List of public subnet IDs"
  value       = module.vpc.public_subnets
}

output "private_subnets" {
  description = "List of private subnet IDs"
  value       = module.vpc.private_subnets
}

output "database_subnets" {
  description = "List of database subnet IDs"
  value       = module.vpc.database_subnets
}

output "database_subnet_group_name" {
  description = "Name of the database subnet group"
  value       = module.vpc.database_subnet_group_name
}
```

### Step 4: Create the Dev Environment Configuration

**File: `envs/dev/backend.tf`** — Tells Terraform where to store state

```bash
code ~/devops/zenpharma/infra/envs/dev/backend.tf
```

```hcl
terraform {
  backend "s3" {
    bucket       = "zen-pharma-terraform-state-<your-name>"  # Replace with your S3 bucket name
    key          = "envs/dev/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true   # S3 native locking (Terraform >= 1.11)
  }
}
```

> **`use_lockfile = true`** — This is a Terraform 1.11+ feature that uses S3 native locking. It creates a `.tflock` file next to the state file. This prevents two people (or CI jobs) from running `terraform apply` at the same time and corrupting the state.

**File: `envs/dev/providers.tf`** — Configures the AWS provider

```bash
code ~/devops/zenpharma/infra/envs/dev/providers.tf
```

```hcl
terraform {
  required_version = ">= 1.11.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Project   = "pharma"
      Env       = "dev"
      ManagedBy = "terraform"
    }
  }
}
```

> **`default_tags`** — Every resource Terraform creates will automatically get these tags. This makes it easy to find and filter resources in the AWS Console and calculate costs per project/environment.

**File: `envs/dev/variables.tf`** — Empty for now (we'll add variables in later steps)

```bash
code ~/devops/zenpharma/infra/envs/dev/variables.tf
```

```hcl
# Variables will be added as we add more modules
```

**File: `envs/dev/outputs.tf`** — Empty for now

```bash
code ~/devops/zenpharma/infra/envs/dev/outputs.tf
```

```hcl
# Outputs will be added as we add more modules
```

**File: `envs/dev/main.tf`** — Calls the VPC module with dev-specific values

```bash
code ~/devops/zenpharma/infra/envs/dev/main.tf
```

```hcl
locals {
  project = "pharma"
  env     = "dev"
  region  = "us-east-1"
}

data "aws_caller_identity" "current" {}

module "vpc" {
  source = "../../modules/vpc"

  project               = local.project
  env                   = local.env
  region                = local.region
  vpc_cidr              = "10.0.0.0/16"
  public_subnet_cidrs   = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs  = ["10.0.3.0/24", "10.0.4.0/24"]
  database_subnet_cidrs = ["10.0.5.0/24", "10.0.6.0/24"]
}
```

> **Why `locals` instead of `variables`?**
> These values (project name, env name, region) are fixed per environment and should never be overridden from the outside. `locals` makes them constants. `variables` are for values that change between runs (like database passwords).

### Step 5: Verify Your File Structure

```bash
find ~/devops/zenpharma/infra -type f -name "*.tf" | sort
```

**Expected output:**
```
envs/dev/backend.tf
envs/dev/main.tf
envs/dev/outputs.tf
envs/dev/providers.tf
envs/dev/variables.tf
modules/vpc/main.tf
modules/vpc/outputs.tf
modules/vpc/variables.tf
```

### Step 6: Commit and Push

```bash
cd ~/devops/zenpharma/infra
git add .
git commit -m "feat: add VPC terraform module and dev environment config"
git push origin main
```

> **Tag `infra` repo: `module-1.5-vpc-module`**
> ```bash
> git tag -a module-1.5-vpc-module -m "Module 1.5: VPC Terraform module with dev environment"
> git push origin module-1.5-vpc-module
> ```

---

## 1.6 Run Terraform Plan and Apply — VPC

### Step 1: Initialize Terraform

```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform init
```

**What `terraform init` does:**
- Downloads the AWS provider plugin
- Downloads the `terraform-aws-modules/vpc/aws` module
- Configures the S3 backend for state storage
- Creates a `.terraform/` directory (ignored by `.gitignore`)

**Expected output (last few lines):**
```
Terraform has been successfully initialized!
```

### Step 2: Validate the Configuration

```bash
terraform validate
```

**Expected output:**
```
Success! The configuration is valid.
```

### Step 3: Run Terraform Plan

```bash
terraform plan
```

This shows you what Terraform **will** create without actually creating anything. Review the output carefully.

**Expected output (summary):**
```
Plan: 24 to add, 0 to change, 0 to destroy.
```

> **What are the 24 resources?**
> The VPC module creates: VPC, Internet Gateway, NAT Gateway, Elastic IP (for NAT), 2 public subnets, 2 private subnets, 2 database subnets, public route table, 2 private route tables, database route table, route table associations, database subnet group, and various routes.

### Step 4: Run Terraform Apply

```bash
terraform apply
```

Terraform will show the plan again and ask for confirmation:
```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

Type `yes` and press Enter.

**This will take 2-3 minutes** (mainly waiting for the NAT Gateway).

**Expected output (last few lines):**
```
Apply complete! Resources: 24 added, 0 changed, 0 destroyed.
```

### Step 5: Verify in AWS Console

1. Go to https://console.aws.amazon.com/vpc/
2. Make sure you're in the **us-east-1** region (top right dropdown)
3. Click **Your VPCs** — you should see `pharma-dev-vpc`
4. Click **Subnets** — you should see 6 subnets:
   - `pharma-dev-vpc-public-us-east-1a`
   - `pharma-dev-vpc-public-us-east-1b`
   - `pharma-dev-vpc-private-us-east-1a`
   - `pharma-dev-vpc-private-us-east-1b`
   - `pharma-dev-vpc-db-us-east-1a`
   - `pharma-dev-vpc-db-us-east-1b`
5. Click **NAT gateways** — you should see 1 NAT Gateway
6. Click **Internet gateways** — you should see 1 Internet Gateway attached to the VPC

### Step 6: Inspect the State File in S3

1. Go to https://console.aws.amazon.com/s3/
2. Click on your bucket `zen-pharma-terraform-state-<your-name>`
3. Navigate to `envs/dev/`
4. You should see `terraform.tfstate` — this is your state file
5. Click on it → **Versions** tab — you should see at least one version

---

## 1.7 Add IAM Terraform Module

The IAM module creates service account roles that allow Kubernetes workloads to access AWS services without storing any static credentials. This pattern is called **IRSA (IAM Roles for Service Accounts)**.

### Understanding IRSA and OIDC

> **Problem:** Pods running in EKS need to access AWS services (Secrets Manager, ECR, etc.). We could put AWS access keys inside the pods, but that's insecure and hard to manage.
>
> **Solution: IRSA**
> 1. EKS creates an **OIDC provider** (a trusted identity source)
> 2. We create IAM roles that **trust** specific Kubernetes service accounts via the OIDC provider
> 3. When a pod starts with that service account, AWS automatically gives it temporary credentials for the linked IAM role
> 4. No static credentials needed — credentials rotate automatically every hour

### Step 1: Create the IAM Module Directory

```bash
cd ~/devops/zenpharma/infra
mkdir -p modules/iam
```

### Step 2: Create the IAM Module Variables

**File: `modules/iam/variables.tf`**

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "env" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "oidc_provider_arn" {
  description = "ARN of the EKS OIDC provider"
  type        = string
}

variable "oidc_provider_url" {
  description = "URL of the EKS OIDC provider"
  type        = string
}

variable "aws_account_id" {
  description = "AWS Account ID"
  type        = string
}

variable "github_org" {
  description = "GitHub organization or username that owns frontend and backend"
  type        = string
}
```

### Step 3: Create the IAM Module — IRSA Roles

**File: `modules/iam/main.tf`**

This file creates 4 IRSA roles:

```hcl
# ─── External Secrets Operator (ESO) IRSA Role ─────────────────────────────
# Allows ESO to read secrets from AWS Secrets Manager

data "aws_iam_policy_document" "eso_assume_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      variable = "${replace(var.oidc_provider_url, "https://", "")}:sub"
      values   = ["system:serviceaccount:external-secrets:external-secrets"]
    }

    condition {
      test     = "StringEquals"
      variable = "${replace(var.oidc_provider_url, "https://", "")}:aud"
      values   = ["sts.amazonaws.com"]
    }

    principals {
      identifiers = [var.oidc_provider_arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "eso_role" {
  name               = "${var.project}-${var.env}-eso-role"
  assume_role_policy = data.aws_iam_policy_document.eso_assume_role.json

  tags = {
    Name    = "${var.project}-${var.env}-eso-role"
    Env     = var.env
    Project = var.project
  }
}

resource "aws_iam_policy" "eso_secrets_policy" {
  name        = "${var.project}-${var.env}-eso-secrets-policy"
  description = "Allow External Secrets Operator to read pharma secrets from AWS Secrets Manager"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = "arn:aws:secretsmanager:*:${var.aws_account_id}:secret:/pharma/*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eso_secrets_attachment" {
  role       = aws_iam_role.eso_role.name
  policy_arn = aws_iam_policy.eso_secrets_policy.arn
}

# ─── ArgoCD IRSA Role ──────────────────────────────────────────────────────

data "aws_iam_policy_document" "argocd_assume_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      variable = "${replace(var.oidc_provider_url, "https://", "")}:sub"
      values   = ["system:serviceaccount:argocd:argocd-application-controller"]
    }

    condition {
      test     = "StringEquals"
      variable = "${replace(var.oidc_provider_url, "https://", "")}:aud"
      values   = ["sts.amazonaws.com"]
    }

    principals {
      identifiers = [var.oidc_provider_arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "argocd_role" {
  name               = "${var.project}-${var.env}-argocd-role"
  assume_role_policy = data.aws_iam_policy_document.argocd_assume_role.json

  tags = {
    Name    = "${var.project}-${var.env}-argocd-role"
    Env     = var.env
    Project = var.project
  }
}

# ─── AWS Load Balancer Controller IRSA Role ─────────────────────────────────

data "aws_iam_policy_document" "alb_controller_assume_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      variable = "${replace(var.oidc_provider_url, "https://", "")}:sub"
      values   = ["system:serviceaccount:kube-system:aws-load-balancer-controller"]
    }

    condition {
      test     = "StringEquals"
      variable = "${replace(var.oidc_provider_url, "https://", "")}:aud"
      values   = ["sts.amazonaws.com"]
    }

    principals {
      identifiers = [var.oidc_provider_arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "alb_controller_role" {
  name               = "${var.project}-${var.env}-alb-controller-role"
  assume_role_policy = data.aws_iam_policy_document.alb_controller_assume_role.json

  tags = {
    Name    = "${var.project}-${var.env}-alb-controller-role"
    Env     = var.env
    Project = var.project
  }
}

resource "aws_iam_policy" "alb_controller_policy" {
  name        = "${var.project}-${var.env}-alb-controller-policy"
  description = "IAM policy for AWS Load Balancer Controller"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["iam:CreateServiceLinkedRole"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "iam:AWSServiceName" = "elasticloadbalancing.amazonaws.com"
          }
        }
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeAccountAttributes",
          "ec2:DescribeAddresses",
          "ec2:DescribeAvailabilityZones",
          "ec2:DescribeInternetGateways",
          "ec2:DescribeVpcs",
          "ec2:DescribeVpcPeeringConnections",
          "ec2:DescribeSubnets",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeInstances",
          "ec2:DescribeNetworkInterfaces",
          "ec2:DescribeTags",
          "ec2:GetCoipPoolUsage",
          "ec2:DescribeCoipPools",
          "ec2:GetSecurityGroupsForVpc",
          "ec2:DescribeIpamPools",
          "ec2:DescribeRouteTables",
          "elasticloadbalancing:DescribeLoadBalancers",
          "elasticloadbalancing:DescribeLoadBalancerAttributes",
          "elasticloadbalancing:DescribeListeners",
          "elasticloadbalancing:DescribeListenerCertificates",
          "elasticloadbalancing:DescribeSSLPolicies",
          "elasticloadbalancing:DescribeRules",
          "elasticloadbalancing:DescribeTargetGroups",
          "elasticloadbalancing:DescribeTargetGroupAttributes",
          "elasticloadbalancing:DescribeTargetHealth",
          "elasticloadbalancing:DescribeTags",
          "elasticloadbalancing:DescribeTrustStores",
          "elasticloadbalancing:DescribeListenerAttributes",
          "elasticloadbalancing:DescribeCapacityReservation"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "cognito-idp:DescribeUserPoolClient",
          "acm:ListCertificates",
          "acm:DescribeCertificate",
          "iam:ListServerCertificates",
          "iam:GetServerCertificate",
          "waf-regional:GetWebACL",
          "waf-regional:GetWebACLForResource",
          "waf-regional:AssociateWebACL",
          "waf-regional:DisassociateWebACL",
          "wafv2:GetWebACL",
          "wafv2:GetWebACLForResource",
          "wafv2:AssociateWebACL",
          "wafv2:DisassociateWebACL",
          "shield:GetSubscriptionState",
          "shield:DescribeProtection",
          "shield:CreateProtection",
          "shield:DeleteProtection"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:AuthorizeSecurityGroupIngress",
          "ec2:RevokeSecurityGroupIngress",
          "ec2:CreateSecurityGroup",
          "ec2:CreateTags",
          "ec2:DeleteTags",
          "ec2:DeleteSecurityGroup",
          "ec2:ModifyNetworkInterfaceAttribute"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "elasticloadbalancing:CreateLoadBalancer",
          "elasticloadbalancing:CreateTargetGroup",
          "elasticloadbalancing:CreateListener",
          "elasticloadbalancing:DeleteListener",
          "elasticloadbalancing:CreateRule",
          "elasticloadbalancing:DeleteRule",
          "elasticloadbalancing:AddTags",
          "elasticloadbalancing:RemoveTags",
          "elasticloadbalancing:ModifyLoadBalancerAttributes",
          "elasticloadbalancing:SetIpAddressType",
          "elasticloadbalancing:SetSecurityGroups",
          "elasticloadbalancing:SetSubnets",
          "elasticloadbalancing:DeleteLoadBalancer",
          "elasticloadbalancing:ModifyTargetGroup",
          "elasticloadbalancing:ModifyTargetGroupAttributes",
          "elasticloadbalancing:DeleteTargetGroup",
          "elasticloadbalancing:ModifyListenerAttributes",
          "elasticloadbalancing:ModifyCapacityReservation",
          "elasticloadbalancing:RegisterTargets",
          "elasticloadbalancing:DeregisterTargets",
          "elasticloadbalancing:SetWebAcl",
          "elasticloadbalancing:ModifyListener",
          "elasticloadbalancing:AddListenerCertificates",
          "elasticloadbalancing:RemoveListenerCertificates",
          "elasticloadbalancing:ModifyRule"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "alb_controller_policy_attachment" {
  role       = aws_iam_role.alb_controller_role.name
  policy_arn = aws_iam_policy.alb_controller_policy.arn
}
```

### Step 4: Create GitHub Actions OIDC Federation

This is separate from IRSA. It allows GitHub Actions to assume an AWS IAM role without storing any static AWS credentials in GitHub Secrets.

**File: `modules/iam/github-actions-oidc.tf`**

```hcl
# ─── GitHub Actions OIDC Federation ─────────────────────────────────────────
#
# Allows GitHub Actions workflows in your repos to assume an IAM role
# without any long-lived AWS credentials stored in GitHub Secrets.
#
# How it works:
#   1. GitHub mints a short-lived OIDC token per workflow run
#   2. The workflow calls aws-actions/configure-aws-credentials with
#      role-to-assume: <this role ARN>
#   3. AWS validates the token against the registered OIDC provider and
#      issues temporary STS credentials (valid for 1 hour max)

resource "aws_iam_openid_connect_provider" "github_actions" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd",
  ]

  tags = {
    Name    = "github-actions-oidc-provider"
    Project = var.project
  }
}

data "aws_iam_policy_document" "github_actions_assume_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github_actions.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values = [
        "repo:${var.github_org}/frontend:ref:refs/heads/main",
        "repo:${var.github_org}/frontend:ref:refs/heads/develop",
        "repo:${var.github_org}/backend:ref:refs/heads/main",
        "repo:${var.github_org}/backend:ref:refs/heads/develop",
      ]
    }
  }
}

resource "aws_iam_role" "github_actions_ci" {
  name                 = "${var.project}-${var.env}-github-actions-role"
  assume_role_policy   = data.aws_iam_policy_document.github_actions_assume_role.json
  max_session_duration = 3600

  tags = {
    Name    = "${var.project}-${var.env}-github-actions-role"
    Env     = var.env
    Project = var.project
  }
}

resource "aws_iam_policy" "github_actions_ci_policy" {
  name        = "${var.project}-${var.env}-github-actions-policy"
  description = "Allow GitHub Actions CI to push images to ECR and read EKS cluster info"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRAuth"
        Effect = "Allow"
        Action = ["ecr:GetAuthorizationToken"]
        Resource = "*"
      },
      {
        Sid    = "ECRPush"
        Effect = "Allow"
        Action = [
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:DescribeRepositories",
          "ecr:ListImages",
          "ecr:DescribeImages",
        ]
        Resource = "arn:aws:ecr:*:${var.aws_account_id}:repository/*"
      },
      {
        Sid    = "EKSRead"
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:ListClusters",
        ]
        Resource = "*"
      },
    ]
  })
}

resource "aws_iam_role_policy_attachment" "github_actions_ci_policy_attachment" {
  role       = aws_iam_role.github_actions_ci.name
  policy_arn = aws_iam_policy.github_actions_ci_policy.arn
}
```

> **Why OIDC instead of Access Keys?**
> - **No static credentials to rotate or leak** — GitHub gets temporary 1-hour tokens
> - **Scoped to specific repos and branches** — The trust policy only allows your repos on `main` and `develop` branches
> - **AWS best practice** — AWS recommends OIDC federation for all CI/CD systems

### Step 5: Create the IAM Module Outputs

**File: `modules/iam/outputs.tf`**

```hcl
output "eso_role_arn" {
  description = "ARN of the External Secrets Operator IAM role"
  value       = aws_iam_role.eso_role.arn
}

output "argocd_role_arn" {
  description = "ARN of the ArgoCD IAM role"
  value       = aws_iam_role.argocd_role.arn
}

output "alb_controller_role_arn" {
  description = "ARN of the AWS Load Balancer Controller IAM role"
  value       = aws_iam_role.alb_controller_role.arn
}
```

### Step 6: Update Dev Environment to Use IAM Module

We cannot add the IAM module yet because it depends on EKS (it needs the OIDC provider ARN and URL). We'll wire it up in Step 1.8 when we add all remaining modules together.

### Step 7: Commit and Push

```bash
cd ~/devops/zenpharma/infra
git add modules/iam/
git commit -m "feat: add IAM terraform module with IRSA roles and GitHub OIDC"
git push origin main
```

> **Tag `infra` repo: `module-1.7-iam-module`**
> ```bash
> git tag -a module-1.7-iam-module -m "Module 1.7: IAM module with IRSA and GitHub OIDC federation"
> git push origin module-1.7-iam-module
> ```

---

## 1.8 Add Remaining Terraform Modules (EKS, ECR, RDS, Secrets Manager)

Now we add the remaining 4 modules and wire everything together in `envs/dev/main.tf`.

### Step 1: Create the EKS Module

```bash
mkdir -p ~/devops/zenpharma/infra/modules/eks
```

**File: `modules/eks/variables.tf`**

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "env" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for the EKS cluster"
  type        = string
}

variable "subnet_ids" {
  description = "Subnet IDs for EKS nodes"
  type        = list(string)
}

variable "kubernetes_version" {
  description = "Kubernetes version for the EKS cluster"
  type        = string
  default     = "1.33"
}

variable "instance_types" {
  description = "EC2 instance types for the node group"
  type        = list(string)
  default     = ["t3.medium"]
}

variable "desired_size" {
  description = "Desired number of worker nodes"
  type        = number
  default     = 2
}

variable "min_size" {
  description = "Minimum number of worker nodes"
  type        = number
  default     = 1
}

variable "max_size" {
  description = "Maximum number of worker nodes"
  type        = number
  default     = 3
}
```

**File: `modules/eks/main.tf`**

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0"

  name               = "${var.project}-${var.env}-cluster"
  kubernetes_version = var.kubernetes_version

  vpc_id     = var.vpc_id
  subnet_ids = var.subnet_ids

  endpoint_private_access = true
  endpoint_public_access  = true

  enable_irsa                              = true
  enable_cluster_creator_admin_permissions = true

  addons = {
    vpc-cni = {
      most_recent    = true
      before_compute = true
    }
    kube-proxy             = { most_recent = true }
    coredns                = { most_recent = true }
    eks-pod-identity-agent = { most_recent = true }
  }

  eks_managed_node_groups = {
    main = {
      instance_types = var.instance_types
      min_size       = var.min_size
      max_size       = var.max_size
      desired_size   = var.desired_size
    }
  }

  tags = {
    Project = var.project
    Env     = var.env
  }
}
```

> **Key decisions explained:**
> - **`enable_irsa = true`** — Creates the OIDC provider that powers IRSA (IAM Roles for Service Accounts)
> - **`enable_cluster_creator_admin_permissions = true`** — Gives the IAM user who creates the cluster admin access to Kubernetes
> - **`endpoint_public_access = true`** — Allows `kubectl` access from your local machine. In production, you might restrict this to a VPN
> - **Addons:** `vpc-cni` (pod networking), `kube-proxy` (service networking), `coredns` (DNS), `eks-pod-identity-agent` (newer pod identity system)
> - **`t3.small` instance type with 3 desired nodes** — Small enough to keep costs low, large enough to run all services

**File: `modules/eks/outputs.tf`**

```hcl
output "cluster_name" {
  description = "Name of the EKS cluster"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "Endpoint for the EKS cluster API server"
  value       = module.eks.cluster_endpoint
}

output "cluster_certificate_authority_data" {
  description = "Base64 encoded certificate data for the cluster"
  value       = module.eks.cluster_certificate_authority_data
}

output "oidc_provider_arn" {
  description = "ARN of the OIDC Provider for IRSA"
  value       = module.eks.oidc_provider_arn
}

output "cluster_oidc_issuer_url" {
  description = "URL of the OIDC Provider for IRSA"
  value       = module.eks.cluster_oidc_issuer_url
}

output "node_security_group_id" {
  description = "Security group ID of the EKS node group"
  value       = module.eks.node_security_group_id
}

output "cluster_security_group_id" {
  description = "Security group ID of the EKS cluster"
  value       = module.eks.cluster_security_group_id
}
```

---

### Step 2: Create the ECR Module

```bash
mkdir -p ~/devops/zenpharma/infra/modules/ecr
```

**File: `modules/ecr/variables.tf`**

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "env" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "repositories" {
  description = "List of ECR repository names to create"
  type        = list(string)
}
```

**File: `modules/ecr/main.tf`**

```hcl
resource "aws_ecr_repository" "main" {
  for_each = toset(var.repositories)

  name                 = each.value
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  tags = {
    Name    = "${var.project}-${each.value}"
    Env     = var.env
    Project = var.project
  }
}

resource "aws_ecr_lifecycle_policy" "main" {
  for_each   = toset(var.repositories)
  repository = aws_ecr_repository.main[each.key].name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 images"
        selection = {
          tagStatus   = "any"
          countType   = "imageCountMoreThan"
          countNumber = 10
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}
```

> **Key decisions explained:**
> - **`for_each = toset(var.repositories)`** — Creates one ECR repo per microservice from a list. No copy-paste needed.
> - **`scan_on_push = true`** — Automatically scans images for vulnerabilities when pushed
> - **Lifecycle policy** — Keeps only the last 10 images per repo to avoid storage costs

**File: `modules/ecr/outputs.tf`**

```hcl
output "repository_urls" {
  description = "Map of repository name to repository URL"
  value       = { for name, repo in aws_ecr_repository.main : name => repo.repository_url }
}
```

---

### Step 3: Create the RDS Module

```bash
mkdir -p ~/devops/zenpharma/infra/modules/rds
```

**File: `modules/rds/variables.tf`**

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "env" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for the RDS security group"
  type        = string
}

variable "db_subnet_group_name" {
  description = "Name of the DB subnet group"
  type        = string
}

variable "eks_node_security_group_id" {
  description = "Security group ID of EKS nodes to allow RDS access"
  type        = string
}

variable "db_name" {
  description = "Name of the database to create"
  type        = string
  default     = "pharmadb"
}

variable "username" {
  description = "Master username for the database"
  type        = string
}

variable "password" {
  description = "Master password for the database"
  type        = string
  sensitive   = true
}

variable "password_version" {
  description = "Increment to trigger a password update"
  type        = number
  default     = 1
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "allocated_storage" {
  description = "Allocated storage in GB"
  type        = number
  default     = 20
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = false
}

variable "skip_final_snapshot" {
  description = "Skip final snapshot on deletion"
  type        = bool
  default     = true
}

variable "backup_retention_period" {
  description = "Number of days to retain backups"
  type        = number
  default     = 0
}

variable "deletion_protection" {
  description = "Enable deletion protection"
  type        = bool
  default     = false
}
```

**File: `modules/rds/main.tf`**

```hcl
resource "aws_security_group" "rds" {
  name        = "${var.project}-${var.env}-rds-sg"
  description = "Security group for RDS PostgreSQL instance"
  vpc_id      = var.vpc_id

  ingress {
    description     = "PostgreSQL from EKS worker nodes"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.eks_node_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name    = "${var.project}-${var.env}-rds-sg"
    Project = var.project
    Env     = var.env
  }
}

module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 7.0"

  identifier = "${var.project}-${var.env}-postgres"

  engine               = "postgres"
  engine_version       = "17.9"
  family               = "postgres17"
  major_engine_version = "17"
  instance_class       = var.instance_class

  allocated_storage = var.allocated_storage
  storage_type      = "gp3"

  db_name                     = var.db_name
  username                    = var.username
  manage_master_user_password = false
  password_wo                 = var.password
  password_wo_version         = var.password_version

  multi_az               = var.multi_az
  db_subnet_group_name   = var.db_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds.id]

  skip_final_snapshot     = var.skip_final_snapshot
  backup_retention_period = var.backup_retention_period
  storage_encrypted       = true
  deletion_protection     = var.deletion_protection
  publicly_accessible     = false

  create_db_option_group = false

  tags = {
    Name    = "${var.project}-${var.env}-postgres"
    Project = var.project
    Env     = var.env
  }
}
```

> **Key decisions explained:**
> - **Security group** — Only allows connections on port 5432 from EKS node security group. No public internet access.
> - **`db.t3.micro`** — Smallest/cheapest instance for dev. Upgrade to `db.t3.medium` or larger for production.
> - **`skip_final_snapshot = true`** — For dev only. In production, set to `false` to take a final backup before deletion.
> - **`storage_encrypted = true`** — Always encrypt data at rest.
> - **`publicly_accessible = false`** — The database is only accessible from within the VPC (via EKS pods).

**File: `modules/rds/outputs.tf`**

```hcl
output "db_instance_endpoint" {
  description = "Connection endpoint for the RDS instance (hostname:port)"
  value       = module.rds.db_instance_endpoint
}

output "db_instance_address" {
  description = "Hostname of the RDS instance (without port)"
  value       = module.rds.db_instance_address
}

output "db_instance_port" {
  description = "Port of the RDS instance"
  value       = module.rds.db_instance_port
}

output "db_instance_name" {
  description = "Name of the database"
  value       = module.rds.db_instance_name
}
```

---

### Step 4: Create the Secrets Manager Module

```bash
mkdir -p ~/devops/zenpharma/infra/modules/secrets-manager
```

**File: `modules/secrets-manager/variables.tf`**

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "env" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "db_username" {
  description = "Database username to store in Secrets Manager"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database password to store in Secrets Manager"
  type        = string
  sensitive   = true
}

variable "jwt_secret" {
  description = "JWT signing secret to store in Secrets Manager"
  type        = string
  sensitive   = true
}

variable "db_host" {
  description = "RDS endpoint hostname to store alongside credentials"
  type        = string
}
```

**File: `modules/secrets-manager/main.tf`**

```hcl
resource "aws_secretsmanager_secret" "db_credentials" {
  name                    = "/pharma/${var.env}/db-credentials"
  description             = "Database credentials for the pharma ${var.env} environment"
  recovery_window_in_days = 0

  tags = {
    Name    = "/pharma/${var.env}/db-credentials"
    Env     = var.env
    Project = var.project
  }
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
    host     = var.db_host
  })
}

resource "aws_secretsmanager_secret" "jwt_secret" {
  name                    = "/pharma/${var.env}/jwt-secret"
  description             = "JWT signing secret for the pharma ${var.env} environment"
  recovery_window_in_days = 0

  tags = {
    Name    = "/pharma/${var.env}/jwt-secret"
    Env     = var.env
    Project = var.project
  }
}

resource "aws_secretsmanager_secret_version" "jwt_secret" {
  secret_id = aws_secretsmanager_secret.jwt_secret.id
  secret_string = jsonencode({
    secret = var.jwt_secret
  })
}
```

> **How does this connect to Kubernetes?**
> 1. Terraform creates secrets in AWS Secrets Manager (this module)
> 2. External Secrets Operator (installed in Module 3) watches for `ExternalSecret` CRDs in Kubernetes
> 3. ESO reads from AWS Secrets Manager using its IRSA role (created in Module 1.7)
> 4. ESO creates regular Kubernetes `Secret` objects that pods can consume
> 5. **Result:** Secrets are managed in one place (AWS), automatically synced to Kubernetes

**File: `modules/secrets-manager/outputs.tf`**

```hcl
output "db_secret_arn" {
  description = "ARN of the database credentials secret"
  value       = aws_secretsmanager_secret.db_credentials.arn
}

output "jwt_secret_arn" {
  description = "ARN of the JWT signing secret"
  value       = aws_secretsmanager_secret.jwt_secret.arn
}
```

---

### Step 5: Wire Everything Together in `envs/dev/main.tf`

Now update the dev environment to use all 6 modules. The order matters because of dependencies.

**File: `envs/dev/main.tf`** — Replace the entire file:

```hcl
locals {
  project = "pharma"
  env     = "dev"
  region  = "us-east-1"
}

data "aws_caller_identity" "current" {}

module "vpc" {
  source = "../../modules/vpc"

  project               = local.project
  env                   = local.env
  region                = local.region
  vpc_cidr              = "10.0.0.0/16"
  public_subnet_cidrs   = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs  = ["10.0.3.0/24", "10.0.4.0/24"]
  database_subnet_cidrs = ["10.0.5.0/24", "10.0.6.0/24"]
}

module "eks" {
  source = "../../modules/eks"

  project            = local.project
  env                = local.env
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnets
  kubernetes_version = "1.33"
  instance_types     = ["t3.small"]
  min_size           = 1
  max_size           = 4
  desired_size       = 3
}

module "rds" {
  source = "../../modules/rds"

  project                    = local.project
  env                        = local.env
  username                   = "pharmaadmin"
  password                   = var.db_password
  vpc_id                     = module.vpc.vpc_id
  db_subnet_group_name       = module.vpc.database_subnet_group_name
  eks_node_security_group_id = module.eks.node_security_group_id
}

module "ecr" {
  source = "../../modules/ecr"

  project = local.project
  env     = local.env
  repositories = [
    "api-gateway",
    "auth-service",
    "drug-catalog-service",
    "inventory-service",
    "manufacturing-service",
    "notification-service",
    "pharma-ui",
    "supplier-service",
    "qc-service",
  ]
}

module "iam" {
  source = "../../modules/iam"

  project           = local.project
  env               = local.env
  oidc_provider_arn = module.eks.oidc_provider_arn
  oidc_provider_url = module.eks.cluster_oidc_issuer_url
  aws_account_id    = data.aws_caller_identity.current.account_id
  github_org        = var.github_org
}

module "secrets_manager" {
  source = "../../modules/secrets-manager"

  project     = local.project
  env         = local.env
  db_username = "pharmaadmin"
  db_password = var.db_password
  db_host     = module.rds.db_instance_address
  jwt_secret  = var.jwt_secret
}
```

> **Dependency chain (Terraform figures this out automatically):**
> ```
> VPC → EKS (needs VPC ID + private subnets)
>     → RDS (needs VPC ID + database subnet group + EKS node security group)
>     → IAM (needs EKS OIDC provider ARN + URL)
>     → Secrets Manager (needs RDS endpoint)
> ECR has no dependencies — runs in parallel with EKS
> ```

### Step 6: Update `envs/dev/providers.tf`

We need to add the `kubernetes` and `tls` providers because the EKS module uses them:

**File: `envs/dev/providers.tf`** — Replace the entire file:

```hcl
terraform {
  required_version = ">= 1.11.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 3.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Project   = "pharma"
      Env       = "dev"
      ManagedBy = "terraform"
    }
  }
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name]
  }
}
```

### Step 7: Update `envs/dev/variables.tf`

**File: `envs/dev/variables.tf`** — Replace the entire file:

```hcl
variable "db_password" {
  description = "Master password for the RDS PostgreSQL database"
  type        = string
  sensitive   = true
}

variable "jwt_secret" {
  description = "JWT signing secret for the application"
  type        = string
  sensitive   = true
}

variable "github_org" {
  description = "GitHub username or organization that owns frontend and backend"
  type        = string
  default     = "zenpharma"
}
```

### Step 8: Update `envs/dev/outputs.tf`

**File: `envs/dev/outputs.tf`** — Replace the entire file:

```hcl
output "rds_endpoint" {
  description = "RDS instance endpoint"
  value       = module.rds.db_instance_endpoint
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}
```

### Step 9: Commit and Push

```bash
cd ~/devops/zenpharma/infra
git add .
git commit -m "feat: add EKS, ECR, RDS, secrets-manager modules and wire dev environment"
git push origin main
```

> **Tag `infra` repo: `module-1.8-all-modules`**
> ```bash
> git tag -a module-1.8-all-modules -m "Module 1.8: All Terraform modules (VPC, EKS, ECR, RDS, IAM, Secrets Manager)"
> git push origin module-1.8-all-modules
> ```

---

## 1.9 Apply All Infrastructure and Verify

### Step 1: Re-initialize Terraform

Since we added new modules and providers, we need to re-run init:

```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform init
```

### Step 2: Run Terraform Plan

We need to pass the sensitive variables. Create a `terraform.tfvars` file — Terraform loads this file **automatically** (no extra flags needed):

```bash
cat > ~/devops/zenpharma/infra/envs/dev/terraform.tfvars << 'EOF'
db_password = "Passw0rd2025!Pharma"
jwt_secret  = "zen-pharma-jwt-s3cret-k3y-2025-xtra"
github_org  = "zenpharma"
github_org  = "zenpharma"
EOF
```

> **Why `terraform.tfvars`?** Terraform automatically loads files named exactly `terraform.tfvars` or `*.auto.tfvars` without needing a `-var-file` flag. Any other name (like `dev.tfvars`) requires you to pass `-var-file="dev.tfvars"` every time.

> **IMPORTANT:** This file contains secrets. It's already in `.gitignore` (`*.tfvars`) so it won't be committed. Never commit secrets to Git.

Now run plan — no flags needed:

```bash
terraform plan
```

Review the plan output. You should see a large number of resources to create.

### Step 3: Run Terraform Apply

```bash
terraform apply
```

Type `yes` when prompted.

**This will take 15-20 minutes** — EKS cluster creation takes the longest.

**Expected output (last few lines):**
```
Apply complete! Resources: XX added, 0 changed, 0 destroyed.

Outputs:

eks_cluster_name = "pharma-dev-cluster"
rds_endpoint = "pharma-dev-postgres.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com:5432"
```

### Step 4: Configure kubectl

```bash
aws eks update-kubeconfig --name pharma-dev-cluster --region us-east-1
```

**Expected output:**
```
Added new context arn:aws:eks:us-east-1:123456789012:cluster/pharma-dev-cluster to /Users/your-user/.kube/config
```

### Step 5: Verify Cluster Access

```bash
kubectl get nodes
```

**Expected output:**
```
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-3-xxx.ec2.internal    Ready    <none>   5m    v1.33.x
ip-10-0-4-xxx.ec2.internal    Ready    <none>   5m    v1.33.x
ip-10-0-3-yyy.ec2.internal    Ready    <none>   5m    v1.33.x
```

You should see 3 nodes (matching `desired_size = 3`).

### Step 6: Verify in AWS Console

**EKS:**
1. Go to https://console.aws.amazon.com/eks/
2. Click on `pharma-dev-cluster`
3. Check the **Compute** tab — you should see the managed node group with 3 nodes
4. Check the **Add-ons** tab — vpc-cni, kube-proxy, coredns should all be Active

**ECR:**
1. Go to https://console.aws.amazon.com/ecr/
2. Click **Private registry** → **Repositories**
3. You should see 9 repositories (api-gateway, auth-service, drug-catalog-service, etc.)

**RDS:**
1. Go to https://console.aws.amazon.com/rds/
2. Click **Databases**
3. You should see `pharma-dev-postgres` with status **Available**

**Secrets Manager:**
1. Go to https://console.aws.amazon.com/secretsmanager/
2. You should see 2 secrets:
   - `/pharma/dev/db-credentials`
   - `/pharma/dev/jwt-secret`

**IAM:**
1. Go to https://console.aws.amazon.com/iam/ → **Roles**
2. Search for `pharma-dev`
3. You should see roles: `pharma-dev-eso-role`, `pharma-dev-argocd-role`, `pharma-dev-alb-controller-role`, `pharma-dev-github-actions-role`

---

## 1.10 Destroy Infrastructure

At the end of Module 1, we have only run Terraform — no Helm charts, no ArgoCD, no Kubernetes workloads have been deployed yet. The destroy is straightforward.

### Step 1: Run Terraform Destroy

```bash
cd ~/devops/zenpharma/infra/envs/dev
terraform destroy
```

Type `yes` when prompted.

**This will take 10–15 minutes** — EKS clusters and NAT Gateways are the slowest to delete.

**Expected output:**
```
Destroy complete! Resources: XX destroyed.
```

### Step 2: Verify Cleanup

```bash
# Check no EKS clusters remain
aws eks list-clusters --region us-east-1

# Check no RDS instances remain
aws rds describe-db-instances --region us-east-1 \
  --query 'DBInstances[].DBInstanceIdentifier'

# Check no VPCs remain (besides the default VPC)
aws ec2 describe-vpcs --region us-east-1 \
  --query 'Vpcs[?Tags].VpcId'
```

All should return empty results.

> **Note:** In later modules (after Helm charts and ArgoCD are installed), the destroy process requires extra pre-cleanup steps. That is covered in Module 2.6.

> **End of Module 1.** Infrastructure is destroyed. In Module 2, we'll recreate it through GitHub Actions instead of running Terraform locally.

---

## Module 1 Summary

| What We Built | Details |
|--------------|---------|
| **AWS Credentials** | IAM user with programmatic access |
| **GitHub Setup** | Organization + 4 repos + SSH auth |
| **S3 Backend** | Versioned bucket for Terraform state |
| **VPC Module** | 2 AZs, public/private/database subnets, NAT Gateway |
| **EKS Module** | Managed Kubernetes cluster with 3 nodes |
| **ECR Module** | 9 container image repositories |
| **RDS Module** | PostgreSQL database in private subnet |
| **IAM Module** | 4 IRSA roles + GitHub Actions OIDC federation |
| **Secrets Manager Module** | DB credentials + JWT secret |

| Tag | Repos |
|-----|-------|
| `module-1.2-initial-code` | infra, frontend, backend |
| `module-1.5-vpc-module` | infra |
| `module-1.7-iam-module` | infra |
| `module-1.8-all-modules` | infra |

> **Next:** [Module 2 — Automating Terraform with GitHub Actions](MODULE-2-AUTOMATING-TERRAFORM-WITH-GITHUB-ACTIONS.md)
