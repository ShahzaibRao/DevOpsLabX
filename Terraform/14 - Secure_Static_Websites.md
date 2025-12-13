# **1.14 : Mini-Project #1 — Static Website (S3 + CloudFront)**

🎯 **Goal**: Deploy a *secure, fast, globally cached* static site — **no console clicks, no hardcoded IDs**.

> ✅ You’ll build:
> 
> - ✅ **Private S3 bucket** (no public exposure!)
> - ✅ **CloudFront Origin Access Control (OAC)** — modern, secure auth
> - ✅ **Bucket policy** — least-privilege access for CloudFront
> - ✅ **Auto-upload** of `index.html`, `style.css`, `script.js`
> - ✅ **HTTPS-ready** distribution (default + ACM-ready)

---

## 🧠 Why This Architecture? (The *Right* Way)

| Approach | Risk | Cost | Performance | Security |
| --- | --- | --- | --- | --- |
| ❌ Public S3 (`s3-website-us-east-1.amazonaws.com`) | DDoS, data leaks | High (global data transfer) | Slow (no caching) | ❌ Open to internet |
| ✅ **S3 + CloudFront (Terraform)** | None | Low (edge caching) | ⚡ Fast (TTL, edge POPs) | ✅ Private bucket + OAC |

> 🔑 Key Insight:
> 
> 
> **CloudFront ≠ CDN** — it’s your *security control plane*:
> 
> - S3 bucket stays **private**
> - Users hit edge locations (Mumbai, Virginia, Sydney…)
> - Requests never reach S3 unless cache miss

---

## 📦 Architecture Diagram

```mermaid
flowchart LR
  A[User in India] -->|HTTPS| B[CloudFront Edge: Mumbai]
  C[User in US] -->|HTTPS| D[CloudFront Edge: Virginia]
  B -->|Cache HIT| A
  B -->|Cache MISS| E[S3 Bucket<br/>(private, us-east-1)]
  D -->|Cache HIT| C
  D -->|Cache MISS| E
  E -.->|OAC Auth| B & D

```

✅ **Critical Components**:

- **S3 Bucket**: Private, no public ACLs
- **Origin Access Control (OAC)**: Replaces deprecated OAI — AWS-managed identity
- **Bucket Policy**: Grants `s3:GetObject` *only* to CloudFront (via ARN condition)
- **CloudFront Distribution**:
    - `default_root_object = "index.html"`
    - `viewer_certificate = cloudfront_default_certificate` (or ACM later)
    - `price_class = "PriceClass_100"` (US/EU/CA only — cheaper!)

---

## ✏️ Hands-On: Terraform Implementation

### 🔹 File Structure (`/day14/`)

```
day14/
├── main.tf           # S3 + CloudFront resources
├── locals.tf         # Reusable values (origin_id)
├── variables.tf      # Inputs (bucket_name, environment)
├── www/              # 📁 Your static files
│   ├── index.html
│   ├── style.css
│   └── script.js
└── TASK.md           # 📝 Your challenge (ACM, Route 53, CI/CD)

```

---

### 1️⃣ `main.tf` — Core Resources

### 🪣 **Private S3 Bucket**

```hcl
resource "aws_s3_bucket" "website" {
  bucket = var.bucket_name
}

# 🔒 Block *all* public access
resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

```

### 🔑 **Origin Access Control (OAC)**

```hcl
# ✅ Modern replacement for OAI
resource "aws_cloudfront_origin_access_control" "website" {
  name                              = "${var.bucket_name}-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

```

### 📜 **Bucket Policy (Least Privilege)**

```hcl
resource "aws_s3_bucket_policy" "website" {
  bucket = aws_s3_bucket.website.id

  # 🔐 Only CloudFront can read objects
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCloudFront"
      Effect    = "Allow"
      Principal = { "Service" = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.website.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.website.arn
        }
      }
    }]
  })

  depends_on = [
    aws_s3_bucket_public_access_block.website,
    aws_cloudfront_origin_access_control.website
  ]
}

```

### 📤 **Auto-Upload Static Files**

```hcl
# ✅ Upload all files in ./www/
resource "aws_s3_object" "website" {
  for_each = fileset("${path.module}/www/", "**")

  bucket       = aws_s3_bucket.website.bucket
  key          = each.value
  source       = "${path.module}/www/${each.value}"
  etag         = filemd5("${path.module}/www/${each.value}")
  content_type = element(split("/", mime.types), index(split("/", mime.types), "*")) == each.value ? "application/octet-stream" : lookup({
    "html" = "text/html"
    "css"  = "text/css"
    "js"   = "application/javascript"
    "png"  = "image/png"
    "jpg"  = "image/jpeg"
  }, element(reverse(split(".", each.value)), 0), "binary/octet-stream")
}

```

### ☁️ **CloudFront Distribution**

