# **1.19 : Terraform Provisioners — Local, Remote & File (With Caveats!)**

🎯 **Goal**: Understand **provisioners** — the *last-resort tool* for bootstrapping, with clear warnings and best-practice patterns.

> ✅ **You’ll master**:  
> - ✅ `local-exec`: Run commands *on your laptop* (e.g., `curl`, `echo`)  
> - ✅ `remote-exec`: Run commands *on EC2* via SSH (e.g., `apt install nginx`)  
> - ✅ `file`: Copy files *to/from EC2* (e.g., config templates)  
> - ❗ **When to avoid them** (and use cloud-init/AMI instead)  

---

## 🧠 Why Provisioners? (The *Right* Context)

> ❌ **Anti-Pattern**:  
> ```hcl
> # DON'T do this in production!
> provisioner "remote-exec" {
>   inline = ["sudo apt install nginx", "sudo systemctl start nginx"]
> }
> ```

✅ **Legitimate Uses**:  
- 🛠️ **One-time bootstrapping** (e.g., generate certs, join cluster)  
- 📦 **Deploy immutable artifacts** (e.g., copy `app-v2.zip` → `unzip`)  
- 🔐 **Seal secrets** (e.g., `vault read` → write to `/etc/secrets`)  
- 🔁 **Post-creation validation** (e.g., `curl http://localhost:8080/health`)  

> 💡 **Golden Rule**:  
> **“If your provisioner runs >5s — bake it into an AMI instead.”**  
> *(See Day 25: Packer + AMI pipelines)*

---

## 📦 Provisioner Cheat Sheet

| Provisioner | Runs On | Use Case | Risk Level |
|-------------|---------|----------|------------|
| **`local-exec`** | Your laptop | `curl healthcheck`, `echo "DNS: $IP"` | ✅ Low |
| **`remote-exec`** | EC2 (via SSH) | One-time config, cluster join | ⚠️ Medium (SSH failures) |
| **`file`** | EC2 (via SSH) | Copy configs, scripts, certs | ⚠️ Medium (permissions, paths) |

> ⚠️ **Critical Limitations**:  
> - ❌ **Not idempotent** — runs *every* `apply` (even if no infra change)  
> - ❌ **No drift detection** — Terraform won’t reconcile if file is modified later  
> - ❌ **SSH dependency** — fails if keys rotate, security groups block port 22  

---

## ✏️ Hands-On: Provisioners in Action

### 🔹 File Structure (`/day19/`)
```
day19/
├── main.tf           # EC2 + provisioners
├── scripts/
│   └── welcome.sh    # 📜 Sample script to copy
├── terraform.tfvars  # Key name, private key path
└── TASK.md           # 📝 Your challenge (cloud-init, SSM, AMI)
```

---

### 1️⃣ `main.tf` — EC2 with Provisioners

#### 🔑 Key Setup (Prerequisite)
```bash
# Generate SSH key (for demo only — use SSM in prod!)
aws ec2 create-key-pair --key-name terraform-demo-key \
  --query 'KeyMaterial' --output text > terraform-demo-key.pem
chmod 400 terraform-demo-key.pem
```

#### 🖥️ EC2 Resource + Provisioners
```hcl
resource "aws_instance" "demo" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  key_name      = var.key_name

  vpc_security_group_ids = [aws_security_group.demo.id]

  # 🔌 SSH connection for remote/file provisioners
  connection {
    type        = "ssh"
    user        = var.ssh_user
    private_key = file(var.private_key_path)
    host        = self.public_ip
  }

  # ✅ local-exec: Log to your terminal (safe!)
  provisioner "local-exec" {
    command = "echo '✅ Instance ${self.id} ready at ${self.public_ip}'"
  }

  # ⚠️ remote-exec: Install nginx (one-time only!)
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx",
      "echo 'Hello from ${self.id}' | sudo tee /var/www/html/index.html"
    ]
  }

  # ⚠️ file: Copy config script
  provisioner "file" {
    source      = "${path.module}/scripts/welcome.sh"
    destination = "/tmp/welcome.sh"
  }

  # ✅ remote-exec: Run the copied script
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/welcome.sh",
      "/tmp/welcome.sh"
    ]
  }
}
```

> 💡 **Why `self`?**  
> `self.public_ip` → gets IP *after* instance creation (dynamic attributes).

---

### 2️⃣ `scripts/welcome.sh` — Sample Script
```bash
#!/bin/bash
echo "Welcome to $(hostname)!" > /tmp/welcome.txt
echo "Deployed by: $(whoami)" >> /tmp/welcome.txt
echo "Time: $(date)" >> /tmp/welcome.txt
```

---

## 🧪 Lab: Deploy & Verify

