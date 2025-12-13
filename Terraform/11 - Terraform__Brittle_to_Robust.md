# **1.11 : Terraform Functions — Part 1 (String, Numeric, Collection, Type Conversion)**

🎯 **Goal**: Move from *expressions* → **functions** — reusable, built-in logic for safe, DRY configs.

> ✅ You’ll master:
> 
> - ✅ **String**: `lower()`, `replace()`, `substr()`, `trim()`
> - ✅ **Numeric**: `max()`, `min()`, `abs()`
> - ✅ **Collection**: `length()`, `concat()`, `merge()`
> - ✅ **Type Conversion**: `toset()`, `tonumber()`, `tostring()`
> - ✅ **Lookup**: `lookup()` for env-specific configs
> - ✅ **`for` expressions** (not `for_each`!) for list/map transforms

---

## 🧠 Why Functions Matter

> ❌ Without functions:
> 
> - Manual S3 bucket name cleanup → `bucket = "Project Alpha!!!"` → **apply fails**
> - Copy-paste `tags = { Env = "dev", Cost = "eng" }` across 20 resources
> - Hardcoded `instance_type = "t3.medium"` in prod

✅ **With functions**:

→ Infrastructure that *self-corrects* invalid inputs

→ Truly DRY configs (no repeated logic)

→ Environment-aware resources (dev/staging/prod)

> 💡 Key Insight:
> 
> 
> **Terraform has 100+ built-in functions — but no custom functions.**
> 
> → Use them like LEGO blocks — small, composable, powerful.
> 

---

## 📦 Function Cheat Sheet (Part 1)

| Category | Key Functions | Use Case |
| --- | --- | --- |
| **String** | `lower()`, `replace()`, `substr()`, `trim()` | Sanitize names, tags, IDs |
| **Numeric** | `max()`, `min()`, `abs()` | Cost tracking, capacity planning |
| **Collection** | `length()`, `concat()`, `merge()` | Combine lists/maps |
| **Type Conversion** | `toset()`, `tonumber()`, `tostring()` | Normalize inputs, dedupe |
| **Lookup** | `lookup(map, key, default)` | Env-specific configs |

> 🎯 Golden Rule:
> 
> 
> **“Chain functions like shell pipes: `lower(replace(...))` → clean, readable, safe.”**
> 

---

## ✏️ Hands-On: Functions in Action

### 🔹 Prerequisites (`variables.tf`)

```hcl
variable "project_name" {
  type    = string
  default = "Project Alpha Resource!!"
}

variable "bucket_name_raw" {
  type    = string
  default = "Project Alpha Storage!! 123"
}

variable "allowed_ports_str" {
  type    = string
  default = "80,443,8080,3306"
}

variable "instance_sizes" {
  type = map(string)
  default = {
    dev      = "t2.micro"
    staging  = "t3.small"
    prod     = "t3.large"
  }
}

variable "default_tags" {
  type = map(string)
  default = {
    ManagedBy = "Terraform"
    Company   = "TechCorp"
  }
}

variable "env_tags" {
  type = map(string)
  default = {
    Environment = "dev"
    CostCenter  = "engineering"
  }
}

```

---

### 1️⃣ String Functions — S3 Bucket Name Sanitization

### `main.tf`

```hcl
locals {
  # 🔹 Chain: lower → replace spaces/special chars → trim to 63 chars
  sanitized_bucket_name = substr(
    replace(
      replace(
        lower(var.bucket_name_raw),
        " ", "-"
      ),
      "[^a-z0-9.-]", ""
    ),
    0, 63
  )
}

resource "aws_s3_bucket" "logs" {
  bucket = local.sanitized_bucket_name  # → "project-alpha-storage-123"
  tags = {
    Name = lower(var.project_name)  # → "project alpha resource!!"
  }
}

```

✅ **How it works** (inside → out):

1. `lower("Project Alpha Storage!! 123")` → `"project alpha storage!! 123"`
2. `replace(..., " ", "-")` → `"project-alpha-storage!!-123"`
3. `replace(..., "[^a-z0-9.-]", "")` → `"project-alpha-storage-123"`
4. `substr(..., 0, 63)` → ensures S3 compliance (≤63 chars, no caps/special chars)

> 💡 Pro Tip:
> 
> 
> Use regex `[^a-z0-9.-]` to remove *all* invalid chars — not just `!` or spaces.
> 

---

### 2️⃣ Collection Functions — Merge Tags

### `main.tf`

```hcl
locals {
  # 🔹 Merge multiple tag maps → single source of truth
  all_tags = merge(var.default_tags, var.env_tags)
}

resource "aws_s3_bucket" "logs" {
  bucket = local.sanitized_bucket_name
  tags   = local.all_tags  # → { ManagedBy="Terraform", Company="TechCorp", Environment="dev", CostCenter="engineering" }
}

```

