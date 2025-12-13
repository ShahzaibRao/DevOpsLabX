# **1.9 : Terraform Lifecycle Rules — Prevent Disasters Before They Happen**

🎯 **Goal**: Go beyond basic resources → use **lifecycle rules** to build *self-healing, zero-downtime, audit-proof* infrastructure.

> ✅ You’ll master:
> 
> - ✅ `create_before_destroy` → zero-downtime updates
> - ✅ `prevent_destroy` → stop accidental `terraform destroy`
> - ✅ `ignore_changes` → coexist with external tools (ASG, monitoring)
> - ✅ `replace_triggered_by` → force rebuilds on dependency changes
> - ✅ `precondition`/`postcondition` → enforce compliance *before & after* apply

---

## 🧠 Why Lifecycle Rules Matter

> ❌ Without lifecycle rules:
> 
> - `terraform apply` → destroys DB → creates new → **60s downtime**
> - `terraform destroy` → deletes prod S3 bucket → **irrecoverable data loss**
> - ASG scales to 100 → `terraform plan` → wants to revert to 2 → **fighting your tools**

✅ **With lifecycle rules**:

→ Infrastructure that *protects itself*

→ Zero-downtime deployments by default

→ Safe coexistence with external systems

---

## 📦 Lifecycle Rules Cheat Sheet

| Rule | Use Case | Risk if Missing | Production Ready? |
| --- | --- | --- | --- |
| **`create_before_destroy = true`** | EC2, RDS, ASG | Downtime during updates | ✅ **Critical for stateful resources** |
| **`prevent_destroy = true`** | Prod DBs, S3 buckets, IAM roles | Accidental deletion = outage | ✅ **Non-negotiable for critical resources** |
| **`ignore_changes = [attr]`** | ASG `desired_capacity`, EC2 `tags` | Terraform fights external tools | ✅ Essential for hybrid infra |
| **`replace_triggered_by = [...]`** | Rebuild instances on SG/AMI change | Stale configs, security gaps | ✅ Key for immutable infra |
| **`precondition`** | Enforce region/tag policies *before* create | Invalid resources get created | ✅ Guardrails for standards |
| **`postcondition`** | Verify tags/compliance *after* create | Deployed resources don’t meet policy | ✅ Audit-proof deployments |

> 🎯 Golden Rule:
> 
> 
> **“If your resource holds data or serves traffic — it needs lifecycle rules.”**
> 

---

## ✏️ Hands-On: Lifecycle Rules in Action

### 🔹 `variables.tf` — Setup

```hcl
variable "allowed_regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2"]
}

variable "compliance_tags" {
  type    = list(string)
  default = ["Environment", "Compliance", "Owner"]
}

```

---

### 1️⃣ `create_before_destroy` — Zero-Downtime EC2

### `main.tf`

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  # 🔒 Create NEW instance BEFORE destroying old (zero downtime!)
  lifecycle {
    create_before_destroy = true
  }
}

```

✅ **How it works**:

`terraform apply` →

1. Creates `aws_instance.app-new`
2. Switches load balancer to new instance
3. Destroys `aws_instance.app-old`

⚠️ **Gotcha**:

- Requires *unique resource names* (e.g., random suffixes, timestamps)
- Use with `random_string` or `uuid()` for naming

---

### 2️⃣ `prevent_destroy` — Stop Accidental Deletion

### `main.tf`

```hcl
resource "aws_s3_bucket" "prod_logs" {
  bucket = "prod-app-logs-${random_string.suffix.result}"

  # 🔒 BLOCK deletion (fails `destroy`/`apply`)
  lifecycle {
    prevent_destroy = true
  }
}

```

✅ **When `terraform destroy` runs**:

> ❌ Error: Instance cannot be destroyed
> 
> 
> `Resource aws_s3_bucket.prod_logs has lifecycle.prevent_destroy set`
> 

➡️ **To delete safely**:

1. Comment out `prevent_destroy = true`
2. Run `terraform apply` → updates state
3. Run `terraform destroy`

> 💡 Pro Tip:
> 
> 
> Apply to *all* production data stores:
> 
> - RDS databases
> - EFS/EBS volumes
> - DynamoDB tables

---

### 3️⃣ `ignore_changes` — Coexist with External Tools

### `main.tf`

```hcl
resource "aws_autoscaling_group" "web" {
  # ... ASG config ...
  desired_capacity = 3
  max_size         = 10

  # 🤝 Allow ASG to manage capacity (ignore external changes)
  lifecycle {
    ignore_changes = [
      desired_capacity,  # Auto-scaling handles this
      target_group_arns  # ALB may add targets
    ]
  }
}

