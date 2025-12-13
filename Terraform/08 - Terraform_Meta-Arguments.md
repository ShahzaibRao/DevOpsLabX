# **1.8 : Terraform Meta-Arguments — Control Flow for Infrastructure**

🎯 **Goal**: Move beyond basic resources → use **meta-arguments** to build *dynamic, reliable, production-grade* configs.

> ✅ You’ll master:
> 
> - ✅ `count` vs `for_each`: When & why to use each
> - ✅ `depends_on`: Explicit dependencies (no more “race conditions”)
> - ✅ `lifecycle`: Protect critical resources (`prevent_destroy`, `create_before_destroy`)
> - ✅ Real-world patterns (S3 buckets, IAM users, multi-region)

---

## 🧠 Why Meta-Arguments Matter

> ❌ Without meta-arguments:
> 
> 
> → 10 copies of `aws_s3_bucket` blocks
> 
> → Resource creation order = guesswork
> 
> → Accidental `terraform destroy` = production outage
> 

✅ **With meta-arguments**:

→ **DRY**: One block → N resources

→ **Reliable**: Control dependencies & lifecycle

→ **Safe**: Guardrails against human error

---

## 📦 Meta-Arguments Cheat Sheet

| Meta-Argument | Input Type | Use Case | Stability | Production Ready? |
| --- | --- | --- | --- | --- |
| **`count`** | `number` | Fixed N identical resources (e.g., dev instances) | ⚠️ Low (index drift) | ❌ Avoid in prod |
| **`for_each`** | `map` / `set(string)` | Named resources (e.g., env-specific buckets) | ✅ High (key-based) | ✅ **Preferred** |
| **`depends_on`** | `[resource]` | Hidden dependencies (e.g., IAM → S3) | — | ✅ Use sparingly |
| **`lifecycle`** | Block | Protect resources, ignore drift | — | ✅ Critical for prod |

> 🎯 Golden Rule:
> 
> 
> **“Use `for_each` 90% of the time — `count` only for simple, temporary resources.”**
> 

---

## ✏️ Hands-On: `count` vs `for_each` — S3 Bucket Lab

### 🔹 `variables.tf` — Input Data

```hcl
# 🔸 List (for `count`)
variable "bucket_names_list" {
  type    = list(string)
  default = ["dev-logs", "staging-logs"]
}

# 🔸 Set (for `for_each`)
variable "bucket_names_set" {
  type = set(string)
  default = toset(["prod-logs", "dr-logs"])  # ← Dedupes automatically
}

# 🔸 Map (for advanced `for_each`)
variable "bucket_configs" {
  type = map(object({
    tags  = map(string)
    acl   = string
  }))
  default = {
    "audit" = {
      tags = { Purpose = "Compliance" },
      acl  = "private"
    }
    "backup" = {
      tags = { Purpose = "Disaster Recovery" },
      acl  = "private"
    }
  }
}

```

---

### 1️⃣ `count` — Simple, but Fragile

### `main.tf`

```hcl
# 🔹 COUNT: Creates 2 buckets (index 0 → "dev-logs", index 1 → "staging-logs")
resource "aws_s3_bucket" "count_example" {
  count  = length(var.bucket_names_list)
  bucket = "${var.bucket_names_list[count.index]}-push"
  tags   = { Name = "count-${count.index}" }
}

```

✅ **When to use**:

- Temporary dev resources
- Fixed-size pools (e.g., 3 identical bastion hosts)

⚠️ **Gotcha**:

> Delete var.bucket_names_list[0] → Terraform renames [1] → [0] → destroys & recreates resource!
> 

---

### 2️⃣ `for_each` — Production-Ready

### `main.tf`

```hcl
# 🔹 FOR_EACH (set): Creates 2 buckets (keys = "prod-logs", "dr-logs")
resource "aws_s3_bucket" "set_example" {
  for_each = var.bucket_names_set
  bucket   = "${each.key}-push"
  tags     = { Name = "set-${each.key}" }
}

# 🔹 FOR_EACH (map): Creates 2 buckets with custom config
resource "aws_s3_bucket" "map_example" {
  for_each = var.bucket_configs
  bucket   = "${each.key}-push"
  tags     = merge({ Name = each.key }, each.value.tags)
}

```

✅ **Why `for_each` wins**:

