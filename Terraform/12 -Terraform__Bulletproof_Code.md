# **1.12 : Terraform Functions — Part 2 (Validation, Numeric, Time, File)**

🎯 **Goal**: Add **safety rails** to your configs — validate inputs, handle time, read files, and crunch numbers *safely*.

> ✅ You’ll master:
> 
> - ✅ **Validation**: `length()`, `regex()`, `can()`, `endswith()`
> - ✅ **Numeric**: `sum(...)`, `max(...)`, `abs()` + **spread operator `...`**
> - ✅ **Time**: `timestamp()`, `formatdate()` for audit trails
> - ✅ **File**: `fileexists()`, `file()`, `jsondecode()` for external configs

---

## 🛡️ 1. Validation Functions — *Fail Fast, Fail Safe*

### ✏️ `variables.tf` — Safe Instance Types

```hcl
variable "instance_type" {
  description = "EC2 instance type (T2/T3 only)"
  type        = string
  default     = "t3.micro"

  validation {
    # 🔹 Rule 1: Length 2–20 chars
    condition = (
      length(var.instance_type) >= 2 &&
      length(var.instance_type) <= 20
    )
    error_message = "Instance type must be 2–20 characters."
  }

  validation {
    # 🔹 Rule 2: Must match t2./t3. (e.g., t3.micro)
    condition = can(regex("^t[23]\\\\.\\\\w+$", var.instance_type))
    error_message = "Instance type must be t2.* or t3.* (e.g., t3.micro)."
  }
}

```

✅ **How `can(regex(...))` works**:

- `regex()` alone → **crashes** on no-match
- `can(regex())` → returns `true`/`false` → safe for validation

> 💡 Pro Tip:
> 
> 
> Use `can()` with *any* error-prone function: `can(cidrhost(...))`, `can(jsondecode(...))`.
> 

---

### ✅ Bonus: `endswith()` & `sensitive`

```hcl
# Enforce backup naming: "db-prod-backup"
variable "backup_name" {
  type = string
  validation {
    condition     = endswith(var.backup_name, "-backup")
    error_message = "Backup name must end with '-backup'."
  }
}

# Hide secrets in logs/state
variable "api_key" {
  type      = string
  sensitive = true  # 🔒 Masks in `plan`/`output`
  # ⚠️ Still base64-encoded in state — use Vault/SSM for true secrets!
}

```

---

## 🔢 2. Numeric Functions — *Math with Lists*

### Problem: Calculate Costs from Mixed Signs

```hcl
variable "monthly_costs" {
  type    = list(number)
  default = [200, -50, 300, 75]  # Negative = credit
}

```

### ✅ Solution: `for` + `abs()` + `sum(...)`

*(Yes — the **spread operator `...`** is critical!)*

```hcl
locals {
  # 🔹 Convert all to positive (abs per element)
  positive_costs = [for c in var.monthly_costs : abs(c)]
  # → [200, 50, 300, 75]

  # 🔹 Use `...` to unpack list → args for sum/max/min
  total_cost = sum(local.positive_costs...)    # → 625
  max_cost   = max(local.positive_costs...)    # → 300
  min_cost   = min(local.positive_costs...)    # → 50
  avg_cost   = local.total_cost / length(local.positive_costs)  # → 156.25
}

```

> ⚠️ Critical Gotcha:
> 
> 
> ❌ `sum([1, 2, 3])` → **invalid**
> 
> ✅ `sum(1, 2, 3)` or `sum(list...)` → **valid**
> 
> → **Always use `...` to unpack lists for numeric functions!**
> 

---

## 🕒 3. Time Functions — *Dynamic Naming & Auditing*

### ✏️ `locals.tf` — ISO-8601 Timestamps

```hcl
locals {
  # 🔹 Raw RFC3339 timestamp
  current_time = timestamp()  # → "2025-12-13T14:30:00Z"

  # 🔹 Formatted date (YYYY-MM-DD)
  date_ymd = formatdate("YYYY-MM-DD", local.current_time)
  # → "2025-12-13"

  # 🔹 Backup name with timestamp + random suffix
  backup_name = "db-${local.date_ymd}-${random_string.suffix.result}-backup"
  # → "db-2025-12-13-abc1-backup"
}

```

