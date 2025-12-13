# **1.16 : Mini-Project #3 — IAM User Management (CSV-Driven, Terraform-Managed)**

🎯 **Goal**: Automate IAM user onboarding — from HR CSV → AWS users, groups, console access — *zero manual steps*.

> ✅ You’ll build:
> 
> - ✅ **26 IAM users** (The Office characters — scalable to 10,000+)
> - ✅ **Dynamic groups** (Education, Managers, Engineers)
> - ✅ **Conditional group membership** (e.g., `JobTitle` contains "Manager")
> - ✅ **Console access** with password reset enforcement
> - ✅ **Tags for audit/search** (`DisplayName`, `Department`, `JobTitle`)

---

## 🧠 Why This Matters (Beyond The Office Jokes)

| Manual IAM Onboarding | Terraform-Managed IAM |
| --- | --- |
| ❌ HR → Email → Ticket → IAM Admin → 3-day delay | ✅ HR → CSV → `terraform apply` → 30 seconds |
| ❌ Inconsistent usernames (`Michael`, `mscott`, `michael-scott`) | ✅ Standardized: `mscott` → `[first_initial][last_name]` |
| ❌ No audit trail for user creation | ✅ Git-tracked CSV + Terraform state |
| ❌ Passwords shared insecurely (Slack/email) | ✅ Auto-generated + reset-on-first-login |

> 💡 Golden Rule:
> 
> 
> **“If your identity system isn’t version-controlled — it’s not production-ready.”**
> 

---

## 📦 Architecture Diagram

```mermaid
flowchart LR
  A[HR System<br/>(users.csv)] -->|Git| B[Terraform Repo]
  B --> C[Terraform Plan/Apply]
  C --> D[AWS IAM]
  D --> E[Users: mscott, dschrute...]
  D --> F[Groups: Education, Managers, Engineers]
  D --> G[Memberships: Dwight → Managers]
  D --> H[Login Profiles: Reset required]

```

✅ **Critical Flow**:

1. HR updates `users.csv` (adds `jdoe,Engineering,Software Engineer`)
2. Git commit → CI/CD triggers `terraform apply`
3. Terraform:
    - Creates `jdoe` IAM user
    - Adds to `Engineers` group
    - Enables console access
4. New hire gets email: *“Your AWS account is ready: https://[account].signin.aws.amazon.com/console”*

---

## ✏️ Hands-On: Terraform Implementation

### 🔹 File Structure (`/day16/`)

```
day16/
├── backend.tf        # S3 remote state (encrypted, versioned)
├── provider.tf       # AWS provider (locked version)
├── users.csv         # 📊 HR data source (26 users)
├── main.tf           # User creation + CSV parsing
├── groups.tf         # Groups + dynamic membership
└── TASK.md           # 📝 Your challenge (MFA, SSO, HR integration)

```

---

### 1️⃣ `users.csv` — Single Source of Truth

```
first_name,last_name,department,job_title
Michael,Scott,Education,Regional Manager
Dwight,Schrute,Sales,Assistant to the Regional Manager
Jim,Halpert,Sales,Sales Representative
Pam,Beesly,Reception,Receptionist
Ryan,Howard,Temps,Temp
# ... 22 more

```

> ⚠️ Security Note:
> 
> 
> Never commit CSV with sensitive data (emails, IDs) to Git. Use:
> 
> - AWS SSM Parameter Store
> - HashiCorp Vault
> - S3 + KMS encryption (covered in Day 20)

---

### 2️⃣ `main.tf` — User Creation (CSV → IAM)

### 🔍 Parse CSV → List of Maps

```hcl
locals {
  # 📊 Convert CSV → Terraform list of maps
  users = csvdecode(file("${path.module}/users.csv"))
}

```

### 👤 Create IAM Users (Standardized Naming)

```hcl
resource "aws_iam_user" "users" {
  # 🔁 For each user in CSV
  for_each = { for u in local.users : "${u.first_name}-${u.last_name}" => u }

  # ✅ Standardized username: mscott, dschrute
  name = lower("${substr(each.value.first_name, 0, 1)}${each.value.last_name}")

  path = "/users/"

  # 🔖 Tag for audit/search (HR data)
  tags = {
    DisplayName = "${each.value.first_name} ${each.value.last_name}"
    Department  = each.value.department
    JobTitle    = each.value.job_title
  }
}

```

> 💡 Why for_each over count?
> 
> - `mscott[0]` → `mscott["Michael-Scott"]` (human-readable keys)
> - Delete `Ryan` from CSV → *only Ryan is destroyed* (no index drift!)

---

### 3️⃣ `main.tf` — Console Access (Secure Onboarding)

```hcl
resource "aws_iam_user_login_profile" "users" {
  for_each = aws_iam_user.users

  user                    = each.value.name
  password_reset_required = true  # 🔑 Force reset on first login
  # ⚠️ AWS won't return passwords without PGP → use CLI/S3 for secrets (see TASK.md)
}

```

✅ **Security Best Practice**:

> password_reset_required = true → Complies with AWS IAM Security Best Practices.
> 