```hcl
resource "aws_cloudfront_distribution" "website" {
  origin {
    domain_name              = aws_s3_bucket.website.bucket_regional_domain_name
    origin_access_control_id = aws_cloudfront_origin_access_control.website.id
    origin_id                = local.origin_id
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  # 🔒 Redirect HTTP → HTTPS
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = local.origin_id

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  price_class = "PriceClass_100"  # US/EU/CA only — 40% cheaper!
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  # ✅ Default cert (use ACM later — see TASK.md)
  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

```

---

### 2️⃣ `locals.tf` — Reusable Values

```hcl
locals {
  origin_id = "S3-${aws_s3_bucket.website.id}"
}

```

---

### 3️⃣ `variables.tf` — Inputs

```hcl
variable "bucket_name" {
  description = "Globally unique S3 bucket name"
  type        = string
  # ⚠️ Must match domain if using custom DNS (e.g., www.example.com)
}

variable "environment" {
  type    = string
  default = "dev"
}

```

---

## 🧪 Lab: Deploy & Validate

### 🔧 Commands to Run:

```bash
# 1. Initialize (backend + providers)
terraform init

# 2. Dry-run (catch errors early)
terraform plan

# 3. Deploy
terraform apply -auto-approve

# 4. Get CloudFront URL
terraform output distribution_domain
# → d1a2b3c4d5e6f7.cloudfront.net

```

### ✅ Verify in AWS Console:

1. **S3 Bucket**:
    - 🔒 "Block public access" = **Enabled**
    - 📜 Bucket policy = attached (only CloudFront ARN)
2. **CloudFront**:
    - 🟢 Status = **Deployed**
    - 🔐 Viewer Protocol Policy = **Redirect HTTP to HTTPS**
3. **Browser**:
    - `https://<distribution-domain>` → Your site loads!
    - `curl -I https://...` → `HTTP/2 200` + `content-type: text/html`

---

## 🚀 Challenge: Complete the Project ([TASK.md](http://task.md/))

### 📝 Your Mission:

| Task | Why It Matters |
| --- | --- |
| ✅ **Add ACM Certificate** | Enable [*yourdomain.com*](http://yourdomain.com/) (not `cloudfront.net`) |
| ✅ **Route 53 DNS** | Point `www.yourdomain.com` → CloudFront |
| ✅ **Cache Invalidation** | Terraform-triggered `aws_cloudfront_cache_invalidation` after uploads |
| ✅ **Custom Error Pages** | `404.html` served by CloudFront (not S3) |
| ✅ **Security Headers** | Add `Content-Security-Policy`, `X-Frame-Options` in CloudFront |

> 💡 Pro Tip:
> 
> 
> Use `aws_acm_certificate_validation` for DNS-validated certs — no manual console steps!
> 

---

## ⚠️ Critical Gotchas (From Your Video!)

| Issue | Fix |
| --- | --- |
| ❌ `aws_s3_bucket_object` deprecated | ✅ Use `aws_s3_object` (v4.0+) |
| ❌ `Principal = "cloudfront.amazonaws.com"` | ✅ `Principal = { "Service" = "cloudfront.amazonaws.com" }` |
| ❌ `s3:ListBucket` in bucket policy | ✅ Only `s3:GetObject` needed (OAC handles origin auth) |
| ❌ Hardcoded bucket name | ✅ Use `bucket_regional_domain_name` (avoids `s3-website` endpoints) |

---

## ❓ FAQ (Day 14 Edition)

**Q: Why `bucket_regional_domain_name` not `bucket_domain_name`?**

A: `bucket_domain_name` = `s3-website-us-east-1.amazonaws.com` → **public, insecure**.

→ `bucket_regional_domain_name` = `bucket.s3.us-east-1.amazonaws.com` → **private, OAC-compatible**.

**Q: How to force cache refresh after uploads?**

A: Add `aws_cloudfront_cache_invalidation` resource triggered by `aws_s3_object.website`:

```hcl
resource "aws_cloudfront_cache_invalidation" "website" {
  distribution_id = aws_cloudfront_distribution.website.id
  paths           = ["/*"]
  depends_on      = [aws_s3_object.website]
}

```

**Q: Can I use `for_each` for S3 objects?**

A: ✅ Yes — `fileset()` + `for_each` is the *only* scalable way (avoids `count` index drift).

---

## ➡️ Summary: Static Sites Done Right

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ Public S3 bucket | ✅ Private bucket + OAC + bucket policy |
| ❌ Manual file uploads | ✅ `aws_s3_object` + `fileset()` |
| ❌ HTTP-only | ✅ HTTPS redirect + ACM-ready config |
| ❌ No cache control | ✅ `default_ttl=3600`, `max_ttl=86400` |

> 🎯 Golden Rule:
> 
> 
> **“If your S3 bucket has `BlockPublicAccess = false` — it’s broken by design.”**
>