# Module 0 — Prerequisites

> Everything you need to set up before starting the course.
> Estimated time: 30–45 minutes.

---

## 0.1 Accounts Required

You need **two accounts** before starting this course. Both are free to create.

### AWS Account

1. Go to https://aws.amazon.com/ and click **Create an AWS Account**
2. Enter your email, choose an account name (e.g., `ZenPharma-DevOps`)
3. Complete the sign-up process (requires a credit card — you will incur costs for EKS, RDS, etc.)
4. Sign in to the **AWS Management Console** at https://console.aws.amazon.com/

> **Cost Warning:** This course provisions real AWS resources (EKS cluster, RDS database, NAT Gateway, ALBs). Expect approximately **$5–10/day** when infrastructure is running. Always destroy resources when not actively working to minimize costs.

### GitHub Account

1. Go to https://github.com/ and click **Sign up**
2. Choose a free plan
3. Verify your email address

---

## 0.2 Install Required Tools

### AWS CLI v2

The AWS CLI lets you interact with AWS services from your terminal.

**macOS:**
```bash
# Option 1: Homebrew
brew install awscli

# Option 2: Official installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux (Ubuntu/Debian):**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows (PowerShell as Administrator):**
```powershell
# Option 1: MSI installer (download and run)
# Download from: https://awscli.amazonaws.com/AWSCLIV2.msi
# Double-click the downloaded file and follow the wizard

# Option 2: winget
winget install Amazon.AWSCLI

# Option 3: Chocolatey
choco install awscli
```

**Verify:**
```bash
aws --version
# Expected: aws-cli/2.x.x ...
```

---

### Terraform (>= 1.11)

Terraform is the Infrastructure as Code tool we use to provision AWS resources.

**macOS:**
```bash
brew install terraform
```

**Linux (Ubuntu/Debian):**
```bash
# Add HashiCorp GPG key and repository
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update && sudo apt-get install terraform
```

**Windows:**
```powershell
# Option 1: Chocolatey
choco install terraform

# Option 2: winget
winget install HashiCorp.Terraform
```

**Verify:**
```bash
terraform --version
# Expected: Terraform v1.11.x or higher
```

---

### kubectl

kubectl is the Kubernetes command-line tool. You use it to run commands against your EKS cluster.

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
# Download the latest stable release
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

**Windows:**
```powershell
# Option 1: Chocolatey
choco install kubernetes-cli

# Option 2: winget
winget install Kubernetes.kubectl

# Option 3: Direct download
curl.exe -LO "https://dl.k8s.io/release/v1.33.0/bin/windows/amd64/kubectl.exe"
# Move kubectl.exe to a directory in your PATH
```

**Verify:**
```bash
kubectl version --client
# Expected: Client Version: v1.3x.x
```

---

### Helm (v3)

Helm is the Kubernetes package manager. We use it to install ArgoCD, AWS Load Balancer Controller, and External Secrets Operator.

**macOS:**
```bash
brew install helm
```

**Linux:**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Windows:**
```powershell
# Option 1: Chocolatey
choco install kubernetes-helm

# Option 2: winget
winget install Helm.Helm
```

**Verify:**
```bash
helm version
# Expected: version.BuildInfo{Version:"v3.x.x", ...}
```

---

### Git

Git is the version control system. You need it to clone repositories and push code.

**macOS:**
```bash
# Git comes pre-installed on macOS. If not:
brew install git
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update && sudo apt-get install -y git
```

**Windows:**
```powershell
# Option 1: Download installer from https://git-scm.com/download/win
# Option 2: winget
winget install Git.Git
```

**Verify:**
```bash
git --version
# Expected: git version 2.x.x
```

**Initial Git configuration (all platforms):**
```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

---

### Python 3

Python is used to run the bootstrap scripts that install ArgoCD, External Secrets Operator, and deploy services to the EKS cluster.

**macOS:**
```bash
# Python 3 comes pre-installed on modern macOS. If not:
brew install python3
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update && sudo apt-get install -y python3 python3-pip
```

**Windows:**
```powershell
# Option 1: Download from https://www.python.org/downloads/
# IMPORTANT: Check "Add Python to PATH" during installation

# Option 2: winget
winget install Python.Python.3.12
```

**Verify:**
```bash
python3 --version
# Expected: Python 3.x.x
```

---

### GitHub CLI (`gh`)

