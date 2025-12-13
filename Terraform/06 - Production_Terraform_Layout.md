# **1.6 : Terraform File Structure & Best Practices**

---

🔗 **Repo**: [`github.com/Push/terraform-aws-labs/day06`](https://github.com/Push/terraform-aws-labs/tree/main/day06)

🎯 **Goal**: Move from monolithic `main.tf` → **organized, maintainable, team-ready** project layout.

---

## 🧠 Why Structure Matters

> 💡 Terraform merges all .tf files into one config — but how you organize them determines:
> 
> - ✅ Readability (Can a new engineer understand it in 5 mins?)
> - ✅ Maintainability (Can you update VPC *without* touching S3?)
> - ✅ Collaboration (Can two people work on `networking/` and `compute/` safely?)
> - ✅ Git hygiene (No accidental state/secret commits!)

❌ **Bad**: One `main.tf` with 2,000 lines

✅ **Good**: Logical, consistent, documented structure

---

## 📁 Production-Grade File Structure (Recommended)

### 🔹 Core Layout (`/day06/`)

```
day06/
├── backend.tf           # 🔐 Remote state config (S3 + locking)
├── provider.tf          # 🌐 AWS provider + default tags
├── variables.tf         # 📥 Input variables (with validation!)
├── locals.tf            # 🧮 Computed values (DRY logic)
├── main.tf              # 🏗️ *Only* top-level resources (or empty!)
├── vpc.tf               # 🌐 Networking (VPC, subnets, IGW)
├── storage.tf           # 🪣 S3, EBS, EFS
├── outputs.tf           # 📤 Exposed values (VPC ID, bucket ARN)
├── terraform.tfvars     # 🎛️ Default values (dev)
├── .gitignore           # 🚫 Block sensitive files
└── README.md            # 📝 Project docs

```

> ✅ Key Principle:
> 
> 
> **“Separation of Concerns”** — group by *function*, not just resource type.
> 

---

## ✏️ Step-by-Step: Refactor Day 5 → Day 6

### 1️⃣ `backend.tf` — Remote State (Isolated & Secure)

```hcl
# 🔐 backend.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.7.0" }
  }

  # ✅ S3 backend (encrypted + locked)
  backend "s3" {
    bucket         = "tech-tutorials-push-terraform-state-2025"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    use_lock_file  = true  # 🔒 Native S3 locking (no DynamoDB!)
  }
}

```

✅ **Why separate?**

- Backend config must be in `terraform {}` block
- Changes here require `terraform init -reconfigure`

---

### 2️⃣ `provider.tf` — Cloud Provider (With Default Tags)

```hcl
# 🌐 provider.tf
provider "aws" {
  region = var.region

  # ✅ Apply common tags to ALL resources (DRY!)
  default_tags {
    tags = local.common_tags
  }
}

```

✅ **Best Practice**:

`default_tags` eliminates repeating `tags = {...}` in *every* resource.

---

### 3️⃣ `variables.tf` — Input Variables (Safe & Validated)

```hcl
# 📥 variables.tf
variable "env" {
  description = "Environment (dev/staging/prod)"
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

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Invalid CIDR block."
  }
}

```

✅ **Critical**:

- Always add `description`
- Use `validation` to catch errors *early* (before `apply`!)

---

### 4️⃣ `locals.tf` — Computed Values (DRY Logic)

```hcl
# 🧮 locals.tf
locals {
  # Reusable naming convention
  name_prefix = "${var.app}-${var.env}"

  # Consistent tags (base + environment-specific)
  common_tags = {
    Environment = var.env
    Application = var.app
    Owner       = "terraform-team"
    Terraform   = "true"
  }

  # Globally unique S3 bucket name (no collisions!)
  bucket_name = "${local.name_prefix}-logs-${random_string.suffix.result}"
}

# Random suffix generator
resource "random_string" "suffix" {
  length  = 4
  special = false
  upper   = false
}

```

✅ **Why Locals?**

- Change `app` → *all* names/tags update automatically
- No more `"${var.app}-${var.env}-logs"` repeated 10×

---

### 5️⃣ `vpc.tf` — Networking (Focused & Reusable)

```hcl
# 🌐 vpc.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = { Name = "${local.name_prefix}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name_prefix}-igw" }
}

# Public subnets (multi-AZ)
resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.value))
  availability_zone = each.value
  tags = {
    Name = "${local.name_prefix}-public-${each.value}"
    Type = "Public"
  }
}

```

✅ **Best Practice**:

- Group *all* VPC-related resources here — no scattering across files
- Use `for_each` for multi-AZ (cleaner than `count`)

---

### 6️⃣ `storage.tf` — S3 (Secure by Default)

```hcl
# 🪣 storage.tf
resource "aws_s3_bucket" "logs" {
  bucket = local.bucket_name
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_bucket_public_access_block" "logs" {
  bucket                  = aws_s3_bucket.logs.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

```

✅ **Security First**:

- Encryption, versioning, public access block — *all enabled by default*
- No more "forgot to secure S3" incidents!

---

### 7️⃣ `outputs.tf` — Exposed Values (For Downstream Use)

```hcl
# 📤 outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.logs.bucket
}

output "environment" {
  description = "Environment used"
  value       = var.env
}

output "common_tags" {
  description = "Tags applied to all resources"
  value       = local.common_tags
  sensitive   = false  # Safe to show
}

```

✅ **Pro Tip**:

- Output `var.env` to *confirm* what was applied
- Use `sensitive = true` only for secrets/ARNs

---

## 🛡️ `.gitignore` — Block Sensitive Files

```
# 🔒 Terraform
terraform.tfstate
terraform.tfstate.backup
.terraform/
*.tfvars       # ← Exclude sensitive values!
*.tfvars.json
crash.log
override.tf
override.tf.json

```

> ✅ Critical:
> 
> 
> Never commit `.tfstate`, `.terraform/`, or `.tfvars` (use `terraform.tfvars.example` for templates!).
> 

---

## 🔄 How Terraform Loads Files

| Fact | Why It Matters |
| --- | --- |
| 📁 Loads **all `.tf` files** in dir | No need to `include` files — Terraform auto-merges them |
| 🔤 **Lexicographical order** (a→z) | `backend.tf` loads before `vpc.tf` — but order *rarely* matters (Terraform resolves dependencies) |
| ⚙️ `.tfvars` auto-loaded | `terraform.tfvars`, `*.auto.tfvars` loaded automatically |
| 📁 Ignores subdirectories | Use `modules/` for reusable components (Day 10+) |

---

## 🏗️ Advanced Patterns (For Future Reference)

### Option 1: Environment-Specific Directories

```
environments/
├── dev/
│   ├── backend.tf      # S3 key: "dev/..."
│   └── terraform.tfvars
├── staging/
│   ├── backend.tf      # S3 key: "staging/..."
│   └── terraform.tfvars
└── prod/
    ├── backend.tf      # S3 key: "prod/..."
    └── terraform.tfvars

```

→ ✅ Clear isolation

→ ❌ Duplicated code (use modules to fix)

### Option 2: Service-Based Modules (Scalable)

```
modules/
├── vpc/          # Reusable VPC module
├── s3-secure/    # S3 with encryption/versioning
└── ec2-web/      # Web server ASG + ALB

prod/
├── main.tf      # Calls modules with prod vars
└── prod.tfvars

```

→ ✅ Max reuse, min duplication

→ 🔜 Covered in Day 12+ (don’t rush!)

---

## 🧪 Lab: Validate & Test Your Structure

```bash
# 1. Initialize (backend + providers)
terraform init

# 2. Format all files (team consistency)
terraform fmt -recursive

# 3. Validate syntax & logic
terraform validate

# 4. Dry-run (should show no changes if refactored correctly)
terraform plan

# 5. Apply (if testing)
terraform apply -auto-approve

# 6. Clean up
terraform destroy -auto-approve

```

> ✅ Success Criteria:
> 
> - `terraform validate` passes
> - `terraform plan` shows *identical* resources as Day 5
> - No errors about missing variables/resources

---

## ❓ FAQ (Day 6 Edition)

**Q: Do file names matter? Can I use `networking.tf` instead of `vpc.tf`?**

A: ✅ Yes! File names are *conventions* — use what makes sense for your team. Just be consistent.

**Q: Should I put `random_string` in `locals.tf` or `main.tf`?**

A: `locals.tf` — it’s a *computed helper*, not a top-level resource.

**Q: Why separate `backend.tf`? Can’t it live in `main.tf`?**

A: Technically yes — but:

- Backend changes require `terraform init -reconfigure`
- Isolating it prevents accidental edits to critical config

---

## ➡️ Summary: Structure = Maintainability

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ One `main.tf` | ✅ Logical files (`vpc.tf`, `storage.tf`) |
| ❌ Hardcoded values | ✅ `variables.tf` + `.tfvars` |
| ❌ Repeated tags | ✅ `locals.tf` + `default_tags` |
| ❌ Committed state | ✅ `.gitignore` + remote backend |
| ❌ No validation | ✅ `validation {}` blocks |

> 🎯 Golden Rule:
> 
> 
> **“If a new engineer can’t understand your structure in 5 minutes — simplify it.”**
> 

---

## 📚 Next Up → #1.7: Type Constraints (`string`, `list`, `map`, `object`)

✅ We’ll:

- Master complex types (`map(string)`, `list(object)`)
- Build region-specific configs
- Validate nested structures

> 📝 Prep Task (Day 6):
> 
> 1. ✅ Refactor Day 5 → Day 6 structure
> 2. ✍️ **Blog Post**: *“How File Structure Prevents Terraform Chaos”*
>     - Show before/after directory trees
>     - Explain why `.gitignore` is non-negotiable
> 3. 📤 Submit via GitHub (`/day06/TASK.md`)

---

You’re not just organizing files — you’re building **infrastructure that scales with your team**. Every `.tf` file now has *purpose*, *consistency*, and *safety* baked in. 🏗️

Ready for type constraints? Send Day 7 — I’ll make `object({ ... })` feel like second nature.