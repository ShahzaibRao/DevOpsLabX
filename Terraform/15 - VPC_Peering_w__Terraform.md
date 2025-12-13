# **1.15 : Mini-Project #2 — VPC Peering (Cross-Region, Secure, Terraform-Managed)**

🎯 **Goal**: Connect two VPCs **privately** (no internet gateway!) — across regions, with Terraform.

> ✅ You’ll build:
> 
> - ✅ Two VPCs (`10.0.0.0/16` + `10.1.0.0/16`) — *non-overlapping CIDRs*
> - ✅ Cross-region VPC peering (us-east-1 ↔ us-west-2)
> - ✅ **Bidirectional** peering acceptance
> - ✅ Route table updates for private traffic
> - ✅ EC2 instances with Apache (test `curl`/`ping` over private IPs)

---

## 🧠 Why VPC Peering? (And Why *Not* Internet Gateway)

| Approach | Security | Cost | Latency | Use Case |
| --- | --- | --- | --- | --- |
| ❌ Public IPs + IGW | ❌ Exposed to internet | High (data transfer) | High (public path) | ❌ Never for internal traffic |
| ✅ **VPC Peering** | ✅ Private, encrypted | Low (regional) | ⚡ Low (direct) | ✅ App ↔ DB, Shared services |

> ⚠️ Critical Rules:
> 
> - 🔒 **CIDRs must NOT overlap** (e.g., `10.0.0.0/16` + `10.1.0.0/16` ✅; `10.0.0.0/16` + `10.0.1.0/24` ❌)
> - ↔️ **Peering is NOT transitive**: `A ↔ B` + `B ↔ C` ≠ `A ↔ C` → **explicit `A ↔ C` required**
> - ✅ **Bidirectional**: `A → B` *and* `B → A` must be accepted

---

## 📦 Architecture Diagram

```mermaid
flowchart LR
  subgraph us-east-1 [VPC A: 10.0.0.0/16]
    A_EC2[EC2: 10.0.1.10]
    A_RT[Route Table]
    A_IGW[Internet Gateway] -. optional .-> A_EC2
    A_RT -->|10.1.0.0/16 via pcx-123| Peering
  end

  subgraph us-west-2 [VPC B: 10.1.0.0/16]
    B_EC2[EC2: 10.1.1.10]
    B_RT[Route Table]
    B_RT -->|10.0.0.0/16 via pcx-456| Peering
  end

  Peering[VPC Peering Connection\\npcx-123 (A→B)\\npcx-456 (B→A)] <--> A_RT & B_RT

```

✅ **Key Flow**:

1. `A_EC2` → `10.1.1.10` (B’s private IP)
2. Route table in VPC A matches `10.1.0.0/16` → sends to `pcx-123`
3. Peering connection routes to VPC B’s `10.1.1.10`
4. 🔐 **All traffic stays on AWS backbone — *never* hits public internet**

---

## ✏️ Hands-On: Terraform Implementation

### 🔹 File Structure (`/day15/`)

```
day15/
├── provider.tf       # Multi-region providers (alias = primary/secondary)
├── variables.tf      # CIDRs, regions, instance types
├── data.tf           # AZs, AMIs (region-specific)
├── vpc-a.tf          # VPC A (us-east-1)
├── vpc-b.tf          # VPC B (us-west-2)
├── peering.tf        # Peering + route tables
├── instances.tf      # EC2 + security groups
└── TASK.md           # 📝 Your challenge (transitive fix, security hardening)

```

---

### 1️⃣ `provider.tf` — Multi-Region Setup

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

# 🔌 Primary region (us-east-1)
provider "aws" {
  region = var.primary_region
  alias  = "primary"
}

# 🔌 Secondary region (us-west-2)
provider "aws" {
  region = var.secondary_region
  alias  = "secondary"
}

```

> 💡 Why aliases?
> 
> 
> Avoid `provider "aws" {}` duplication → clean, scalable configs.
> 

---

### 2️⃣ `variables.tf` — Inputs

```hcl
variable "primary_region" {
  type    = string
  default = "us-east-1"
}