### 📅 Format Codes:

| Code | Output | Example |
| --- | --- | --- |
| `YYYY` | 4-digit year | `2025` |
| `MM` | 2-digit month | `12` |
| `DD` | 2-digit day | `13` |
| `hh:mm:ss` | Time | `14:30:45` |

> 💡 Pro Tip:
> 
> 
> Combine `random_string` + `timestamp` for *globally unique* names (S3/RDS).
> 

---

## 📁 4. File Functions — *External Configs (Safely!)*

### 📄 `config.json`

```json
{
  "database": {
    "host": "prod.cluster-xyz.us-east-1.rds.amazonaws.com",
    "port": 5432
  },
  "api": {
    "endpoint": "<https://api.example.com/v1>",
    "timeout": 30
  }
}

```

### ✏️ `locals.tf` — Safe File Loading

```hcl
locals {
  # 🔹 Check if file exists (avoid crashes)
  config_exists = fileexists("${path.module}/config.json")

  # 🔹 Load & decode (safe with fallback)
  config_data = local.config_exists ? (
    jsondecode(file("${path.module}/config.json"))
  ) : {}

  # 🔹 Extract values (with defaults)
  db_host    = lookup(local.config_data.database, "host", "localhost")
  api_timeout = lookup(local.config_data.api, "timeout", 10)
}

```

### 🔑 Key Functions:

| Function | Purpose | Safety Tip |
| --- | --- | --- |
| `fileexists(path)` | ✅ Check before reading | Prevents `apply` crashes |
| `file(path)` | Read raw content | Only for local files |
| `jsondecode(json)` | Parse JSON → HCL map | Wrap in `can()` for validation |
| `yamldecode(yaml)` | Parse YAML → HCL map | Same as above |

> ⚠️ Critical:
> 
> 
> Never hardcode secrets in `.json`/`.tfvars`! Use:
> 
> - AWS SSM Parameter Store
> - HashiCorp Vault
> - `TF_VAR_secret=xxx` (short-lived)

---

## 🧪 Lab: Test It Yourself

### 🔧 Commands to Try:

```bash
# 1. Test validation (edit variables → run plan)
terraform plan -var="instance_type=t4.large"

# 2. See outputs
terraform output

# 3. Format code (team consistency)
terraform fmt -recursive

# 4. Validate syntax
terraform validate

```

### ✅ Expected Output:

```bash
$ terraform output
avg_cost = 156.25
backup_name = "db-2025-12-13-abc1-backup"
db_host = "prod.cluster-xyz.us-east-1.rds.amazonaws.com"
max_cost = 300
total_cost = 625

```

---

## ❓ FAQ (Day 12 Edition)

**Q: Why use `can(regex())` instead of just `regex()`?**

A: `regex()` **fails** on no-match → `can()` returns `false` → safe for validation.

**Q: What’s the spread operator (`...`) for?**

A: Unpacks lists → `sum([1,2,3]...)` = `sum(1,2,3)`. Required for `sum`/`max`/`min`.

**Q: Can I use `file()` with remote files (e.g., S3)?**

A: ❌ No — `file()` is local-only. Use `http` data source for remote JSON/YAML.

---

## ➡️ Summary: Functions = Config Superpowers

| Category | Key Functions | Production Use Case |
| --- | --- | --- |
| **Validation** | `can(regex)`, `endswith`, `length` | Enforce naming, cost, security rules |
| **Numeric** | `sum(...)`, `max(...)`, `abs` | Cost tracking, capacity planning |
| **Time** | `timestamp`, `formatdate` | Backup naming, audit trails |
| **File** | `fileexists`, `jsondecode` | External config (non-secrets!) |

> 🎯 Golden Rule:
> 
> 
> **“If you’re repeating logic in 3+ places — it’s a function waiting to be written.”**
>