The GitHub CLI lets you create repos, manage PRs, and trigger workflows from the terminal.

**macOS:**
```bash
brew install gh
```

**Linux (Ubuntu/Debian):**
```bash
# Add GitHub CLI repository
(type -p wget >/dev/null || (sudo apt update && sudo apt-get install wget -y)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update \
  && sudo apt install gh -y
```

**Windows:**
```powershell
# Option 1: winget
winget install GitHub.cli

# Option 2: Chocolatey
choco install gh
```

**Verify:**
```bash
gh --version
# Expected: gh version 2.x.x

# Authenticate with your GitHub account
gh auth login
# Select: GitHub.com → HTTPS → Yes (authenticate with browser) → Login with browser
```

---

## 0.3 Install VS Code and Extensions

### Install VS Code

**macOS:**
1. Go to https://code.visualstudio.com/
2. Download the macOS version
3. Open the `.zip` file — drag **Visual Studio Code** to Applications
4. Launch VS Code

**Linux (Ubuntu/Debian):**
```bash
# Option 1: Snap
sudo snap install --classic code

# Option 2: .deb package
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] \
  https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list > /dev/null
sudo apt update && sudo apt install code
```

**Windows:**
1. Go to https://code.visualstudio.com/
2. Download the Windows installer
3. Run the installer — check "Add to PATH" during installation

### Install Recommended Extensions

Open VS Code, then press `Cmd+Shift+X` (macOS) or `Ctrl+Shift+X` (Linux/Windows) to open the Extensions panel. Search for and install each:

| Extension | Publisher | What It Does |
|-----------|-----------|-------------|
| **HashiCorp Terraform** | HashiCorp | Syntax highlighting, autocomplete, and validation for `.tf` files |
| **Kubernetes** | Microsoft | Cluster explorer, manifest validation, IntelliSense for K8s YAML |
| **YAML** | Red Hat | YAML language support with schema validation |
| **Docker** | Microsoft | Dockerfile syntax, image management, container explorer |
| **GitLens** | GitKraken | Rich Git history, blame annotations, file history |

**Or install all at once from the terminal:**
```bash
code --install-extension hashicorp.terraform
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
code --install-extension redhat.vscode-yaml
code --install-extension ms-azuretools.vscode-docker
code --install-extension eamodio.gitlens
```

---

## 0.4 Create Project Directory

Create a working directory where all course repositories will live.

```bash
mkdir -p ~/devops/zenpharma
cd ~/devops/zenpharma
```

This directory will eventually contain 4 folders:
```
~/devops/zenpharma/
├── infra/          # Terraform code (Module 1)
├── frontend/       # Pharma-UI React app (Module 4)
├── backend/        # 8 microservices (Module 7)
└── gitops/         # Helm charts + ArgoCD apps (Module 5)
```

---

## 0.5 Final Verification

Run all checks at once to make sure everything is installed:

```bash
echo "=== Required Tools ==="
echo -n "AWS CLI:    " && aws --version 2>&1 | head -1
echo -n "Terraform:  " && terraform --version 2>&1 | head -1
echo -n "kubectl:    " && kubectl version --client 2>&1 | head -1
echo -n "Helm:       " && helm version --short 2>&1
echo -n "Git:        " && git --version
echo -n "Python:     " && python3 --version
echo -n "GitHub CLI: " && gh --version 2>&1 | head -1
```

**Expected output:** All tools should show version numbers without errors.

> **Why no Docker, Node.js, Java, or Maven?** All application builds happen in GitHub Actions (CI), not on your laptop. GitHub Actions runners come pre-installed with Docker, Node.js, Java, and Maven. You only need the infrastructure and cluster management tools locally.

---

## Summary

| Category | Item | Status |
|----------|------|--------|
| **Accounts** | AWS Account | |
| | GitHub Account | |
| **Required Tools** | AWS CLI v2 | |
| | Terraform >= 1.11 | |
| | kubectl | |
| | Helm v3 | |
| | Git | |
| | Python 3 | |
| | GitHub CLI (`gh`) | |
| **Editor** | VS Code + Extensions | |
| **Directory** | `~/devops/zenpharma` created | |

> **Next:** [Module 1 — Infrastructure as Code with Terraform](MODULE-1-INFRASTRUCTURE-AS-CODE-WITH-TERRAFORM.md)