### 🔧 Commands to Run:
```bash
# 1. Initialize
terraform init

# 2. Deploy (pass vars)
terraform apply \
  -var="key_name=terraform-demo-key" \
  -var="private_key_path=./terraform-demo-key.pem"
```

### ✅ Verify Provisioners:
```bash
# SSH into instance
ssh -i terraform-demo-key.pem ubuntu@$(terraform output -raw public_ip)

# Check files
ls /tmp
# → welcome.sh, welcome.txt, remote_exec.txt

# Check nginx
curl localhost
# → "Hello from i-abc123"
```

### 📜 Expected Logs:
```
Executing: echo '✅ Instance i-abc123 ready at 54.123.45.67'
Executing: sudo apt-get update -y
Executing: sudo apt-get install -y nginx
...
```

---

## ⚠️ Critical Gotchas (From Your Video!)

| Issue | Fix |
|-------|-----|
| ❌ `provisioner` outside `resource` | ✅ Must be *inside* resource block |
| ❌ SSH key permissions (`chmod 600`) | ✅ `chmod 400 terraform-demo-key.pem` |
| ❌ Provisioner runs on *every* `apply` | ✅ Use `when = create` (see below) |
| ❌ No error handling | ✅ Wrap in `|| exit 1` for shell scripts |

#### 🛡️ Safer Provisioner Patterns:
```hcl
# Run ONLY on create (not update/destroy)
provisioner "remote-exec" {
  when    = create
  inline  = ["sudo apt install nginx"]
}

# Fail fast on errors
provisioner "remote-exec" {
  inline = [
    "set -e",  # Exit on any error
    "sudo apt install nginx",
    "sudo systemctl start nginx"
  ]
}
```

---

## 🚫 When *NOT* to Use Provisioners (Prod-Ready Alternatives)

| Use Case | Provisioner (❌) | Better Approach (✅) |
|----------|------------------|----------------------|
| **Install packages** | `remote-exec` + `apt install` | **Pre-baked AMI** (Packer) |
| **App deployment** | `file` + `unzip` | **Artifact store** (S3/ECS) |
| **Config management** | `remote-exec` + `sed` | **Immutable configs** (S3 + `aws_s3_object`) |
| **Secret injection** | `remote-exec` + `curl vault` | **SSM Parameter Store** + IAM roles |

> 💡 **Cloud-Init Alternative (No SSH!)**:
> ```hcl
> resource "aws_instance" "demo" {
>   user_data = base64encode(<<-EOF
>     #cloud-config
>     packages:
>       - nginx
>     runcmd:
>       - echo "Hello from cloud-init" > /var/www/html/index.html
>   EOF)
> }
> ```
> ✅ No SSH keys  
> ✅ Idempotent  
> ✅ Runs at *first boot only*  

---

## 🚀 Challenge: Complete the Project (TASK.md)

### 📝 Your Mission:
| Task | Why It Matters |
|------|----------------|
| ✅ **Replace `remote-exec` with `user_data`** | Eliminate SSH dependency (cloud-init) |
| ✅ **Use SSM Session Manager** | No open SSH ports (prod standard) |
| ✅ **Bake nginx into AMI** | Eliminate runtime provisioning (Packer) |
| ✅ **Add health check** | `local-exec` + `curl http://$IP/health` |
| ✅ **Error handling** | `set -e` in all shell scripts |

> 💡 **Pro Tip**:  
> For production:  
> - **Dev**: `user_data` for speed  
> - **Prod**: Immutable AMIs + blue/green deployments  

---

## ❓ FAQ (Day 19 Edition)

**Q: Can I use `provisioner` with `for_each`?**  
A: ✅ Yes — but use `self` for dynamic attributes:  
```hcl
provisioner "local-exec" {
  command = "echo '${each.key}: ${self.public_ip}'"
}
```

**Q: How to debug failing `remote-exec`?**  
A: Add `TF_LOG=DEBUG` + check `/var/log/cloud-init-output.log` on EC2.

**Q: Why does `terraform taint` force provisioner runs?**  
A: `taint` marks resource as *degraded* → `apply` destroys + recreates → provisioners run on *new* instance.

---

## ➡️ Summary: Provisioners — Handle With Care

| Pattern | Safe? | Production-Ready? |
|---------|-------|-------------------|
| `local-exec` for logging | ✅ Yes | ✅ |
| `remote-exec` for one-time setup | ⚠️ Sometimes | ❌ (Prefer cloud-init/AMI) |
| `file` for configs | ⚠️ Sometimes | ❌ (Prefer S3 + IAM roles) |
| SSH-based provisioning | ❌ Avoid | ❌ (Use SSM/cloud-init) |

> 🎯 **Golden Rule**:  
> **“If you’re using `remote-exec` in prod — you’re doing it wrong.”**

---