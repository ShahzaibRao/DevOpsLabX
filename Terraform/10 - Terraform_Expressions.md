# **1.10 : Terraform Expressions — Conditional, Dynamic & Splat**

🎯 **Goal**: Replace repetitive or verbose logic with **3 expressive patterns** — no functions needed.

> ✅ You’ll master:
> 
> - ✅ **Conditional (`condition ? true : false`)** → Environment-aware configs
> - ✅ **Dynamic blocks** → Generate nested configs (security groups, IAM policies)
> - ✅ **Splat (`[*]`)** → Extract lists from `count` resources in one line

---

## 🧠 Why Expressions > Copy-Paste

> ❌ Without expressions:
> 
> - 10x `ingress` blocks for 10 security rules
> - `if env == "prod"` logic duplicated across 5 resources
> - Manual `aws_instance.example[0].id`, `[1].id`, `[2].id`

✅ **With expressions**:

→ **DRY**, **readable**, and **scalable** configs — *before* you learn functions.

---

## 📦 Expression Cheat Sheet

| Expression | Syntax | Use Case | When to Use |
| --- | --- | --- | --- |
| **Conditional** | `cond ? true_val : false_val` | Toggle values by env/flag | One-off decisions (instance size, tags) |
| **Dynamic Block** | `dynamic "block" { for_each = ... }` | Generate *nested* blocks (e.g., `ingress`) | Variable-length nested configs |
| **Splat** | `resource[*].attr` | Extract lists from `count` resources | Feed IDs/IPs to other resources |

> 💡 Golden Rule:
> 
> 
> **“If you’re repeating a block 3+ times — `dynamic` is calling.”**
> 

---

## ✏️ Hands-On: Expressions in Action

### 🔹 Prerequisites (`variables.tf`)

```hcl
variable "env" {
  type    = string
  default = "dev"
}

variable "instance_count" {
  type    = number
  default = 2
}

# 🔹 For Dynamic Blocks: list of security rules
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS"
    }
  ]
}

```

---

### 1️⃣ Conditional Expression — Environment-Aware EC2

### `main.tf`

```hcl
resource "aws_instance" "app" {
  count = var.instance_count

  ami           = "ami-0c7217cdde317cfec"  # Amazon Linux 2
  # ✅ Conditional: dev = t2.micro, prod = t3.medium
  instance_type = var.env == "dev" ? "t2.micro" : "t3.medium"

  tags = {
    Name        = "app-${count.index}"
    Environment = var.env
  }
}

```

✅ **How it works**:

- `var.env == "dev"` → condition
- `? "t2.micro"` → value if **true**
- `: "t3.medium"` → value if **false**

> 🔄 Test it:
> 
> 
> ```bash
> terraform plan -var="env=prod"  # → t3.medium
> terraform plan -var="env=dev"   # → t2.micro
> 
> ```
> 

---

### 2️⃣ Dynamic Block — Security Group Rules (No Copy-Paste!)

### `main.tf`

```hcl
resource "aws_security_group" "app" {
  name = "app-sg-${var.env}"

  # 🔁 Dynamic: Generate `ingress` blocks from `var.ingress_rules`
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
}

```

✅ **Why this beats copy-paste**:

- Add a new rule to `ingress_rules` → **auto-added** to SG
- Remove a rule → **auto-removed** (no manual diff!)
- Zero risk of typos (e.g., `433` vs `443`)

> 💡 Pro Tip:
> 
> 
> Use `dynamic` for *any* nested block:
> 
> - `aws_iam_policy_document` → `statement`
> - `aws_cloudwatch_log_metric_filter` → `metric_transformation`

---

### 3️⃣ Splat Expression — Get All IDs in One Line

### `main.tf`

```hcl
# ✅ Get *all* instance IDs from `count` resource
locals {
  instance_ids = aws_instance.app[*].id
  private_ips  = aws_instance.app[*].private_ip
}

# Output the lists
output "instance_ids" {
  value = local.instance_ids
}

output "private_ips" {
  value = local.private_ips
}

```

✅ **How `[*]` works**:

`aws_instance.app[*].id` =

→ `[aws_instance.app[0].id, aws_instance.app[1].id, ...]`

> 🔄 Compare to old way:
> 
> 
> ```hcl
> # ❌ Manual, fragile, doesn’t scale
> instance_ids = [
>   aws_instance.app[0].id,
>   aws_instance.app[1].id,
>   aws_instance.app[2].id  # Oops — forgot to update when count=4!
> ]
> 
> ```
> 

---

## 🧪 Lab: Expressions in Action

### 🔧 Commands to Try:

```bash
# 1. See conditional in action
terraform plan -var="env=prod" | grep "instance_type"

# 2. Verify dynamic SG rules
terraform plan | grep -A5 "ingress {"

# 3. Inspect splat outputs
terraform apply -auto-approve
terraform output instance_ids
# → ["i-abc123", "i-def456"]

```

### ✅ Expected Output:

```bash
$ terraform output
instance_ids = [
  "i-0a1b2c3d4e5f6a7b8",
  "i-0f9e8d7c6b5a4f3e2"
]
private_ips = [
  "10.0.1.10",
  "10.0.1.11"
]

```

---

## ❓ FAQ (Day 10 Edition)

**Q: Can I use `dynamic` with `for_each` resources?**

A: ✅ Yes! But use `each.value` inside `content`:

```hcl
dynamic "ingress" {
  for_each = each.value.rules
  content {
    from_port = ingress.value.from_port
    # ...
  }
}

```

**Q: What’s the difference between `[*]` and `.*`?**

A:

- `[*]` → **safe**: `[1, 2, 3][*].id` = `[]` if empty
- `.*` → **legacy**: `[1, 2, 3].*.id` = `null` if empty → avoid

**Q: Can I chain expressions?** (e.g., `dynamic` + conditional)

A: ✅ Yes! Example:

```hcl
dynamic "ingress" {
  for_each = var.env == "prod" ? var.prod_rules : var.dev_rules
  # ...
}

```

---

## ➡️ Summary: Expressions = Config Superpowers

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ Copy-pasted `ingress` blocks | ✅ `dynamic "ingress"` |
| ❌ Hardcoded `t3.large` for prod | ✅ `env == "prod" ? "t3.large" : "t2.micro"` |
| ❌ Manual `[0].id, [1].id` lists | ✅ `resource[*].id` |

> 🎯 Golden Rule:
> 
> 
> **“If your config has a `for` loop in comments — it’s time for expressions.”**
>