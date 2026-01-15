# **1.18 : Mini-Project #4 — Serverless Image Processor (Lambda + S3 Events)**

---

🔗 **Repo**: [`github.com/Push/terraform-aws-labs/day18`](https://github.com/Push/terraform-aws-labs/tree/main/day18)

🎯 **Goal**: Build a *fully automated* image pipeline — upload → process → 5 variants — **zero servers, zero manual steps**.

> ✅ You’ll build:
> 
> - ✅ **Upload S3 bucket** (source) + **Processed S3 bucket** (destination)
> - ✅ **Lambda function** (Python + Pillow) — compress, resize, convert formats
> - ✅ **S3 event trigger** — auto-invoke Lambda on `s3:ObjectCreated:*`
> - ✅ **Least-privilege IAM roles** — no `S3FullAccess`!
> - ✅ **CloudWatch logging** — track cold starts, errors, durations

---

## 🧠 Why Serverless? (The “It Works on My Machine” Fix)

| Traditional EC2 | Serverless Lambda |
| --- | --- |
| ❌ Servers run 24/7 → pay even when idle | ✅ Pay per *millisecond* of execution |
| ❌ Manual scaling (ASG configs, ALB rules) | ✅ Auto-scales to 1,000+ concurrent invocations |
| ❌ OS patching, security updates | ✅ AWS manages OS/runtime |
| ❌ “It works on my Mac” → breaks in prod | ✅ ✅ **Docker-based layer builds** (Day 18’s key fix!) |

> 💡 Golden Rule:
> 
> 
> **“If your workload is event-driven and <15 mins — Lambda is your friend.”**
> 

---

## 📦 Architecture Diagram

```mermaid
flowchart LR
  A[Upload Image<br/>to S3] -->|s3:ObjectCreated:*| B[Lambda Trigger]
  B --> C[Lambda Function<br/>(Python + Pillow)]
  C --> D1[Compressed JPEG<br/>(85% quality)]
  C --> D2[Low-Quality JPEG<br/>(60% quality)]
  C --> D3[WebP<br/>(85% quality)]
  C --> D4[PNG<br/>(lossless)]
  C --> D5[Thumbnail<br/>(200x200)]
  D1 --> E[Processed S3 Bucket]
  D2 --> E
  D3 --> E
  D4 --> E
  D5 --> E
  C --> F[CloudWatch Logs]

```

✅ **Critical Flow**:

1. User uploads `photo.jpg` to `upload-bucket-dev`
2. S3 emits `s3:ObjectCreated:Put` event
3. Lambda invoked with event payload:
    
    ```json
    {"Records": [{"s3": {"bucket": {"name": "upload-bucket-dev"}, "object": {"key": "photo.jpg"}}}]}
    
    ```
    
4. Lambda:
    - Downloads `photo.jpg` from S3
    - Processes with Pillow (5 variants)
    - Uploads to `processed-bucket-dev`
5. Logs duration/memory to CloudWatch

---

## ✏️ Hands-On: Terraform Implementation

### 🔹 File Structure (`/day18/`)

```
day18/
├── main.tf             # Core resources (S3, Lambda, IAM)
├── lambda/             # 📦 Lambda code (Python + Docker layer)
│   ├── lambda_function.py
│   └── build_layer.sh  # ✅ Docker-based build (fixes "it works on my machine"!)
├── scripts/
│   ├── deploy.sh       # One-click deploy
│   └── destroy.sh      # Safe cleanup
└── TASK.md             # 📝 Your challenge (error handling, VPC, cost optimization)

```

---

### 1️⃣ `main.tf` — Infrastructure as Code

### 🪣 S3 Buckets (Secure by Default)

```hcl
# 🔒 Upload bucket (source)
resource "aws_s3_bucket" "upload" {
  bucket = "${local.bucket_prefix}-upload-${random_id.suffix.hex}"
}

resource "aws_s3_bucket_versioning" "upload" {
  bucket = aws_s3_bucket.upload.id
  versioning_configuration { status = "Enabled" }
}

# 🔒 Processed bucket (destination)
resource "aws_s3_bucket" "processed" {
  bucket = "${local.bucket_prefix}-processed-${random_id.suffix.hex}"
}

# 🛡️ Block ALL public access
resource "aws_s3_bucket_public_access_block" "all" {
  for_each = toset([aws_s3_bucket.upload.id, aws_s3_bucket.processed.id])
  bucket                  = each.value
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

```

### 🔑 IAM Role (Least Privilege)

```hcl
# ✅ Lambda execution role
resource "aws_iam_role" "lambda" {
  name = "${local.lambda_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

# 🔐 Granular permissions (no S3FullAccess!)
resource "aws_iam_role_policy" "lambda" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # 📝 CloudWatch logs
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      # 📥 Get from upload bucket
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:GetObjectVersion"]
        Resource = "${aws_s3_bucket.upload.arn}/*"
      },
      # 📤 Put to processed bucket
      {
        Effect = "Allow"
        Action = ["s3:PutObject", "s3:PutObjectAcl"]
        Resource = "${aws_s3_bucket.processed.arn}/*"
      }
    ]
  })
}

```

### ☁️ Lambda Function (Event-Driven)

```hcl
# 📦 Docker-built layer (Pillow 10.4.0)
resource "aws_lambda_layer_version" "pillow" {
  filename         = "${path.module}/lambda/pillow-layer.zip"
  layer_name       = "pillow-10-4-0"
  compatible_runtimes = ["python3.12"]
}

# 🖼️ Image processor
resource "aws_lambda_function" "image_processor" {
  function_name = local.lambda_name
  role          = aws_iam_role.lambda.arn
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.12"
  timeout       = 60
  memory_size   = 1024

  filename         = "${path.module}/lambda/lambda_function.zip"
  source_code_hash = data.archive_file.lambda.output_base64sha256
  layers           = [aws_lambda_layer_version.pillow.arn]

  environment {
    variables = {
      PROCESSED_BUCKET = aws_s3_bucket.processed.id
      LOG_LEVEL        = "INFO"
    }
  }
}

```

### ⚡ S3 Event Trigger (Auto-Invoke)

```hcl
# 🔗 S3 → Lambda permission
resource "aws_lambda_permission" "s3_trigger" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.image_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.upload.arn
}

# 📡 Enable S3 event notifications
resource "aws_s3_bucket_notification" "upload" {
  bucket = aws_s3_bucket.upload.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.image_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_suffix       = ".jpg,.jpeg,.png,.webp"
  }
}

```

---

### 2️⃣ `lambda/lambda_function.py` — Image Processing Logic

```python
import boto3
from PIL import Image
import io
import logging
import os

# 📝 Configurable variants
VARIANTS = {
    "compressed": {"format": "JPEG", "quality": 85},
    "low":        {"format": "JPEG", "quality": 60},
    "webp":       {"format": "WEBP", "quality": 85},
    "png":        {"format": "PNG", "quality": None},
    "thumbnail":  {"format": "JPEG", "size": (200, 200)},
}

def lambda_handler(event, context):
    logger = logging.getLogger()
    logger.setLevel(os.getenv("LOG_LEVEL", "INFO"))

    s3 = boto3.client("s3")
    bucket = event["Records"][0]["s3"]["bucket"]["name"]
    key = event["Records"][0]["s3"]["object"]["key"]

    logger.info(f"Processing {key} from {bucket}")

    # 📥 Download original
    response = s3.get_object(Bucket=bucket, Key=key)
    img = Image.open(io.BytesIO(response["Body"].read()))

    # 🖼️ Process variants
    for name, config in VARIANTS.items():
        buffer = io.BytesIO()
        processed = img.copy()

        # Resize for thumbnail
        if "size" in config:
            processed.thumbnail(config["size"])

        # Save with quality
        save_kwargs = {"format": config["format"]}
        if config.get("quality"):
            save_kwargs["quality"] = config["quality"]

        processed.save(buffer, **save_kwargs)
        buffer.seek(0)

        # 📤 Upload to processed bucket
        dest_key = f"{os.path.splitext(key)[0]}_{name}.jpg"
        s3.put_object(
            Bucket=os.getenv("PROCESSED_BUCKET"),
            Key=dest_key,
            Body=buffer,
            ContentType=f"image/{config['format'].lower()}"
        )
        logger.info(f"Uploaded {dest_key}")

    return {"statusCode": 200, "body": "Success!"}

```

---

### 3️⃣ `lambda/build_layer.sh` — **Docker-Based Layer Build** (The Fix!)

```bash
#!/bin/bash
# ✅ Solves "it works on my Mac" by building in AWS Lambda runtime
docker run -v "$PWD":/var/task "public.ecr.aws/sam/build-python3.12" /bin/sh -c "
  pip install Pillow==10.4.0 -t python/lib/python3.12/site-packages/
  cd python
  zip -r ../pillow-layer.zip .
"
mv pillow-layer.zip ../

```

> 💡 Why Docker?
> 
> - Lambda runs on **Amazon Linux 2**
> - Mac/Windows binaries ≠ Linux binaries → `ImportError: libjpeg.so.9`
> - Docker builds in the *exact* runtime environment → no more cold-start fails!

---

## 🧪 Lab: Deploy & Validate

### 🔧 Commands to Run:

```bash
# 1. Build layer (Docker required)
./lambda/build_layer.sh

# 2. Deploy
./scripts/deploy.sh
# → Outputs: upload_bucket, processed_bucket, lambda_function

# 3. Upload test image
aws s3 cp test.jpg "s3://$(terraform output -raw upload_bucket)/"

# 4. Verify outputs
aws s3 ls "s3://$(terraform output -raw processed_bucket)/"
# → test_compressed.jpg, test_low.jpg, test_webp.webp, ...

```

### ✅ Expected CloudWatch Logs:

```
INFO: Processing photo.jpg from upload-bucket-dev
INFO: Uploaded photo_compressed.jpg
INFO: Uploaded photo_low.jpg
INFO: Uploaded photo_webp.webp
INFO: Uploaded photo_png.png
INFO: Uploaded photo_thumbnail.jpg
REPORT RequestId: abc123 Duration: 2543.21 ms Billed Duration: 2544 ms Memory Size: 1024 MB Max Memory Used: 116 MB

```

---

## 🚀 Challenge: Complete the Project ([TASK.md](http://task.md/))

### 📝 Your Mission:

| Task | Why It Matters |
| --- | --- |
| ✅ **Add Error Handling** | Handle invalid images (corrupt files, unsupported formats) |
| ✅ **VPC Lambda** | Deploy Lambda in private subnets (for HIPAA/GDPR workloads) |
| ✅ **Cost Optimization** | Tune `memory_size`/`timeout` — 512MB may be enough |
| ✅ **Dead Letter Queue** | Capture failed invocations (SNS/SQS) |
| ✅ **Custom Domains** | API Gateway + CloudFront for `https://cdn.example.com/photo.jpg` |

> 💡 Pro Tip:
> 
> 
> Use `aws lambda update-function-configuration` to tune memory *without* redeploying code.
> 

---

## ⚠️ Critical Gotchas (From Your Video!)

| Issue | Fix |
| --- | --- |
| ❌ `ImportError: libjpeg.so.9` | ✅ **Docker-based layer builds** (never build locally!) |
| ❌ S3 buckets not emptying | ✅ `destroy.sh` must delete *all versions* (enable versioning cleanup) |
| ❌ Lambda timeout (60s) | ✅ Increase `timeout` in `main.tf` for large images |
| ❌ Cold starts (470ms) | ✅ Provisioned Concurrency (Day 25!) |

---

## ❓ FAQ (Day 18 Edition)

**Q: How to handle images >10MB?**

A: Use **S3 presigned URLs** + Lambda proxy integration — avoid Lambda’s 6MB payload limit.

**Q: Can I add EXIF stripping?**

A: ✅ Yes — add `img.info.clear()` before `processed.save()`.

**Q: Why not use AWS Batch for large images?**

A: ✅ **Hybrid pattern**:

- Lambda for <5MB images (fast, cheap)
- Batch for >5MB (no timeout limits)
→ Use S3 event filtering: `filter_prefix = "small/"` vs `"large/"`

---

## ➡️ Summary: Serverless Done Right

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ EC2 + cron jobs | ✅ Lambda + S3 events |
| ❌ Local layer builds | ✅ Docker-based builds |
| ❌ `S3FullAccess` policies | ✅ Least-privilege IAM |
| ❌ No logging | ✅ CloudWatch + structured JSON |

> 🎯 Golden Rule:
> 
> 
> **“If your Lambda layer isn’t built in Docker — it’s broken by design.”**
>