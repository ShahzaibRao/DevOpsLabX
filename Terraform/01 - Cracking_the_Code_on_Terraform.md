# 1.1 : What is Terraform? — *Why Infrastructure as Code?*

---

⏱️ **Skip if**: You already know *IaC*, *Terraform workflow*, and *pain points of manual infra*.

---

## 🧠 Big Picture: Why Do We Even Need Terraform?

> ❓ “Why write code when I can just click in the AWS Console?”
> 

Let’s walk through a **real-world scenario** 🚨:

### ⚙️ Scenario: 3-Tier App × 6 Environments × 100+ Apps

You need to deploy:

- Web tier (load balancer + EC2 + CDN)
- App tier (ASG + internal LB)
- DB tier (RDS + replicas)
- Networking (VPC, subnets, SGs, Route 53)
- Monitoring, logging, IAM...

✅ **Manually in AWS Console** per environment: ~2 hours

🔁 For **6 environments** (dev, test, staging, prod, DR, perf): **12 hours/app**

🏢 For **100 apps**: **1,200 hours** ⏳ → *That’s 6 months of full-time work!*

> 🔥 This is why manual provisioning doesn’t scale — and defeats the purpose of using the cloud.
> 

---

## 📉 Problems with Manual Infrastructure

| Problem | Description |
| --- | --- |
| ⏳ **Time-Consuming** | Repetitive, slow, blocks dev/QA teams |
| 👥 **Human Error** | Typos, missed security settings (e.g., open S3 bucket), inconsistent configs |
| 🔄 **Inconsistency** | *“It works on my machine!”* → Dev ≠ Staging ≠ Prod |
| 💸 **Costly** | Need large infra teams just for clicks; hard to clean up |
| 🔐 **Insecure** | No audit trail; anyone with console access can change anything |
| 🚫 **No Versioning** | No history, no rollback, no blameless post-mortems |

> ✅ All these vanish with Infrastructure as Code (IaC).
> 

---

## ✅ What is *Infrastructure as Code* (IaC)?

> 📜 Definition: Managing and provisioning infrastructure through machine-readable definition files — not manual processes.
> 

💡 Think:

> “Git for your cloud”
> 
> 
> *“Automate the boring stuff — reliably.”*
> 

### 🔧 IaC Tools Overview

| Tool | Type | Cloud Support | Notes |
| --- | --- | --- | --- |
| **Terraform** | Universal | ✅ AWS, Azure, GCP, Kubernetes, on-prem, **2000+ providers** | **Most popular**, declarative, stateful |
| Pulumi | Universal | ✅ Multi-cloud | Uses real languages (Python, TS) |
| AWS CloudFormation | Vendor-Locked | ❌ AWS only | Template-based (JSON/YAML) |
| Azure ARM / Bicep | Vendor-Locked | ❌ Azure only | Native but less portable |
| GCP Deployment Manager | Vendor-Locked | ❌ GCP only | Deprecated in favor of Config Connector |

> 🎯 Why Terraform?
> 
> 
> ✔️ **Cloud-agnostic** → reusable skills
> 
> ✔️ **Declarative** → you say *what*, not *how*
> 
> ✔️ **Strong ecosystem** (providers, modules, community)
> 

---

## 🔄 How Terraform *Actually* Works (High-Level)

```mermaid
graph LR
A[DevOps Engineer] -->|Writes| B[main.tf, variables.tf, etc.]
B -->|Stored in| C[Git (GitHub/GitLab)]
C -->|Triggers| D[Terraform CLI or CI/CD]
D -->|Runs| E[terraform init]
D -->|Runs| F[terraform plan]
D -->|Runs| G[terraform apply]
G -->|Calls| H[AWS/Azure/GCP APIs via Provider]
H -->|Creates| I[Real Cloud Resources]

```

### 🛠️ Core Commands (The “IaC Lifecycle”)

| Command | Purpose | Safety Level |
| --- | --- | --- |
| `terraform init` | Downloads providers & modules | ✅ Safe |
| `terraform validate` | Checks syntax & config errors | ✅ Safe |
| `terraform plan` | **Dry-run**: shows *what* will change (✅/⚠️/❌) | ✅ Safe *(READ-ONLY)* |
| `terraform apply` | **Executes** the plan — creates/modifies infrastructure | ⚠️ Changes real resources! |
| `terraform destroy` | Tears down *all* managed resources | 🔥 Destructive — use carefully! |

> 🔑 Key Insight:
> 
> 
> Terraform **doesn’t bypass APIs** — it *uses* them (just like AWS Console or `awscli`).
> 
> → No magic. Just automation + consistency.
> 

---

## 🧩 Terraform File Basics

- **File extension**: `.tf` (e.g., `main.tf`, `network.tf`)
- **Language**: **HCL** (HashiCorp Configuration Language)
    - Human-readable ✅
    - Structured like JSON, but cleaner & more expressive
    - *Not a programming language* — it’s *declarative config*

### Example: Minimal S3 Bucket

```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-12345"
}

```

> 🧪 We’ll write & run this in the next lab!
> 

---

## 🧪 Lab: Let’s Get Terraform Ready! ✅

### 🖥️ Step 1: Install Terraform

| OS | Command |
| --- | --- |
| **macOS (Homebrew)** | `brew install hashicorp/tap/terraform` |
| **Linux (apt)** | `sudo apt install terraform` |
| **Windows (Chocolatey)** | `choco install terraform` |

✅ Verify:

```bash
terraform version
# Output: Terraform v1.13.3 (or newer)

```

### 🧰 Step 2: VS Code Setup

1. Install extension: **“HashiCorp Terraform”** (by HashiCorp)
2. Optional but helpful:
    - Set alias: `alias tf=terraform` (add to `~/.bashrc` or `~/.zshrc`)
    - Enable autocomplete: `terraform -install-autocomplete`

> 💡 Pro Tip: Always use version-controlled .tf files — never run apply from random folders!
> 

---

## ❓ FAQ (Anticipating Your Questions)

**Q: Is Terraform a programming language?**

A: ❌ No! It’s *declarative configuration*. You define *desired state* — Terraform figures out how to get there.

**Q: Can I use JSON instead of HCL?**

A: ✅ Yes — `.tf.json` files are valid. But HCL is preferred (more readable, comments, etc.).

**Q: What if two people run `apply` at the same time?**

A: 👉 **State locking** (via remote backends like S3 + DynamoDB) prevents conflicts — we’ll cover this soon!

**Q: Does Terraform replace Kubernetes or Ansible?**

A: 🤝 **No — it complements them**:

- **Terraform** = *Provision infrastructure* (VMs, VPCs, DBs)
- **Ansible** = *Configure software* on VMs (install nginx, users, etc.)
- **Kubernetes** = *Orchestrate containers* (on infra Terraform created)

---

## ➡️ Summary: Why Terraform Wins

| Benefit | Impact |
| --- | --- |
| ✅ **Speed** | Provision 100 envs in minutes, not months |
| ✅ **Consistency** | Dev = Staging = Prod (no more “works on my machine”) |
| ✅ **Auditability** | Every change tracked in Git — who, what, when |
| ✅ **Cost Control** | Spin up/down non-prod envs on-demand (`terraform destroy`) |
| ✅ **DRY Principle** | Write once → deploy everywhere (modules, workspaces) |
| ✅ **Collaboration** | Infra becomes *code reviewable* — no more solo console heroes |

> 🎯 Bottom Line:
> 
> 
> **Terraform shifts infrastructure from *art* → *engineering***.
> 

---