variable "secondary_region" {
  type    = string
  default = "us-west-2"
}

variable "vpc_a_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "vpc_b_cidr" {
  type    = string
  default = "10.1.0.0/16"
}

```

---

### 3️⃣ `data.tf` — Region-Specific Discovery

```hcl
# 🔍 AZs & AMIs for primary region
data "aws_availability_zones" "primary" {
  provider = aws.primary
  state    = "available"
}

data "aws_ami" "ubuntu_primary" {
  provider    = aws.primary
  most_recent = true
  owners      = ["099720109477"] # Ubuntu

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# 🔍 AZs & AMIs for secondary region (identical, different provider)
data "aws_availability_zones" "secondary" {
  provider = aws.secondary
  state    = "available"
}

data "aws_ami" "ubuntu_secondary" {
  provider    = aws.secondary
  most_recent = true
  owners      = ["099720109477"]
  # ... same filters
}

```

> ⚠️ Critical:
> 
> - AMI IDs **vary by region** → *separate data sources required*
> - Never hardcode `ami = "ami-123"` → breaks portability!

---

### 4️⃣ `vpc-a.tf` + `vpc-b.tf` — VPCs & Subnets

```hcl
# VPC A (us-east-1)
resource "aws_vpc" "a" {
  provider = aws.primary
  cidr_block           = var.vpc_a_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "vpc-a" }
}

resource "aws_subnet" "a" {
  provider          = aws.primary
  vpc_id            = aws_vpc.a.id
  cidr_block        = cidrsubnet(var.vpc_a_cidr, 8, 0) # 10.0.1.0/24
  availability_zone = data.aws_availability_zones.primary.names[0]
  tags = { Name = "subnet-a" }
}

```

*(Repeat for VPC B with `provider = aws.secondary`, `var.vpc_b_cidr`)*

---

### 5️⃣ `peering.tf` — **The Core**

### 🔗 Create & Accept Peering

```hcl
# A → B (requestor)
resource "aws_vpc_peering_connection" "a_to_b" {
  provider        = aws.primary
  vpc_id          = aws_vpc.a.id
  peer_vpc_id     = aws_vpc.b.id
  peer_region     = var.secondary_region
  auto_accept     = false  # B must accept
  tags = { Name = "a-to-b" }
}

# B → A (acceptor)
resource "aws_vpc_peering_connection_accepter" "b_accepts_a" {
  provider                = aws.secondary
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
  auto_accept             = true
}

```

### 🛣️ Update Route Tables

```hcl
# Route in VPC A: traffic to VPC B → peering
resource "aws_route" "a_to_b" {
  provider         = aws.primary
  route_table_id   = aws_vpc.a.main_route_table_id
  destination_cidr_block = var.vpc_b_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
}

# Route in VPC B: traffic to VPC A → peering
resource "aws_route" "b_to_a" {
  provider         = aws.secondary
  route_table_id   = aws_vpc.b.main_route_table_id
  destination_cidr_block = var.vpc_a_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.a_to_b.id
}

```

> ✅ Why main_route_table_id?
> 
> 
> Avoids extra `aws_route_table_association` — uses VPC’s default route table.
> 

---

### 6️⃣ `instances.tf` — Test Connectivity

### 🔐 Security Groups (Least Privilege)

```hcl
# Allow VPC-A ↔ VPC-B traffic
resource "aws_security_group" "a" {
  provider = aws.primary
  vpc_id   = aws_vpc.a.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.vpc_b_cidr]  # ← Only VPC B
  }

  ingress {
    protocol    = "icmp"
    from_port   = -1
    to_port     = -1
    cidr_blocks = [var.vpc_b_cidr]  # ← Ping from VPC B
  }

  egress { protocol = "-1"; from_port = 0; to_port = 0; cidr_blocks = ["0.0.0.0/0"] }
}