- Delete `"dr-logs"` → only **that bucket** is destroyed (others untouched)
- Keys = human-readable (`aws_s3_bucket.set_example["prod-logs"]`)
- Works with `map` of configs (not just names!)

---

## 🔄 `depends_on` — Explicit Dependencies

### Problem: Hidden Dependencies

```hcl
resource "aws_iam_user" "deployer" {
  name = "deployer"
}

resource "aws_s3_bucket" "assets" {
  bucket = "my-assets"
  # ❌ No *explicit* link to IAM user → Terraform may create S3 first!
}

```

### ✅ Solution: `depends_on`

```hcl
resource "aws_s3_bucket" "assets" {
  bucket = "my-assets"

  # ✅ Wait for IAM user to exist before creating bucket
  depends_on = [aws_iam_user.deployer]
}

```

> 💡 Best Practice:
> 
> 
> Only use `depends_on` for **hidden dependencies** (e.g., IAM policies, DNS records).
> 
> If you reference `resource.id` → Terraform auto-detects dependency → no `depends_on` needed!
> 

---

## 🛡️ `lifecycle` — Production Guardrails

### `main.tf`

```hcl
resource "aws_s3_bucket" "critical_logs" {
  bucket = "prod-critical-logs-push"

  lifecycle {
    # 🔒 Prevent accidental deletion (fails `destroy`/`apply`)
    prevent_destroy = true

    # 🔄 Replace bucket *before* destroying old (zero downtime)
    create_before_destroy = true

    # 🚫 Ignore external tag changes (e.g., cost allocation tags)
    ignore_changes = [tags]
  }
}

```

| Rule | Use Case | Danger if Missing |
| --- | --- | --- |
| `prevent_destroy = true` | Production DBs, S3 buckets | `terraform destroy` → data loss! |
| `create_before_destroy = true` | Stateful resources (RDS, buckets) | Downtime during updates |
| `ignore_changes = [tags]` | Resources tagged by external tools | Constant drift in `plan` |

> ⚠️ Critical:
> 
> 
> `prevent_destroy` **fails** `terraform apply` if the resource would be destroyed.
> 
> → To delete: Comment out `prevent_destroy`, run `apply`, then delete.
> 

---

## 🧪 Lab: Test Meta-Arguments

### 🔧 Commands to Try:

```bash
# 1. See resource addresses
terraform state list
# → aws_s3_bucket.count_example[0]
# → aws_s3_bucket.set_example["prod-logs"]
# → aws_s3_bucket.map_example["audit"]

# 2. Test `prevent_destroy` (try to delete critical bucket)
terraform destroy -target=aws_s3_bucket.critical_logs
# → ❌ "Resource has lifecycle.prevent_destroy set"

# 3. Verify dependencies
terraform graph | dot -Tpng > graph.png  # Visualize dependencies

```

### ✅ Expected Output:

```
# S3 buckets created (4 total)
aws_s3_bucket.count_example[0]  # "dev-logs-push"
aws_s3_bucket.count_example[1]  # "staging-logs-push"
aws_s3_bucket.set_example["prod-logs"]  # "prod-logs-push"
aws_s3_bucket.map_example["audit"]      # "audit-push"

```

---

## ❓ FAQ (Day 8 Edition)

**Q: When should I use `count` vs `for_each`?**

A:

- `count`: Dev/test resources, fixed-size pools (e.g., 3 bastions)
- `for_each`: Production resources, env-specific configs, maps of data

**Q: Can I use `count` and `for_each` together?**

A: ❌ **Never** — Terraform blocks both in the same resource.

**Q: Why does `for_each` require `toset()` for lists?**

A: Lists have order/duplicates → `for_each` needs unordered, unique keys → `toset()` fixes this.

---

## ➡️ Summary: Meta-Arguments = Infrastructure Control

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ 10x copy-pasted resources | ✅ `for_each` with `map(object({...}))` |
| ❌ Implicit dependencies | ✅ `depends_on` for hidden links |
| ❌ No resource protection | ✅ `lifecycle { prevent_destroy = true }` |
| ❌ `count` in production | ✅ `for_each` for stable addressing |

> 🎯 Golden Rule:
> 
> 
> **“If your resource has a *name* (not just an index) — use `for_each`.”**
>