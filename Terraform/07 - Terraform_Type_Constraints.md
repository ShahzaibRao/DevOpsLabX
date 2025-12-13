# **1.7 : Terraform Type Constraints — Primitive, Complex & Special Types**

🔗 **Repo**: [`github.com/Push/terraform-aws-labs/day07`](https://github.com/Push/terraform-aws-labs/tree/main/day07)

🎯 **Goal**: Master **all 7 type constraints** — write configs that *fail fast* on invalid inputs.

> ✅ You’ll master:
> 
> - ✅ **Primitive**: `string`, `number`, `bool`
> - ✅ **Complex**: `list`, `set`, `map`, `tuple`, `object`
> - ✅ **Special**: `null`, `any`
> - ✅ When to use each — and *critical gotchas* (like `set` vs `list`)

---

## 🧠 Why Type Constraints Matter

> ❌ var.region = 123 → crashes late (during apply)
> 
> 
> ✅ `type = string` → **fails early** (during `plan`/`validate`)
> 

Type constraints = **self-documenting, self-validating** configs.

---

## 📦 Type Constraints Cheat Sheet

| Category | Type | Format | Use Case | Duplicates? | Ordered? | Index Access? |
| --- | --- | --- | --- | --- | --- | --- |
| **Primitive** | `string` | `"hello"` | Names, IDs, tags | — | — | — |
|  | `number` | `42`, `3.14` | Ports, counts, sizes | — | — | — |
|  | `bool` | `true`, `false` | Flags (e.g., `monitoring`) | — | — | — |
| **Complex** | `list(<TYPE>)` | `["a", "b"]` | Ordered lists (AZs, ports) | ✅ | ✅ | ✅ (`[0]`) |
|  | `set(<TYPE>)` | `["a", "b"]` | Unique values (regions, tags) | ❌ | ❌ | ❌ (→ `tolist()`) |
|  | `map(<TYPE>)` | `{k="v"}` | Key-value (tags, env vars) | Keys ❌ | ❌ | ❌ (→ `["key"]`) |
|  | `tuple([T1, T2])` | `[42, "tcp"]` | Fixed-position mixed types | ✅ | ✅ | ✅ (`[0]`) |
|  | `object({k=T})` | `{port=443}` | Structured data (config blocks) | Keys ❌ | ❌ | ❌ (`["key"]`) |
| **Special** | `null` | `null` | Optional/unset values | — | — | — |
|  | `any` | `42`, `"hi"` | Legacy/fallback (avoid in prod) | — | — | — |

> 💡 Golden Rule:
> 
> 
> **“Use the *most specific* type possible — not `any`.”**
> 

---

## ✏️ Hands-On: Type Constraints in Action

### 1️⃣ Primitive Types — The Basics

### `variables.tf`

```hcl
# 🔹 string (names, regions)
variable "region" {
  type        = string
  default     = "us-east-1"
  description = "AWS region"
}

# 🔹 number (ports, counts)
variable "instance_count" {
  type        = number
  default     = 1
  description = "Number of EC2 instances"
}

# 🔹 bool (flags)
variable "enable_monitoring" {
  type        = bool
  default     = true
  description = "Enable CloudWatch detailed monitoring"
}

```

### `main.tf`

```hcl
resource "aws_instance" "app" {
  count         = var.instance_count          # ← number
  ami           = "ami-0c7217cdde317cfec"
  instance_type = "t3.micro"
  monitoring    = var.enable_monitoring      # ← bool

  tags = {
    Region = var.region                      # ← string
  }
}

```

✅ **Validation**:

`terraform plan -var="region=123"` → ❌ `Invalid value for "region": string required.`

---

### 2️⃣ `list(string)` — Ordered, Duplicates Allowed

### `variables.tf`

```hcl
# 🔹 List of CIDR blocks (order matters for priority)
variable "allowed_cidrs" {
  type    = list(string)
  default = [
    "10.0.0.0/8",      # Corp network (index 0)
    "192.168.0.0/16",  # Remote offices (index 1)
    "10.0.0.0/8"       # ❌ Duplicate allowed!
  ]
}

```

### `main.tf`

```hcl
resource "aws_security_group_rule" "ingress" {
  # Allow from FIRST CIDR only (index 0)
  cidr_blocks = [var.allowed_cidrs[0]]  # ← ["10.0.0.0/8"]
}

```

> ⚠️ Gotcha:
> 
> 
> `var.allowed_cidrs[99]` → ❌ `Invalid index` (out of bounds)
> 

---

### 3️⃣ `set(string)` — Unique, Unordered

### `variables.tf`

```hcl
# 🔹 Set of regions (no duplicates!)
variable "deploy_regions" {
  type = set(string)
  default = [
    "us-east-1",
    "us-west-2",
    "us-east-1"  # ← Duplicate auto-removed!
  ]
}

```

### `main.tf` — **Critical: Can’t use `[0]`!**

```hcl
# ❌ WRONG: set has no index!
# region = var.deploy_regions[0]

# ✅ CORRECT: Convert to list first
locals {
  regions_list = tolist(var.deploy_regions)
}

# Now safe to use index (but order is *arbitrary*!)
region = local.regions_list[0]

```

✅ **Why sets?**

- Prevent accidental duplicates (e.g., `["prod", "prod"]`)
- Enforce uniqueness (e.g., region lists)

---

### 4️⃣ `map(string)` — Key-Value Pairs

### `variables.tf`

```hcl
# 🔹 Tags (all values must be string)
variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Owner       = "terraform-team"
    CostCenter  = "engineering"  # ← All strings
  }
}

```

### `main.tf`

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-bucket"
  tags   = var.tags  # ← Entire map in one line!
}

