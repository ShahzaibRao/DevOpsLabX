# **1.13 : Terraform Data Sources — Query, Don’t Hardcode**

🎯 **Goal**: Stop guessing AMI IDs or subnet IDs — **query AWS dynamically** and build *portable, environment-agnostic* configs.

> ✅ You’ll master:
> 
> - ✅ `data "aws_ami"` → Get latest AMIs (no more `ami-0c7217cdde317cfec`!)
> - ✅ `data "aws_vpc"` + `data "aws_subnet"` → Reuse shared infrastructure
> - ✅ Filtering with `tags`, `name`, `cidr_block`
> - ✅ Safe, read-only discovery (no `terraform apply` side effects)

---

## 🧠 Why Data Sources Matter

> ❌ Without data sources:
> 
> - Hardcoded `ami = "ami-0c721..."` → breaks when AMI rotates
> - Manual `subnet_id = "subnet-abc123"` → fails in staging/prod
> - Copy-paste VPC IDs across teams → configuration drift

✅ **With data sources**:

→ Infrastructure that *self-discovers* dependencies

→ Truly portable configs (dev → staging → prod)

→ Zero manual lookup — fully automated

> 💡 Golden Rule:
> 
> 
> **“If it already exists in AWS — `data` it. If you’re creating it — `resource` it.”**
> 

---

## 📦 Data Source Cheat Sheet (AWS)

| Data Source | Use Case | Key Filter Attributes |
| --- | --- | --- |
| **`aws_ami`** | Find OS images | `owners`, `name`, `virtualization_type`, `root_device_type` |
| **`aws_vpc`** | Reuse shared VPCs | `tags`, `cidr_block`, `default` |
| **`aws_subnet`** | Target specific subnets | `vpc_id`, `availability_zone`, `tags`, `cidr_block` |
| **`aws_iam_role`** | Attach to existing roles | `name` |
| **`aws_ssm_parameter`** | Read config from SSM | `name` |

> 🎯 Critical:
> 
> - `data` blocks are **read-only** — no side effects on `apply`
> - Values resolved during `terraform plan` → fully predictable

---

## ✏️ Hands-On: Data Sources in Action

### 🔹 Scenario: Deploy to *Shared* VPC & Subnets

- ✅ VPC: `Name = "shared-prod-vpc"` (already exists)
- ✅ Subnets: `Name = "app-subnet-a"`, `"app-subnet-b"` (in `us-east-1a`, `us-east-1b`)
- ✅ OS: Latest **Amazon Linux 2023** AMI

---

### 1️⃣ `data.tf` — Discover Existing Infrastructure

```hcl
# 🔹 Find shared VPC by tag
data "aws_vpc" "shared" {
  filter {
    name   = "tag:Name"
    values = ["shared-prod-vpc"]
  }
}

# 🔹 Find subnets in shared VPC
data "aws_subnet" "app_a" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.shared.id]
  }
  filter {
    name   = "tag:Name"
    values = ["app-subnet-a"]
  }
}

data "aws_subnet" "app_b" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.shared.id]
  }
  filter {
    name   = "tag:Name"
    values = ["app-subnet-b"]
  }
}

# 🔹 Get latest Amazon Linux 2023 AMI
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]  # Official AMIs

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

```

✅ **Why `owners = ["amazon"]`?**

Prevents AMIs from third parties → security + consistency.

---

### 2️⃣ `main.tf` — Deploy Using Discovered IDs

```hcl
# 🔹 Deploy to shared subnets (no hardcoded IDs!)
resource "aws_instance" "app" {
  count = 2

  ami           = data.aws_ami.al2023.id          # ← Dynamic AMI
  instance_type = "t3.micro"
  subnet_id     = [data.aws_subnet.app_a.id, data.aws_subnet.app_b.id][count.index]

  tags = {
    Name        = "app-${count.index + 1}"
    Environment = "prod"
  }
}

```

✅ **Key Insight**:

- `data.aws_ami.al2023.id` → always the *latest* AL2023 AMI
- `subnet_id` → pulls IDs from *existing* subnets
- Full portability: Works in **any account/region** with matching tags

---

## 🧪 Lab: Test Data Sources

### 🔧 Commands to Try:

```bash
# 1. See discovered values
terraform console
> data.aws_vpc.shared.cidr_block
"10.0.0.0/16"

> data.aws_ami.al2023.id
"ami-0abcdef1234567890"

# 2. Dry-run (no AWS writes!)
terraform plan

# 3. Apply safely
terraform apply -auto-approve

# 4. Verify in AWS Console
aws ec2 describe-instances --query 'Reservations[].Instances[].SubnetId'
# → Should match data.aws_subnet.app_a.id / app_b.id

```

---

## ⚠️ Critical Best Practices

| Do ✅ | Don’t ❌ |
| --- | --- |
| ✅ Filter by `tags` (not IDs) | ❌ Hardcode `subnet_id = "subnet-abc123"` |
| ✅ Use `most_recent = true` for AMIs | ❌ Pin AMI unless required (security patching!) |
| ✅ Isolate `data` blocks in `data.tf` | ❌ Mix `data`/`resource` in `main.tf` |
| ✅ Validate filters early (`terraform plan`) | ❌ Wait for `apply` to catch errors |

> 💡 Pro Tip:
> 
> 
> Use `aws_vpc.default` for the *default VPC* (no filters needed!):
> 
> ```hcl
> data "aws_vpc" "default" {
>   default = true
> }
> 
> ```
> 

---

## ❓ FAQ (Day 13 Edition)

**Q: Can data sources cause `apply` to fail if resources don’t exist?**

A: ✅ Yes — `terraform plan` will fail *early* if no resources match filters → safe!

**Q: What if multiple resources match my filter?**

A: ❌ `data` requires **exactly one match** → use more specific filters (e.g., add `cidr_block`).

**Q: Can I use data sources for secrets (e.g., RDS passwords)?**

A: ❌ Never! Use `aws_secretsmanager_secret_version` or `aws_ssm_parameter` instead.

---

## ➡️ Summary: Data Sources = Infrastructure Intelligence

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ `ami = "ami-0c721..."` | ✅ `ami = data.aws_ami.al2023.id` |
| ❌ Manual subnet lookup | ✅ `subnet_id = data.aws_subnet.app_a.id` |
| ❌ Copy-paste VPC IDs | ✅ Filter by `tag:Name` |

> 🎯 Golden Rule:
> 
> 
> **“If it’s not in your Terraform state — `data` it before you use it.”**
>