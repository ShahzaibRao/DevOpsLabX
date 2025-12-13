# **1.5 : Terraform Variables — Input, Locals, Output & Precedence Rules**

> 🔗 Repo: github.com/Push/terraform-aws-labs/day05
> 
> 
> 🎯 **Goal**: Replace hardcoded values with **reusable, environment-aware configs** — no more copy-paste errors!
> 

---

## 🧠 Why Variables? The *Real* Pain Point

Imagine this in a real team:

```hcl
# ❌ Disaster waiting to happen
tags = { Environment = "dev" }   # S3
tags = { Environment = "stg" }   # VPC ← typo!
tags = { Environment = "dev" }   # EC2

```

→ Your “dev” environment has a staging VPC → failed deployments, broken pipelines, and 3 AM pages.

✅ **Variables fix this** by letting you define *once*, reuse *everywhere*, change in *one place*.

---

## 📦 The 3 Variable Types (By Purpose)

| Type | Analogy | Scope | When to Use |
| --- | --- | --- | --- |
| **`variable`** (Input) | Function *parameters* | Configurable input | `env`, `region`, `instance_type` |
| **`locals`** | Local *variables* (`let x = …`) | Computed, internal-only | `bucket_name = "${var.app}-${var.env}-logs"` |
| **`output`** | Function *return values* | Exported results | `vpc_id`, `bucket_arn`, `alb_dns` |

> 💡 Golden Rule:
> 
> - **Input** → *What you control*
> - **Locals** → *What you compute*
> - **Output** → *What you share*

---

## ✏️ Hands-On: From Hardcoded → Parameterized

Let’s refactor a multi-resource config (S3 + VPC + EC2) using all 3 variable types.

### 📁 File Structure (Clean & Organized)

```
day05/
├── main.tf          # Resources
├── variables.tf     # Input variables (env, region)
├── locals.tf        # Computed values (names, tags)
├── outputs.tf       # Results to expose
├── terraform.tfvars # Default values (dev)
└── dev.tfvars       # Dev overrides

```

---

### 1️⃣ `variables.tf` — Input Variables (User-Provided)

```hcl
# 🔹 variables.tf
variable "env" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.env)
    error_message = "env must be 'dev', 'staging', or 'prod'."
  }
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "app" {
  description = "Application name"
  type        = string
  default     = "tech-tutorials-push"
}

```

✅ **Best Practice**:

- Always add `description`
- Use `validation` for safety
- Avoid `type = any` in production

---

### 2️⃣ `locals.tf` — Local Variables (Computed Once)

```hcl
# 🔹 locals.tf
locals {
  # Reusable computed name
  bucket_name = "${var.app}-${var.env}-logs"
  vpc_name    = "${var.env}-vpc"

  # Consistent tags (DRY!)
  common_tags = {
    Environment = var.env
    Application = var.app
    Owner       = "terraform-team"
  }
}

```

✅ **Why Locals?**

- No more repeating `"${var.app}-${var.env}-..."` 10×
- Change `app` → *all* names update automatically
- Centralized logic → fewer bugs

> 💡 Syntax Tip:
> 
> 
> Inside strings, **always use `"${var.env}"`** — not `"var.env"` (that’s a literal string!).
> 

---

### 3️⃣ `main.tf` — Resources (Clean & Consistent)

```hcl
# 🔹 main.tf
provider "aws" {
  region = var.region
}

resource "aws_s3_bucket" "logs" {
  bucket = local.bucket_name
  tags   = merge(local.common_tags, { Name = local.bucket_name })
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = merge(local.common_tags, { Name = local.vpc_name })
}

resource "aws_instance" "app" {
  ami           = "ami-0c7217cdde317cfec"  # Amazon Linux 2 (us-east-1)
  instance_type = "t3.micro"
  tags          = merge(local.common_tags, { Name = "${var.env}-app" })
}

```

✅ **Key Technique**:

`merge(local.common_tags, { Name = ... })` → inherit base tags + override per-resource.

---

### 4️⃣ `outputs.tf` — Output Variables (Results to Share)

```hcl
# 🔹 outputs.tf
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.logs.arn
  sensitive   = true  # 🔒 Hide in console!
}

output "environment" {
  description = "Environment used"
  value       = var.env
}

```

✅ **Best Practice**:

- Use `sensitive = true` for secrets/ARNs
- Output `var.env` to *verify* what was applied

---

## 🧪 Lab: Test Variable Precedence (Priority Rules)

Terraform loads variables in **strict order** — higher wins:

| Source | Precedence | Example | Use Case |
| --- | --- | --- | --- |
| `default = "dev"` | 1 (Lowest) | Fallback values | Safe defaults |
| `export TF_VAR_env=staging` | 2 | Shell env vars | CI/CD (use cautiously) |
| `terraform.tfvars` | 3 | Auto-loaded | Standard: env configs |
| `-var-file="prod.tfvars"` | 4 | Explicit file | Production overrides |
| `-var="env=prod"` | 5 (Highest) | CLI override | Debugging/ad-hoc |

### 🖥️ Try It Step-by-Step:

```bash
# 1. Start with defaults
mv terraform.tfvars terraform.tfvars.bak
terraform plan | grep Environment
# → "Environment" = "dev"

# 2. Restore tfvars (auto-loaded)
mv terraform.tfvars.bak terraform.tfvars
terraform plan | grep Environment
# → "Environment" = "staging" (from terraform.tfvars)

# 3. Override via CLI (highest priority!)
terraform plan -var="env=prod" | grep Environment
# → "Environment" = "prod"

# 4. Try environment variable
export TF_VAR_env="from-env"
terraform plan | grep Environment
# → "from-env" (but CLI still wins over env vars!)

```

> ⚠️ Security Note:
> 
> 
> Never commit secrets in `.tfvars`. Use:
> 
> - `TF_VAR_secret=xxx` (short-lived)
> - AWS SSM / Secrets Manager (Day 10+)

---

## 🛠️ Essential Variable Commands

| Command | Use Case |
| --- | --- |
| `terraform plan -var="env=prod"` | Quick override |
| `terraform apply -var-file="prod.tfvars"` | Apply prod config |
| `terraform output` | Show all outputs |
| `terraform output vpc_id` | Get specific output |
| `terraform console` | Interactive HCL shell → Try `> var.env`! |
| `terraform output -json \| jq .bucket_arn.value` | Parse outputs in scripts |

---

## ❓ FAQ (Day 5 Deep Dive)

**Q: When should I use `locals` vs `variables`?**

A:

- `variable`: External input (you *change* it)
- `local`: Internal helper (computed from vars/resources)
→ Use `locals` for non-configurable values.

**Q: Why do I get `Invalid interpolation syntax`?**

A: You’re missing `$`! ✅ `"${var.env}"` ❌ `"{var.env}"`

**Q: How do I pass lists/maps?**

A: Use `type = list(string)` and `.tfvars`:

```hcl
# variables.tf
variable "allowed_ips" { type = list(string) }

# prod.tfvars
allowed_ips = ["1.2.3.4/32", "5.6.7.8/32"]

```

---

## ➡️ Summary: Variables = Config Superpowers

| Concept | Benefit |
| --- | --- |
| **Input Vars** | → One config, *many* environments |
| **Locals** | → DRY code, fewer copy-paste bugs |
| **Outputs** | → Compose modules (VPC → EKS) |
| **Precedence** | → Granular control: defaults → CLI |

> 🎯 Golden Rule:
> 
> 
> **“No hardcoded values in production Terraform — only variables.”**
>