```

✅ **Why this matters**:

- Auto-scaling scales to 50 → Terraform *won’t revert to 3*
- ALB adds targets → Terraform *won’t fight it*
- Monitoring tools add `CostCenter` tag → Terraform *ignores it*

> ⚠️ Danger Zone:
> 
> 
> `ignore_changes = all` → Terraform *stops managing the resource* → **avoid in prod**.
> 

---

### 4️⃣ `replace_triggered_by` — Immutable Infrastructure

### `main.tf`

```hcl
resource "aws_security_group" "app" {
  name = "app-sg"
  # ... ingress/egress rules ...
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  vpc_security_group_ids = [aws_security_group.app.id]

  # 🔁 Rebuild instance if security group changes
  lifecycle {
    replace_triggered_by = [
      aws_security_group.app.id  # Key change → force rebuild
    ]
  }
}

```

✅ **When SG rules update**:

`terraform apply` →

1. Creates new SG (`app-sg-v2`)
2. Creates new EC2 with `app-sg-v2`
3. Destroys old EC2 + `app-sg`

> 💡 Best Practice:
> 
> 
> Chain with AMI changes:
> 
> ```hcl
> replace_triggered_by = [
>   aws_security_group.app.id,
>   data.aws_ami.new.id  # Rebuild on new AMI
> ]
> 
> ```
> 

---

### 5️⃣ `precondition` — Enforce Policies *Before* Create

### `main.tf`

```hcl
data "aws_region" "current" {}

resource "aws_s3_bucket" "compliant" {
  bucket = "compliant-bucket-${random_string.suffix.result}"

  # ✅ Pre-check: Region must be allowed
  lifecycle {
    precondition {
      condition = contains(
        var.allowed_regions,
        data.aws_region.current.name
      )
      error_message = "Region ${data.aws_region.current.name} not allowed. Use: ${join(", ", var.allowed_regions)}"
    }
  }
}

```

✅ **On invalid region (`eu-west-1`)**:

> ❌ Error: Resource precondition failed
> 
> 
> `Region eu-west-1 not allowed. Use: us-east-1, us-west-2`
> 

> 💡 Precondition Ideas:
> 
> - `length(var.tags["Owner"]) > 0` → Owner tag required
> - `var.instance_type != "t2.micro"` → No dev instances in prod

---

### 6️⃣ `postcondition` — Verify Compliance *After* Create

### `main.tf`

```hcl
resource "aws_s3_bucket" "audited" {
  bucket = "audited-bucket-${random_string.suffix.result}"
  tags = {
    Environment = "prod"
    Compliance  = "SOC2"
    Owner       = "security-team"
  }

  # ✅ Post-check: All required tags exist
  lifecycle {
    postcondition {
      condition = alltrue([
        for tag in var.compliance_tags :
        contains(keys(self.tags), tag)
      ])
      error_message = "Missing required tags: ${join(", ", var.compliance_tags)}"
    }
  }
}

```

✅ **On missing `Compliance` tag**:

> ❌ Error: Resource postcondition failed
> 
> 
> `Missing required tags: Environment, Compliance, Owner`
> 

> 💡 Postcondition Ideas:
> 
> - `self.arn != ""` → ARN must be set
> - `length(self.tags) >= 3` → At least 3 tags
> - `self.server_side_encryption_configuration[0].rule[0].apply_server_side_encryption_by_default.sse_algorithm == "AES256"` → S3 encryption enforced

---

## 🧪 Lab: Test Lifecycle Rules

### 🔧 Commands to Try:

```bash
# 1. Test `prevent_destroy` (try to delete prod bucket)
terraform destroy -target=aws_s3_bucket.prod_logs
# → ❌ "Resource has lifecycle.prevent_destroy set"

# 2. Test `ignore_changes` (manually update ASG capacity)
aws autoscaling set-desired-capacity --auto-scaling-group-name app-asg --desired-capacity 5
terraform plan
# → ✅ "No changes" (capacity ignored)

# 3. Test `precondition` (deploy to blocked region)
TF_VAR_region=eu-west-1 terraform plan
# → ❌ "Region eu-west-1 not allowed"

```

---

## ❓ FAQ (Day 9 Edition)

**Q: Can I use multiple lifecycle rules together?**

A: ✅ Yes! Example:

```hcl
lifecycle {
  create_before_destroy = true
  prevent_destroy       = true
  ignore_changes        = [tags]
}

```

**Q: Why does `create_before_destroy` sometimes still cause downtime?**

A: If dependencies aren’t handled:

- LB must support blue/green
- DNS TTL must be low
- App must be stateless

**Q: Do lifecycle rules apply to `module` blocks?**

A: ✅ Yes — add `lifecycle` to the module call:

```hcl
module "db" {
  source = "./rds"
  lifecycle {
    prevent_destroy = true
  }
}

```

---

## ➡️ Summary: Lifecycle Rules = Infrastructure Safety

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ Manual `terraform apply` without guardrails | ✅ `prevent_destroy` on all data stores |
| ❌ Downtime during updates | ✅ `create_before_destroy` for stateful resources |
| ❌ Fighting with auto-scaling | ✅ `ignore_changes = [desired_capacity]` |
| ❌ Non-compliant resources deployed | ✅ `precondition` + `postcondition` |

> 🎯 Golden Rule:
> 
> 
> **“If you wouldn’t allow a human to delete it — `prevent_destroy = true`.”**
>