✅ **Why `merge()` > copy-paste**:

- Change `Company = "TechCorp"` → updates *all* resources
- Add `Department` to `default_tags` → auto-propagates

> ⚠️ Gotcha:
> 
> 
> Later keys overwrite earlier ones: `merge({a="x"}, {a="y"})` → `{a="y"}`
> 

---

### 3️⃣ Type Conversion — Dedupe with `toset()`

### `main.tf`

```hcl
locals {
  # 🔹 Convert list → set to remove duplicates
  unique_regions = toset(["us-east-1", "us-west-2", "us-east-1"])
  # → ["us-east-1", "us-west-2"]
}

```

✅ **Use Cases**:

- Region lists (no duplicates)
- Security group CIDRs (deduped automatically)

---

### 4️⃣ `for` Expressions — Transform Lists (Not `for_each`!)

### `main.tf`

```hcl
locals {
  # 🔹 Split string → list of ports
  port_list = split(",", var.allowed_ports_str)
  # → ["80", "443", "8080", "3306"]

  # 🔹 Transform list → map of SG rules (using `for`)
  sg_rules = {
    for port in local.port_list :
    "port-${port}" => {
      port        = port
      description = "Allow traffic on port ${port}"
    }
  }
}

# Output to verify
output "port_list" { value = local.port_list }
output "sg_rules"  { value = local.sg_rules }

```

✅ **Output**:

```bash
port_list = [
  "80",
  "443",
  "8080",
  "3306"
]
sg_rules = {
  "port-80" = {
    "port" = "80"
    "description" = "Allow traffic on port 80"
  }
  "port-443" = {
    "port" = "443"
    "description" = "Allow traffic on port 443"
  }
  # ... etc
}

```

> 💡 Key Distinction:
> 
> - `for_each` → creates *resources*
> - `for` expression → transforms *data* (lists/maps)

---

### 5️⃣ Lookup Function — Env-Specific Instance Sizes

### `main.tf`

```hcl
variable "env" {
  type    = string
  default = "dev"
}

locals {
  # 🔹 Safe lookup: env → instance size (with fallback)
  instance_size = lookup(var.instance_sizes, var.env, "t2.micro")
  # → "t2.micro" (if env="dev"), "t2.micro" (if env="invalid")
}

resource "aws_instance" "app" {
  ami           = "ami-0c7217cdde317cfec"
  instance_type = local.instance_size
}

```

✅ **Test it**:

```bash
terraform plan -var="env=prod"   # → t3.large
terraform plan -var="env=staging" # → t3.small
terraform plan -var="env=qa"      # → t2.micro (fallback)

```

> ⚠️ Better Alternative:
> 
> 
> Use `try(var.instance_sizes[var.env], "t2.micro")` — more explicit, no deprecated `lookup()`.
> 

---

## 🧪 Lab: Test Functions in `terraform console`

### 🔧 Try These Live:

```bash
# 1. String chain
> lower(replace("HELLO WORLD!", "!", ""))
"hello world"

# 2. Sanitize S3 name
> substr(replace(replace(lower("Project Alpha!!"), " ", "-"), "[^a-z0-9.-]", ""), 0, 10)
"project-al"

# 3. Merge tags
> merge({a="1"}, {b="2"})
{ "a" = "1", "b" = "2" }

# 4. for expression
> { for p in ["80", "443"] : "port-${p}" => p }
{ "port-80" = "80", "port-443" = "443" }

```

---

## ❓ FAQ (Day 11 Edition)

**Q: Why use `replace()` twice for S3 names?**

A: First `replace()` → spaces → hyphens; second → *all* invalid chars (`!`, `_`, etc.) via regex.

**Q: Can I use `for` expressions in `resource` blocks?**

A: ❌ No — only in `locals`, `variables`, `outputs`. Use `for_each` for resources.

**Q: Is `lookup()` deprecated?**

A: ✅ Yes — HashiCorp recommends `try(map[key], default)` instead (more explicit, no surprises).

---

## ➡️ Summary: Functions = Config Superpowers

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ `bucket = "My Bucket!!"` | ✅ `bucket = lower(replace(var.raw, " ", "-"))` |
| ❌ Hardcoded `t3.large` in prod | ✅ `instance_type = try(var.sizes[var.env], "t2.micro")` |
| ❌ Copy-pasted tags | ✅ `tags = merge(local.common, var.custom)` |

> 🎯 Golden Rule:
> 
> 
> **“If you’re manipulating data *before* it hits a resource — use functions.”**
>