---

### 4️⃣ `groups.tf` — Dynamic Group Membership

### 👥 Create Groups

```hcl
resource "aws_iam_group" "education" {
  name = "Education"
  path = "/groups/"
}

resource "aws_iam_group" "managers" {
  name = "Managers"
  path = "/groups/"
}

```

### 🔗 Assign Users Conditionally

```hcl
# 🎯 Education group: Users with Department = "Education"
resource "aws_iam_group_membership" "education" {
  name  = "education-members"
  group = aws_iam_group.education.name
  users = [
    for u in aws_iam_user.users : u.name
    if u.tags.Department == "Education"  # ← Filter by tag!
  ]
}

# 🎯 Managers group: Users with "Manager" in JobTitle
resource "aws_iam_group_membership" "managers" {
  name  = "managers-members"
  group = aws_iam_group.managers.name
  users = [
    for u in aws_iam_user.users : u.name
    if can(regex("Manager|CEO", u.tags.JobTitle))  # ← Safe regex check
  ]
}

```

> 💡 Why can(regex(...))?
> 
> - `regex("Manager", u.tags.JobTitle)` → **crashes** if no match
> - `can(regex(...))` → returns `true`/`false` → safe for filters

---

## 🧪 Lab: Deploy & Validate

### 🔧 Commands to Run:

```bash
# 1. Initialize (S3 backend)
terraform init

# 2. Preview (26 users + 3 groups + 26 login profiles)
terraform plan | grep "will be created"

# 3. Deploy
terraform apply -auto-approve

# 4. Verify outputs
terraform output user_names
# → ["mscott", "dschrute", "jhalpert", ...]

terraform output -json user_passwords | jq '.[] | select(.value != "sensitive")'
# → Empty (passwords masked — see TASK.md for secure handling)

```

### ✅ Verify in AWS Console:

1. **IAM → Users**:
    - `mscott` → Tags: `Department=Education`, `JobTitle=Regional Manager`
    - `dschrute` → **Groups: Managers** (JobTitle contains "Manager")
2. **IAM → Groups**:
    - `Managers` → Members: `mscott`, `dschrute`, `dwallace`
3. **User Login**:
    - `mscott` → Security Credentials → **Console password required**

---

## 🚀 Challenge: Complete the Project ([TASK.md](http://task.md/))

### 📝 Your Mission:

| Task | Why It Matters |
| --- | --- |
| ✅ **Secure Password Handling** | Store passwords in AWS Secrets Manager (not outputs!) |
| ✅ **MFA Enforcement** | Add `aws_iam_account_password_policy` + `mfa_delete` |
| ✅ **AWS SSO Integration** | Replace IAM users with SSO groups (production standard) |
| ✅ **HR System Sync** | Automate CSV updates from Workday/ADP via Lambda |
| ✅ **Policy Attachments** | Add `ReadOnlyAccess` to Engineers, `PowerUser` to Managers |

> 💡 Pro Tip:
> 
> 
> Use `aws_iam_policy_document` + `aws_iam_group_policy_attachment` for least-privilege policies.
> 

---

## ⚠️ Critical Gotchas (From Your Video!)

| Issue | Fix |
| --- | --- |
| ❌ `password = aws_iam_user_login_profile.users.password` | ✅ Never output passwords — use Secrets Manager |
| ❌ `user.tags.department` (lowercase) | ✅ `user.tags.Department` (tags are case-sensitive!) |
| ❌ CSV path errors (`users.csv` vs `./users.csv`) | ✅ Use `file("${path.module}/users.csv")` |
| ❌ `regex()` crashes on no-match | ✅ Always wrap in `can(regex(...))` |

---

## ❓ FAQ (Day 16 Edition)

**Q: How to import *existing* IAM users into Terraform?**

A: Use `terraform import`:

```bash
terraform import 'aws_iam_user.users["Michael-Scott"]' mscott

```

**Q: Can I add email/phone to the CSV?**

A: ✅ Yes — but store securely:

```
first_name,last_name,email,department
Michael,Scott,michael@dundermifflin.com,Education

```

→ Use `aws_ssm_parameter` to store PII:

```hcl
resource "aws_ssm_parameter" "user_email" {
  for_each = aws_iam_user.users
  name  = "/iam/users/${each.value.name}/email"
  value = lookup(each.value, "email", "")
  type  = "SecureString"
}

```

**Q: What if HR deletes a user from CSV?**

A: Terraform **destroys the IAM user** → use `lifecycle { prevent_destroy = true }` for prod.

---

## ➡️ Summary: IAM Automation Done Right

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ Manual user creation | ✅ CSV-driven `for_each` |
| ❌ Inconsistent usernames | ✅ Standardized naming (`${first_initial}${last_name}`) |
| ❌ Passwords in Slack | ✅ Secrets Manager + reset-on-first-login |
| ❌ Static group assignments | ✅ Dynamic `for` expressions (`if u.tags.JobTitle == "..."`) |

> 🎯 Golden Rule:
> 
> 
> **“Your HR system is the source of truth — Terraform is the enforcer.”**
>