```

### 🖥️ EC2 with Apache (Test via `curl`)

```hcl
resource "aws_instance" "a" {
  provider      = aws.primary
  ami           = data.aws_ami.ubuntu_primary.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.a.id
  vpc_security_group_ids = [aws_security_group.a.id]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update && apt-get install -y apache2
    echo "<h1>VPC A: $(curl -s <http://169.254.169.254/latest/meta-data/local-ipv4>)</h1>" > /var/www/html/index.html
    systemctl start apache2
  EOF
}

```

*(Repeat for VPC B)*

---

## 🧪 Lab: Validate Peering

### 🔧 Commands to Run:

```bash
terraform apply -auto-approve

# Get private IPs
A_IP=$(terraform output -raw instance_a_private_ip)
B_IP=$(terraform output -raw instance_b_private_ip)

# SSH into VPC A instance
ssh -i vpc-a-key.pem ubuntu@$A_IP

# From VPC A → ping/curl VPC B
ping $B_IP          # ✅ Should reply
curl http://$B_IP   # ✅ "VPC B: 10.1.1.10"

```

### ✅ Expected Output:

```bash
$ curl <http://10.1.1.10>
<h1>VPC B: 10.1.1.10</h1>

```

---

## 🚀 Challenge: Complete the Project ([TASK.md](http://task.md/))

### 📝 Your Mission:

| Task | Why It Matters |
| --- | --- |
| ✅ **Fix Transitive Peering** | Add `VPC C (192.168.0.0/16)` → why `A ↔ C` fails → fix with explicit peering |
| ✅ **Remove IGW for Peering** | Current route tables use IGW → **breaks private traffic** → update to `local` routes only |
| ✅ **Harden SSH** | Replace `cidr_blocks = [var.vpc_b_cidr]` → `cidr_blocks = [var.my_ip]` (your IP) |
| ✅ **Simplify with `for_each`** | Replace `vpc-a.tf`/`vpc-b.tf` → single module with `for_each = { a = {...}, b = {...} }` |

> 💡 Pro Tip:
> 
> 
> Use `terraform state list` to debug resource addresses:
> 
> `aws_vpc_peering_connection.a_to_b` vs `aws_vpc_peering_connection_accepter.b_accepts_a`
> 

---

## ⚠️ Critical Gotchas (From Your Video!)

| Issue | Fix |
| --- | --- |
| ❌ Peering `ACTIVE` but no traffic | ✅ Check **route tables** — missing `destination_cidr_block` + `vpc_peering_connection_id` |
| ❌ `ping: Operation not permitted` | ✅ Security group: allow `icmp` (not just `tcp/22`) |
| ❌ AMI forces replacement | ✅ Pin AMI with `filter { name = "name"; values = ["exact-ami-name"] }` |
| ❌ Overlapping CIDRs | ✅ `terraform validate` early — catch with `cidrsubnet()` math |

---

## ❓ FAQ (Day 15 Edition)

**Q: Why `aws_vpc_peering_connection_accepter` and not `aws_vpc_peering_connection` for B→A?**

A: Peering is *asymmetric*:

- Requestor (`A→B`) = `aws_vpc_peering_connection`
- Acceptor (`B`) = `aws_vpc_peering_connection_accepter` (simpler, no duplicate config)

**Q: Can I peer VPCs in the *same* region?**

A: ✅ Yes — same config, omit `peer_region` (Terraform infers it).

**Q: What if I delete the acceptor?**

A: Peering status = `FAILED` → recreate acceptor (no need to recreate requestor).

---

## ➡️ Summary: VPC Peering Done Right

| Anti-Pattern | Production Practice |
| --- | --- |
| ❌ Public IPs for internal traffic | ✅ Private IPs + peering routes |
| ❌ Overlapping CIDRs | ✅ `10.0.0.0/16` + `10.1.0.0/16` (non-overlapping) |
| ❌ Transitive assumption | ✅ Explicit `A↔B`, `B↔C`, `A↔C` |
| ❌ Hardcoded AMIs | ✅ Region-specific `data "aws_ami"` |

> 🎯 Golden Rule:
> 
> 
> **“If your VPC peering uses internet gateways for internal traffic — it’s broken.”**
>