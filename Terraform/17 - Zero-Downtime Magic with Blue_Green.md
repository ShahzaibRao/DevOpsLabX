# **1.17: Mini-Project #5 — Blue/Green Deployments (Elastic Beanstalk + Terraform)**

🎯 **Goal**: Deploy app versions to *identical environments* → swap DNS → **zero-downtime releases**.

> ✅ **You’ll build**:  
> - ✅ **Blue environment** (v1.0 production)  
> - ✅ **Green environment** (v2.0 staging)  
> - ✅ **S3-backed app versions** (zip artifacts)  
> - ✅ **One-click swap** (Route 53 CNAME swap)  
> - ✅ **Rollback in seconds** (swap back if v2.0 fails)  

---

## 🧠 Why Blue/Green? (Not Just Theory)

| Deployment Strategy | Downtime | Risk | Use Case |
|---------------------|----------|------|----------|
| ❌ In-Place (EC2) | High (minutes) | Critical (rollback = manual) | Legacy apps |
| ❌ Rolling Update | Medium (seconds) | Medium (partial traffic) | Stateless apps |
| ✅ **Blue/Green** | **None** (swap in <30s) | **Low** (rollback = 1 click) | **Production APIs, E-commerce** |

> 💡 **Golden Rule**:  
> **“If your app can’t tolerate 5 seconds of downtime — Blue/Green is non-negotiable.”**

---

## 📦 Architecture Diagram

```mermaid
flowchart LR
  subgraph BEFORE_SWAP [Before Swap]
    A[Users] -->|blue.example.com| B[Blue Env (v1.0)]
    A -->|green.example.com| C[Green Env (v2.0)]
    B --> D[(S3: app-v1.zip)]
    C --> E[(S3: app-v2.zip)]
  end

  subgraph AFTER_SWAP [After Swap]
    A -->|blue.example.com| C
    A -->|green.example.com| B
  end

  Click[Swap CNAME] --> AFTER_SWAP
```

✅ **Critical Flow**:  
1. Deploy v1.0 to **Blue** → users → `blue.example.com`  
2. Deploy v2.0 to **Green** (identical infra, no traffic)  
3. Test v2.0 (smoke tests, perf, security scans)  
4. **Swap CNAMEs** → traffic instantly shifts to v2.0  
5. **Rollback?** Swap again → back to v1.0 in <30s  

---

## ✏️ Hands-On: Terraform Implementation

### 🔹 File Structure (`/day17/`)
```
day17/
├── main.tf             # Shared resources (S3, IAM roles, EB app)
├── blue_env.tf         # Blue environment (v1.0)
├── green_env.tf        # Green environment (v2.0)
├── package-apps.sh     # 📦 Build app zip (v1, v2)
└── TASK.md             # 📝 Your challenge (Canary, CloudWatch alarms)
```

---

### 1️⃣ `main.tf` — Shared Infrastructure

#### 📦 S3 Bucket (App Artifacts)
```hcl
resource "aws_s3_bucket" "app_versions" {
  bucket = "eb-app-versions-${random_id.suffix.hex}"
}

# 🔒 Block ALL public access
resource "aws_s3_bucket_public_access_block" "app" {
  bucket = aws_s3_bucket.app_versions.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 📥 Upload app versions (v1, v2)
resource "aws_s3_object" "v1" {
  bucket = aws_s3_bucket.app_versions.id
  key    = "app-v1.zip"
  source = "${path.module}/app-v1.zip"
}

resource "aws_s3_object" "v2" {
  bucket = aws_s3_bucket.app_versions.id
  key    = "app-v2.zip"
  source = "${path.module}/app-v2.zip"
}
```

#### 🔑 IAM Roles (Least Privilege)
```hcl
# EC2 instances in EB need S3 access
resource "aws_iam_instance_profile" "eb" {
  name = "eb-instance-profile"
  role = aws_iam_role.ec2.name
}

resource "aws_iam_role" "ec2" {
  name = "eb-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

# Attach managed policy (no inline policies!)
resource "aws_iam_role_policy_attachment" "s3" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}
```

#### ☁️ Elastic Beanstalk Application
```hcl
resource "aws_elastic_beanstalk_application" "demo" {
  name        = "blue-green-demo"
  description = "Demo for zero-downtime deployments"
}
```

---

### 2️⃣ `blue_env.tf` — Production (v1.0)

```hcl
resource "aws_elastic_beanstalk_environment" "blue" {
  name                = "blue-env"
  application         = aws_elastic_beanstalk_application.demo.name
  solution_stack_name = "64bit Amazon Linux 2 v5.8.4 running Python 3.8"

  # 📦 Deploy v1.0
  version_label = "v1.0"
  settings {
    namespace = "aws:elasticbeanstalk:application:environment"
    name      = "APP_VERSION"
    value     = "1.0"
  }

  # 🌐 DNS: blue.example.com
  settings {
    namespace = "aws:elasticbeanstalk:environment:domain"
    name      = "CNAME"
    value     = "blue"
  }

  # 📊 Health checks
  setting {
    namespace = "aws:elasticbeanstalk:application"
    name      = "Application Healthcheck URL"
    value     = "/health"
  }

  tags = merge(var.tags, { Environment = "blue", Role = "production" })
}
```

---

### 3️⃣ `green_env.tf` — Staging (v2.0)

