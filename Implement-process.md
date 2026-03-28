# 🚀 Automating EKS Cluster Creation with Jenkins + Terraform + AWS

> A complete, production-ready guide to provisioning an Amazon EKS (Elastic Kubernetes Service) cluster using a fully automated Jenkins CI/CD pipeline backed by Terraform Infrastructure-as-Code.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture & Workflow](#architecture--workflow)
- [Tech Stack](#tech-stack)
- [Phase 1 — VM Setup & Tool Installation](#phase-1--vm-setup--tool-installation)
- [Phase 2 — IAM User & AWS Access](#phase-2--iam-user--aws-access)
- [Phase 3 — Jenkins Configuration](#phase-3--jenkins-configuration)
- [Phase 4 — Writing Terraform Files](#phase-4--writing-terraform-files)
- [Phase 5 — Jenkins Pipeline for EKS Automation](#phase-5--jenkins-pipeline-for-eks-automation)
- [Verification & Testing](#verification--testing)
- [Teardown & Cleanup](#teardown--cleanup)
- [Quick Reference — All Commands](#quick-reference--all-commands)

---

## Project Overview

This project automates the **end-to-end creation of a production-grade Kubernetes cluster on AWS** using:

- **Jenkins** as the CI/CD orchestrator
- **Terraform** as the Infrastructure-as-Code (IaC) tool
- **AWS EKS** as the managed Kubernetes control plane

Once set up, you can spin up or destroy an entire EKS cluster — including VPC, subnets, NAT gateway, node groups, and Kubernetes add-ons — with a **single Jenkins pipeline run**. No manual AWS Console clicks required.

### What Gets Created

| Resource | Details |
|---|---|
| VPC | CIDR `10.123.0.0/16`, 2 Availability Zones |
| Subnets | Public (load balancers), Private (worker nodes), Intra (control plane) |
| NAT Gateway | Enables private nodes to reach the internet |
| EKS Cluster | `watches-eks-cluster`, region `ap-south-1` |
| Node Group | `t3.small` SPOT instances, min 1 / max 2 |
| Add-ons | CoreDNS, kube-proxy, VPC CNI (all latest) |

---

## Architecture & Workflow

```
Developer / Operator
        │
        ▼
  Jenkins Pipeline  ←──── GitHub (Terraform files)
        │
        │  terraform init → validate → plan → apply
        ▼
   AWS (ap-south-1)
   ┌─────────────────────────────────────┐
   │  VPC  10.123.0.0/16                 │
   │  ┌──────────┐  ┌──────────────────┐ │
   │  │  Public  │  │  Private Subnets  │ │
   │  │ Subnets  │  │  (Worker Nodes)   │ │
   │  └──────────┘  └──────────────────┘ │
   │         NAT Gateway                  │
   │  ┌───────────────────────────────┐   │
   │  │     EKS Control Plane         │   │
   │  │   (Intra Subnets)             │   │
   │  └───────────────────────────────┘   │
   └─────────────────────────────────────┘
```

**High-Level Flow:**

```
Launch EC2
    → Install Java, Jenkins, Terraform, kubectl, AWS CLI
        → Create IAM User (EKS-user)
            → Store credentials in Jenkins
                → Write Terraform .tf files
                    → Push to GitHub
                        → Run Jenkins Pipeline
                            → EKS Cluster is Live ✅
```

---

## Tech Stack

| Tool | Version / Type | Purpose |
|---|---|---|
| AWS EC2 | m7.large, Ubuntu AMI | Host machine for Jenkins |
| Java (Temurin) | JDK 17 | Required runtime for Jenkins |
| Jenkins | LTS (Debian stable) | CI/CD pipeline orchestrator |
| Terraform | Latest (HashiCorp) | Infrastructure as Code |
| kubectl | Latest stable | Kubernetes CLI |
| AWS CLI | v2 | Authenticate & interact with AWS |
| AWS EKS | Managed K8s | Kubernetes control plane |
| AWS VPC Module | `~> 4.0` | Terraform community VPC module |
| AWS EKS Module | `19.15.1` | Terraform community EKS module |

---

## Phase 1 — VM Setup & Tool Installation

### Why this phase?

The EC2 instance acts as the **automation control server**. Jenkins runs here and executes all Terraform commands. Every tool — Java, Terraform, kubectl, and AWS CLI — must be pre-installed so the pipeline never has external dependency failures at runtime.

---

### Step 1.1 — Launch EC2 Instance

Go to AWS Console → EC2 → Launch Instance:

- **AMI:** Ubuntu (22.04 LTS recommended)
- **Instance type:** `m7.large`
- **Security group:** Open port `8080` (Jenkins UI) and port `22` (SSH)

---

### Step 1.2 — Install Java & Jenkins

Create the installation script:

```bash
vi jenkins.sh
```

Paste the following content:

```bash
#!/bin/bash
sudo apt update -y

# Install Temurin JDK 17
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Run the script:

```bash
sudo chmod +x jenkins.sh
./jenkins.sh
```

> **⚠️ Fallback:** If the above gives a GPG key error, use these commands instead:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

**Access Jenkins UI:**

Open your browser and go to:
```
http://<EC2-PUBLIC-IP>:8080
```

Get the initial admin password:
```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy the output, paste it into the Jenkins UI, and set your own credentials.

---

### Step 1.3 — Install Terraform

```bash
vi terraform.sh
```

```bash
#!/bin/bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

gpg --no-default-keyring \
  --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
  --fingerprint

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt-get install terraform -y
```

```bash
sudo chmod +x terraform.sh
./terraform.sh

# Verify installation
terraform -version
```

---

### Step 1.4 — Install kubectl

```bash
vi k8s.sh
```

```bash
#!/bin/bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

```bash
sudo chmod +x k8s.sh
./k8s.sh
```

---

### Step 1.5 — Install AWS CLI

```bash
vi awscli.sh
```

```bash
#!/bin/bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

```bash
sudo chmod +x awscli.sh
./awscli.sh

# Verify
aws --version
```

---

### ✅ Phase 1 Summary

By the end of this phase you have a fully equipped EC2 instance running:

- Jenkins (accessible on port 8080)
- Terraform (IaC engine)
- kubectl (K8s CLI)
- AWS CLI (cloud authentication)

---

## Phase 2 — IAM User & AWS Access

### Why this phase?

Terraform needs AWS credentials to create and manage cloud resources. Using root account credentials is a serious security risk. Instead, a **dedicated IAM user** (`EKS-user`) is created with only the access needed, and its **programmatic Access Key + Secret Key** are used to authenticate Terraform via the Jenkins pipeline.

---

### Step 2.1 — Create IAM User in AWS Console

Navigate to: **AWS Console → IAM → Users → Create User**

| Field | Value |
|---|---|
| Username | `EKS-user` |
| Access type | Programmatic access |
| Permissions | Attach `AdministratorAccess` policy directly |

### Step 2.2 — Generate Access Keys

After creating the user:

1. Go to the user → **Security credentials** tab
2. Click **Create access key** → choose **CLI** use case
3. Copy and save both values securely:

```
Access Key ID     : (your generated key)
Secret Access Key : (your generated secret — shown ONCE only)
```

> **⚠️ Security Note:** Never commit these credentials to any Git repository. They will be stored securely in Jenkins in the next phase.

---

### ✅ Phase 2 Summary

A non-root IAM user is created with programmatic access. Its credentials will be injected into the Jenkins pipeline as encrypted secrets — never hardcoded anywhere in the codebase.

---

## Phase 3 — Jenkins Configuration

### Why this phase?

Jenkins needs to pass AWS credentials to Terraform at pipeline runtime. Jenkins **Credentials Manager** stores them encrypted on disk and injects them as environment variables — a secure alternative to hardcoding secrets in `Jenkinsfile` or `.tf` files.

---

### Step 3.1 — Install Required Plugin

Go to: **Jenkins Dashboard → Manage Jenkins → Plugins → Available plugins**

Search for and install:
```
Pipeline Stage View
```

---

### Step 3.2 — Add AWS Access Key as Jenkins Credential

Navigate to:
**Dashboard → Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Field | Value |
|---|---|
| Kind | Secret text |
| Secret | *(paste your AWS Access Key ID)* |
| ID | `AWS_Access_Key` |
| Description | AWS Access Key for EKS provisioning |

---

### Step 3.3 — Add AWS Secret Key as Jenkins Credential

Repeat the same steps:

| Field | Value |
|---|---|
| Kind | Secret text |
| Secret | *(paste your AWS Secret Access Key)* |
| ID | `AWS_Secret_Key` |
| Description | AWS Secret Key for EKS provisioning |

---

### ✅ Phase 3 Summary

Jenkins now securely holds both AWS credentials under the IDs `AWS_Access_Key` and `AWS_Secret_Key`. The pipeline will reference these IDs and Jenkins will inject the actual values at runtime — your keys never appear in any code file.

---

## Phase 4 — Writing Terraform Files

### Why this phase?

Terraform `.tf` files **declare the desired state** of your AWS infrastructure. Instead of clicking through the AWS Console, you describe what you want — and Terraform figures out how to create it. The files are split into three focused modules for clarity and reusability.

---

### File Structure

```
terraform/
├── provider.tf    ← Shared local values + AWS provider config
├── vpc.tf         ← VPC, subnets, NAT gateway
└── eks.tf         ← EKS cluster + managed node groups + add-ons
```

---

### Step 4.1 — `provider.tf`

Defines shared local variables (region, cluster name, CIDR ranges) used by all modules, and configures the AWS provider.

```hcl
locals {
  region          = "ap-south-1"
  name            = "watches-eks-cluster"
  vpc_cidr        = "10.123.0.0/16"
  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.123.1.0/24", "10.123.2.0/24"]
  private_subnets = ["10.123.3.0/24", "10.123.4.0/24"]
  intra_subnets   = ["10.123.5.0/24", "10.123.6.0/24"]
  tags = {
    Example = local.name
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

---

### Step 4.2 — `vpc.tf`

Creates a production-grade VPC with three subnet tiers across 2 Availability Zones:

- **Public subnets** — for internet-facing load balancers
- **Private subnets** — for EKS worker nodes (no direct internet exposure)
- **Intra subnets** — for the EKS control plane ENIs
- **NAT Gateway** — allows private nodes to pull container images from the internet

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 4.0"

  name = local.name
  cidr = local.vpc_cidr

  azs             = local.azs
  private_subnets = local.private_subnets
  public_subnets  = local.public_subnets
  intra_subnets   = local.intra_subnets

  enable_nat_gateway = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

> **Note:** The subnet tags (`kubernetes.io/role/elb` and `kubernetes.io/role/internal-elb`) are required for Kubernetes to auto-discover which subnets to use for load balancer provisioning.

---

### Step 4.3 — `eks.tf`

Provisions the EKS cluster with:

- Public API endpoint access
- Three managed Kubernetes add-ons (CoreDNS, kube-proxy, VPC CNI)
- A SPOT-based managed node group (`t3.small`) for cost efficiency

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.1"

  cluster_name                   = local.name
  cluster_endpoint_public_access = true

  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
  }

  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.intra_subnets

  eks_managed_node_group_defaults = {
    ami_type       = "AL2_x86_64"
    instance_types = ["m7i-flex.large"]
    attach_cluster_primary_security_group = true
  }

  eks_managed_node_groups = {
    ok-cluster-wg = {
      min_size     = 1
      max_size     = 2
      desired_size = 1

      instance_types = ["t3.small"]
      capacity_type  = "SPOT"

      tags = {
        description = "helloworld"
      }
    }
  }

  tags = local.tags
}
```

---

### ✅ Phase 4 Summary

Three Terraform files now declare the complete desired state of your infrastructure. No clicking — the entire stack (VPC, networking, EKS control plane, node groups, add-ons) is described in code and is version-controllable, reproducible, and reviewable.

---

## Phase 5 — Jenkins Pipeline for EKS Automation

### Why this phase?

The Jenkins pipeline **connects everything**: it pulls Terraform files from GitHub, authenticates with AWS using the stored credentials, and runs Terraform commands in sequence with a **manual approval gate** before provisioning. The `$action` parameter makes the same pipeline serve both creation (`apply`) and destruction (`destroy`) — no separate pipelines needed.

---

### Step 5.1 — Push Terraform Files to GitHub

```bash
git init
git add .
git commit -m "Add EKS terraform files"
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

---

### Step 5.2 — Create Jenkins Pipeline Job

1. Jenkins Dashboard → **New Item**
2. Enter a name (e.g., `EKS-Cluster-Pipeline`)
3. Select **Pipeline** → click **OK**
4. Under **General**, check ✅ **This project is parameterized**
5. Click **Add Parameter** → **Choice Parameter**:

| Field | Value |
|---|---|
| Name | `action` |
| Choices | `apply` *(line 1)* `destroy` *(line 2)* |

---

### Step 5.3 — Pipeline Script

Paste the following into the **Pipeline Script** section:

```groovy
pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_Access_Key')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_Secret_Key')
        AWS_DEFAULT_REGION    = 'ap-south-1'
    }
    stages {
        stage('Clone the Code') {
            steps {
                script {
                    checkout scmGit(
                        branches: [[name: '*/main']],
                        extensions: [],
                        userRemoteConfigs: [[url: 'https://github.com/<your-username>/<your-repo>.git']]
                    )
                }
            }
        }
        stage('Terraform Initialization') {
            steps {
                script {
                    dir('terraform') {
                        sh 'terraform init'
                    }
                }
            }
        }
        stage('Terraform Validation') {
            steps {
                script {
                    dir('terraform') {
                        sh 'terraform validate'
                    }
                }
            }
        }
        stage('Infrastructure Checks') {
            steps {
                script {
                    dir('terraform') {
                        sh 'terraform plan'
                    }
                    input(message: "Approve?", ok: "proceed")
                }
            }
        }
        stage('Create/Destroy EKS cluster') {
            steps {
                script {
                    dir('terraform') {
                        sh 'terraform $action --auto-approve'
                    }
                }
            }
        }
    }
}
```

### Pipeline Stage Breakdown

| Stage | What it does |
|---|---|
| **Clone the Code** | Checks out Terraform files from the GitHub repository |
| **Terraform Initialization** | Downloads required providers and modules |
| **Terraform Validation** | Checks `.tf` files for syntax errors |
| **Infrastructure Checks** | Runs `terraform plan` — shows what WILL be created/destroyed. Pauses for manual approval. |
| **Create/Destroy EKS cluster** | Runs `terraform apply` or `terraform destroy` based on the `$action` parameter |

---

### Step 5.4 — Run the Pipeline

1. Click **Build with Parameters**
2. Select `apply` → click **Build**
3. Watch the stages progress in Stage View
4. At the **Infrastructure Checks** stage, review the plan and click **Proceed** to approve
5. Wait for provisioning to complete (~10–15 minutes for a full EKS cluster)

---

### ✅ Phase 5 Summary

The Jenkins pipeline is a fully parameterized, gated automation. It pulls code from GitHub, validates infrastructure changes, waits for human approval, then provisions or destroys the entire EKS stack on AWS — completely hands-free after the initial trigger.

---

## Verification & Testing

### Step 6.1 — Verify in AWS Console

Go to: **AWS Console → EKS → Clusters → watches-eks-cluster**

Check the following tabs:

| Tab | What to look for |
|---|---|
| **Resources** | Nodes, deployments, pods showing up |
| **Add-ons** | CoreDNS, kube-proxy, vpc-cni all in ACTIVE state |
| **Compute** | Node group `ok-cluster-wg` with running instances |

---

### Step 6.2 — Connect Your Local Machine to the Cluster

Make sure AWS CLI is configured with the `EKS-user` credentials:

```bash
aws configure
# Enter: Access Key ID, Secret Access Key, region: ap-south-1, output: json
```

Update your local kubeconfig to point to the new cluster:

```bash
aws eks update-kubeconfig --region ap-south-1 --name watches-eks-cluster
```

---

### Step 6.3 — Run a Test Pod

```bash
# Create a test nginx pod
kubectl run nginx --image=nginx

# Verify the pod is running
kubectl get pods
```

Expected output:
```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          30s
```

If you see this, your EKS cluster is fully operational. 🎉

---

## Teardown & Cleanup

When you're done, destroy all resources to avoid AWS charges:

### Step 1 — Destroy via Jenkins

1. Jenkins → Pipeline job → **Build with Parameters**
2. Select `destroy` → click **Build**
3. Approve the plan when prompted
4. Wait for all resources to be deleted

### Step 2 — Terminate the EC2 Instance

Go to **AWS Console → EC2 → Instances** → select the Jenkins VM → **Terminate instance**.

> **⚠️ Important:** Always destroy resources when not in use. An EKS cluster + NAT gateway left running can cost significantly on AWS.

---

## Quick Reference — All Commands

### Phase 1 — Tool Installation

```bash
# Run each script after creating and chmod-ing it:
sudo chmod +x jenkins.sh   && ./jenkins.sh
sudo chmod +x terraform.sh && ./terraform.sh
sudo chmod +x k8s.sh       && ./k8s.sh
sudo chmod +x awscli.sh    && ./awscli.sh

# Get Jenkins initial password
cat /var/lib/jenkins/secrets/initialAdminPassword

# Verify installs
java --version
terraform -version
kubectl version --client
aws --version
```

### Phase 5 — Git

```bash
git init
git add .
git commit -m "Add EKS terraform files"
git push -u origin main
```

### Phase 6 — Cluster Access

```bash
aws configure
aws eks update-kubeconfig --region ap-south-1 --name watches-eks-cluster
kubectl run nginx --image=nginx
kubectl get pods
kubectl get nodes
```

### Teardown

```bash
# Via Jenkins pipeline: Build with Parameters → action: destroy
# Then terminate EC2 instance from AWS Console
```

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Jenkins GPG key error during install | Use fallback `jenkins.io-2026.key` commands in Phase 1 |
| Pipeline fails at `terraform init` | Check GitHub repo URL and branch name in the pipeline script |
| AWS credentials error in pipeline | Verify credential IDs in Jenkins match exactly: `AWS_Access_Key`, `AWS_Secret_Key` |
| `kubectl` can't connect to cluster | Re-run `aws eks update-kubeconfig` with correct region and cluster name |
| EKS node group stuck in `Creating` | Check IAM permissions — `EKS-user` needs `AdministratorAccess` |

---

## Project Author Notes

- **Region used:** `ap-south-1` (Mumbai) — change `locals` in `provider.tf` for a different region
- **Node type:** `t3.small` SPOT — cheap for learning; change to `t3.medium` or On-Demand for production
- **Cluster name:** `watches-eks-cluster` — customizable in `provider.tf` locals block
- **EKS module version:** `19.15.1` — pinned for stability; check for newer versions before production use