```

> 💡 Pro Tip:
> 
> 
> Use `merge(local.common_tags, var.custom_tags)` for layered tags.
> 

---

### 5️⃣ `tuple([number, string, number])` — Mixed Types, Fixed Order

### `variables.tf`

```hcl
# 🔹 Ingress rule config: [from_port, protocol, to_port]
variable "ingress_rule" {
  type = tuple([number, string, number])
  default = [443, "tcp", 443]  # ← Position matters!
}

```

### `main.tf`

```hcl
resource "aws_security_group_rule" "https" {
  from_port   = var.ingress_rule[0]  # ← 443 (number)
  protocol    = var.ingress_rule[1]  # ← "tcp" (string)
  to_port     = var.ingress_rule[2]  # ← 443 (number)
  cidr_blocks = ["0.0.0.0/0"]
}

```

✅ **When to use**:

- Fixed-position configs (e.g., port/protocol/port)
- When order *and* type matter

---

### 6️⃣ `object({ ... })` — Structured Configuration

### `variables.tf`

```hcl
# 🔹 Structured EC2 config (mixed types, named fields)
variable "instance_config" {
  type = object({
    region        = string
    instance_type = string
    count         = number
    monitoring    = bool
  })
  default = {
    region        = "us-east-1"
    instance_type = "t3.micro"
    count         = 2
    monitoring    = true
  }
}

```

### `main.tf`

```hcl
resource "aws_instance" "app" {
  count         = var.instance_config.count
  ami           = "ami-0c7217cdde317cfec"
  instance_type = var.instance_config.instance_type
  monitoring    = var.instance_config.monitoring

  tags = {
    Region = var.instance_config.region
  }
}

```

✅ **Why objects?**

- Group related configs (e.g., per-env settings)
- Self-documenting (keys = clear intent)
- Safer than `map(any)` (enforces field types)

---

## 🚫 Special Types: `null` & `any`

| Type | When to Use | Example | Warning |
| --- | --- | --- | --- |
| `null` | Optional/unset values | `default = null` | Use `coalesce(var.x, "default")` to handle |
| `any` | Legacy configs (avoid!) | `type = any` | ❌ No validation — defeats the purpose! |

### Safe `null` Handling:

```hcl
variable "optional_tag" {
  type    = string
  default = null
}

# Use coalesce() to provide fallback
tags = {
  Optional = coalesce(var.optional_tag, "not-set")
}

```

---

## 🧪 Lab: Test Type Constraints

### 🔧 Commands to Try:

```bash
# 1. Test invalid string → number
terraform plan -var="instance_count=two"

# 2. Test set duplicate removal
terraform console
> toset(["a", "b", "a"])
# → [ "a", "b" ]

# 3. Test tuple type enforcement
terraform plan -var='ingress_rule=[80, "http", "invalid"]'
# → ❌ "Invalid value for ingress_rule[2]: number required."

```

---

## ❓ FAQ (Day 7 Edition)

**Q: When should I use `list` vs `set`?**

A:

- `list`: Order matters (e.g., AZ priority), duplicates allowed
- `set`: Uniqueness required (e.g., region lists), order irrelevant

**Q: Can I use `map(any)`?**

A: ❌ Avoid! Use `object({ ... })` for structured data — `map(any)` bypasses validation.

**Q: Why can’t I access `set` with `[0]`?**

A: Sets are *unordered* — `[0]` implies order. Convert to `list` first if needed.

---

## ➡️ Summary: Type Constraints = Config Integrity

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ `type = any` | ✅ `type = string` / `object({ ... })` |
| ❌ Hardcoded lists | ✅ `list(string)` + validation |
| ❌ Duplicate regions | ✅ `set(string)` for uniqueness |
| ❌ Unnamed tuples | ✅ `object({region=string})` for clarity |

> 🎯 Golden Rule:
> 
> 
> **“If your config accepts `123` for a region — it’s broken by design.”**
>