```hcl
resource "aws_elastic_beanstalk_environment" "green" {
  name                = "green-env"
  application         = aws_elastic_beanstalk_application.demo.name
  solution_stack_name = "64bit Amazon Linux 2 v5.8.4 running Python 3.8"

  # 📦 Deploy v2.0
  version_label = "v2.0"
  settings {
    namespace = "aws:elasticbeanstalk:application:environment"
    name      = "APP_VERSION"
    value     = "2.0"
  }

  # 🌐 DNS: green.example.com
  settings {
    namespace = "aws:elasticbeanstalk:environment:domain"
    name      = "CNAME"
    value     = "green"
  }

  tags = merge(var.tags, { Environment = "green", Role = "staging" })
}
```

---

### 4️⃣ `package-apps.sh` — Build Artifacts

```bash
#!/bin/bash
# 📦 Build v1.0 (stable)
mkdir -p app-v1 && echo "v1.0" > app-v1/index.html
zip -r app-v1.zip app-v1 && rm -rf app-v1

# 📦 Build v2.0 (new features)
mkdir -p app-v2 && echo "v2.0 - NEW FEATURE!" > app-v2/index.html
zip -r app-v2.zip app-v2 && rm -rf app-v2

echo "✅ Artifacts built: app-v1.zip, app-v2.zip"
```

> 💡 **Pro Tip**:  
> In prod, replace this with CI/CD (GitHub Actions → build → push to S3).

---

## 🧪 Lab: Deploy & Swap

### 🔧 Commands to Run:
```bash
# 1. Build artifacts
./package-apps.sh

# 2. Deploy
terraform init
terraform apply -auto-approve

# 3. Verify environments
echo "Blue URL: $(terraform output -raw blue_url)"
echo "Green URL: $(terraform output -raw green_url)"
# → https://blue-env...elasticbeanstalk.com
# → https://green-env...elasticbeanstalk.com
```

### ✅ Expected Output:
| Environment | URL | Content |
|-------------|-----|---------|
| **Blue** | `blue.example.com` | `"v1.0 - Stable"` |
| **Green** | `green.example.com` | `"v2.0 - NEW FEATURE!"` |

---

### 🔀 Perform the Swap (3 Methods)

#### Method 1: AWS Console (Simplest)
1. Go to **Elastic Beanstalk → Environments**  
2. Select **Blue Env → Actions → Swap Environment CNAMEs**  
3. Choose **Green Env → Swap**  

#### Method 2: CLI (Automation-Friendly)
```bash
aws elasticbeanstalk swap-environment-cnames \
  --source-environment-name blue-env \
  --destination-environment-name green-env
```

#### Method 3: Terraform (Idempotent)
```hcl
# Add to green_env.tf
lifecycle {
  create_before_destroy = true
}

# Then run:
terraform apply -target=aws_elastic_beanstalk_environment.green
```

> ✅ **After Swap**:  
> - `blue.example.com` → v2.0  
> - `green.example.com` → v1.0  

---

## 🚀 Challenge: Complete the Project (TASK.md)

### 📝 Your Mission:
| Task | Why It Matters |
|------|----------------|
| ✅ **Add CloudWatch Alarms** | Auto-rollback on 5xx errors (e.g., `HTTPCode_Backend_5XX > 1%`) |
| ✅ **Canary Deployment** | Shift 5% traffic to Green → monitor → full swap |
| ✅ **S3 Versioning + Lifecycle** | Auto-delete old app versions after 30 days |
| ✅ **Terraform State Locking** | Prevent concurrent swaps (S3 + DynamoDB) |
| ✅ **Custom Domain (Route 53)** | `app.example.com` → Blue/Green CNAMEs |

> 💡 **Pro Tip**:  
> Use `aws_cloudwatch_metric_alarm` + `aws_sns_topic` for auto-rollback:  
> ```hcl
> alarm_actions = ["arn:aws:automate:us-east-1:elasticbeanstalk:environment:swap"]
> ```

---

## ⚠️ Critical Gotchas (From Your Video!)

| Issue | Fix |
|-------|-----|
| ❌ Swap fails (CNAME in use) | ✅ Wait 60s between swaps (Route 53 TTL) |
| ❌ v2.0 breaks prod | ✅ **Always test Green first** (smoke tests, load tests) |
| ❌ S3 bucket not versioned | ✅ Enable versioning + lifecycle rules (Day 17 TASK.md) |
| ❌ IAM roles missing S3 access | ✅ Attach `AmazonS3ReadOnlyAccess` (never `S3FullAccess`) |

---

## ❓ FAQ (Day 17 Edition)

**Q: Can I do Blue/Green without Elastic Beanstalk?**  
A: ✅ **Yes — better with ALB + Auto Scaling Groups**:  
- Blue: `target_group_arn_blue`  
- Green: `target_group_arn_green`  
- Swap: Update ALB listener rule to point to Green  

**Q: How to handle database migrations?**  
A: ✅ **Forward-compatible schema changes**:  
- Deploy v1.1 (adds column, but v1.0 ignores it)  
- Swap → v1.1 live  
- Deploy v2.0 (uses new column)  

**Q: Cost of running 2 environments?**  
A: ✅ **Green can be scaled to 0 instances** (warm pool only):  
```hcl
# In green_env.tf
setting {
  namespace = "aws:autoscaling:asg"
  name      = "MinSize"
  value     = "0"  # Scale to 0 when idle
}
```

---

## ➡️ Summary: Blue/Green Done Right

| Anti-Pattern | Production Practice |
|--------------|---------------------|
| ❌ Deploy to prod directly | ✅ Test identical Green first |
| ❌ Manual DNS changes | ✅ EB CNAME swap / ALB rule update |
| ❌ No rollback plan | ✅ Swap back in <30s |
| ❌ Shared DB with breaking changes | ✅ Forward-compatible migrations |

> 🎯 **Golden Rule**:  
> **“If your deployment requires a maintenance window — it’s not Blue/Green.”**

---