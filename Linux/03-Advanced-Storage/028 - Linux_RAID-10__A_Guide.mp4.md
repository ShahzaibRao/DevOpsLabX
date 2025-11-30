# 028:Creating and Managing RAID-10 in Linux

---

## 1. Understanding RAID-10 (1+0)

### Key Concepts

- **Hybrid RAID**: Combines **RAID-1 (mirroring)** + **RAID-0 (striping)**.
- **Layout**:
    - **Lower layer**: Mirrors (pairs of disks)
    - **Upper layer**: Stripes across mirrors
- **Fault Tolerance**:
    - Survives **multiple disk failures**—**if no mirror pair is fully lost**.
    - Example (4-disk): Can lose 1 disk per mirror pair (e.g., `/dev/sda1` + `/dev/sdc1`).
- **Capacity**: **50% efficiency** (e.g., 4×2TB → 4TB usable).
- **Minimum Disks**: **4 disks required** (must be even number).
- **Use Cases**:
    - High-performance databases (MySQL, Oracle)
    - Virtualization hosts
    - Any workload needing **max performance + redundancy**

> ⚡ RAID-10 Advantage:
> 
> - **Fastest rebuild time** (only mirror partner needs sync)
> - **No parity calculations** → better write performance than RAID-5/6

---

## 2. Creating RAID-10 Array

### Prepare Disks

```bash
# Create partitions on 4 disks (e.g., /dev/sdb-sde)
for disk in sdb sdc sdd sde; do
  sudo fdisk /dev/$disk <<EOF
n
p
1

t
fd
w
EOF
done

sudo partprobe    # Reload partition table
lsblk             # Verify sdb1, sdc1, sdd1, sde1

```

### Create RAID-10

```bash
# Create RAID-10 with 4 disks
sudo mdadm -C /dev/md0 -l 10 -n 4 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1

# Verify
cat /proc/mdstat    # Shows [UUUU] (all disks active)
sudo mdadm -D /dev/md0    # Detailed status (note "Layout: n2" = near copies)

```

> 💡 Layout Note:
> 
> 
> Default layout `n2` = **near copies** (mirrors adjacent stripes). Most common and performant.
> 

### Format and Mount

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir /raid10
sudo mount /dev/md0 /raid10
df -hT              # Verify mounted

# Add test data
echo "cal" | sudo tee /raid10/cal
echo "date" | sudo tee /raid10/date

```

---

## 3. Simulating Disk Failures

### Fail First Disk (Safe Failure)

```bash
# Fail /dev/sdb1 (mirror partner = /dev/sdc1)
sudo mdadm /dev/md0 -f /dev/sdb1

# Verify degraded state
cat /proc/mdstat    # Shows [U_UU]
ls /raid10          # Data accessible

```

### Fail Second Disk (Critical Failure)

```bash
# ✅ SAFE: Fail from different mirror pair (/dev/sdd1, partner = /dev/sde1)
sudo mdadm /dev/md0 -f /dev/sdd1
cat /proc/mdstat    # Shows [U_U_]

# ❌ DANGEROUS: Fail mirror partner of first disk
# sudo mdadm /dev/md0 -f /dev/sdc1  # Would cause total array failure!

```

> ✅ RAID-10 survives:
> 
> - Failures in **different mirror pairs**
> ❌ **Fails if**:
> - Both disks in **any mirror pair** fail

---

## 4. Replacing Failed Disks

### Steps to Replace Disks

```bash
# 1. Unmount (recommended)
sudo umount /raid10

# 2. Remove failed disks
sudo mdadm /dev/md0 -r /dev/sdb1 /dev/sdd1

# 3. Prepare new disks (e.g., /dev/sdf1, /dev/sdg1)
for disk in sdf sdg; do
  sudo fdisk /dev/$disk <<EOF
n
p
1

t
fd
w
EOF
done
sudo partprobe

# 4. Add new disks
sudo mdadm /dev/md0 -a /dev/sdf1 /dev/sdg1

# 5. Verify rebuild (fast!)
cat /proc/mdstat    # Shows [U_U_] → [UUU_] → [UUUU]
sudo mdadm -D /dev/md0

```

> 🔍 Rebuild Advantage:
> 
> - Only **mirror partner** is read (not entire array)
> - Rebuild time ≈ **single disk sync** (much faster than RAID-5/6)

### Remount and Verify

```bash
sudo mount /dev/md0 /raid10
cat /raid10/cal     # Data intact!

```

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Create RAID-10 | `mdadm -C /dev/md0 -l 10 -n 4 /dev/sdX1 /dev/sdY1 /dev/sdZ1 /dev/sdW1` |
| Fail disk | `mdadm /dev/md0 -f /dev/sdX1` |
| Remove disk | `mdadm /dev/md0 -r /dev/sdX1` |
| Add disk | `mdadm /dev/md0 -a /dev/sdA1` |
| View status | `cat /proc/mdstat`, `mdadm -D /dev/md0` |

---

## 6. Critical Best Practices

### Disk Pairing Strategy

- **Physically separate mirror pairs**:
    - Put `/dev/sdb1` + `/dev/sdc1` on **different controllers/power supplies**
    - Avoid pairing disks from same RAID controller

### Monitoring

- **Check mirror health**:
    
    ```bash
    # Identify mirror pairs (layout n2)
    sudo mdadm -D /dev/md0 | grep "Number"
    # Disks 0+1 = pair1, 2+3 = pair2, etc.
    
    ```
    
- **Test failure scenarios**:
    
    ```bash
    # Simulate disk removal
    echo 1 | sudo tee /sys/block/sdb/device/delete
    
    ```
    

### Performance Tuning

- **Stripe size**: Default 512KB works for most workloads
- **Filesystem alignment**:
    
    ```bash
    sudo mkfs.ext4 -E stride=128,stripe_width=256 /dev/md0
    
    ```
    

---

## 7. RAID-10 vs. Alternatives

| Scenario | Recommendation |
| --- | --- |
| **Max performance + redundancy** | RAID-10 |
| **Capacity efficiency + redundancy** | RAID-6 |
| **Budget storage, non-critical** | RAID-5 |
| **Small setup (<4 disks)** | RAID-1 |

> 💡 When to Choose RAID-10:
> 
> - You need **predictable performance** (no parity write penalty)
> - **Fast rebuilds** are critical (e.g., 24/7 systems)
> - Budget allows for **50% capacity loss**

---

## Summary Workflow

### Create RAID-10

```bash
fdisk → t fd → mdadm -C /dev/md0 -l 10 -n 4 /dev/sdX1..sdW1 → mkfs → mount

```

### Replace Failed Disks

```bash
mdadm -f /dev/sdX1 → mdadm -r /dev/sdX1 → add new disk → mdadm -a /dev/sdA1

```

> 💡 Golden Rule:
> 
> 
> **RAID-10 = Speed + Resilience**. Always:
> 
> - Pair disks across **independent failure domains**
> - Replace failed disks **immediately**
> - Use for **I/O-intensive workloads** where performance is critical!