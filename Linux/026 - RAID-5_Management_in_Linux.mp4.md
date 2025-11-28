# 026: Creating and Managing RAID-5 in Linux

---

## 1. Understanding RAID-5

### Key Concepts

- **Striping with Distributed Parity**: Data + parity blocks spread across **all disks**.
- **Fault Tolerance**: Survives **failure of 1 disk** (in a 3+ disk array).
- **Capacity**: Usable space = **(N-1) × smallest disk** (e.g., 3×2TB → 4TB usable).
- **Minimum Disks**: **3 disks required**.
- **Use Cases**:
    - File servers
    - Database servers
    - Virtualization hosts
    - Any workload needing **balance of performance + redundancy**

> ⚠️ Critical Limitation:
> 
> 
> **Fails completely if 2+ disks fail**. Avoid RAID-5 for large disks (>2TB)—rebuild time increases risk of second failure.
> 

---

## 2. Creating RAID-5 Array

### Prepare Disks

```bash
# Create partitions on 3+ disks (e.g., /dev/sdb, /dev/sdc, /dev/sdd)
for disk in sdb sdc sdd; do
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
lsblk             # Verify sdb1, sdc1, sdd1

```

### Create RAID-5

```bash
# Create RAID-5 with 3 disks
sudo mdadm --create /dev/md0 -l 5 -n 3 /dev/sdb1 /dev/sdc1 /dev/sdd1

# Verify
cat /proc/mdstat    # Shows [UUU] (all disks active)
sudo mdadm -D /dev/md0    # Detailed status (note "Active Devices: 3")

```

> 💡 Initial Sync:
> 
> 
> RAID-5 initializes parity—check `/proc/mdstat` for `[=>...]` progress (can take hours).
> 

### Format and Mount

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir /raid5
sudo mount /dev/md0 /raid5
df -hT              # Verify mounted

# Add test data
echo "cal" | sudo tee /raid5/cal
echo "date" | sudo tee /raid5/date

```

---

## 3. Simulating Disk Failure

### Fail a Disk

```bash
# Check current status
cat /proc/mdstat    # Should show [UUU]

# Fail /dev/sdb1
sudo mdadm /dev/md0 -f /dev/sdb1

# Verify degraded state
cat /proc/mdstat    # Shows [U_U] (one disk failed)
sudo mdadm -D /dev/md0    # Shows /dev/sdb1 as "faulty"

# Data remains accessible!
ls /raid5           # Files still readable

```

> ✅ RAID-5 in degraded mode:
> 
> - Reads work (using parity to reconstruct data)
> - Writes work (but slower—parity recalculated on every write)

---

## 4. Replacing Failed Disk

### Steps to Replace

```bash
# 1. Unmount (recommended but not required)
sudo umount /raid5

# 2. Remove faulty disk
sudo mdadm /dev/md0 -r /dev/sdb1

# 3. Prepare new disk (e.g., /dev/sde1)
sudo fdisk /dev/sde <<EOF
n
p
1

t
fd
w
EOF
sudo partprobe

# 4. Add new disk to array
sudo mdadm /dev/md0 -a /dev/sde1

# 5. Verify rebuild
cat /proc/mdstat    # Shows [U_U] → [UUU] (rebuilding)
sudo mdadm -D /dev/md0    # Shows "rebuilding" status

```

> 🔍 Rebuild Process:
> 
> - New disk syncs data using **parity + surviving disks**
> - Array remains **usable during rebuild** (but vulnerable to second failure)

### Remount and Verify

```bash
sudo mount /dev/md0 /raid5
cat /raid5/cal      # Data intact!

```

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Create RAID-5 | `mdadm --create /dev/md0 -l 5 -n 3 /dev/sdX1 /dev/sdY1 /dev/sdZ1` |
| Fail disk | `mdadm /dev/md0 -f /dev/sdX1` |
| Remove disk | `mdadm /dev/md0 -r /dev/sdX1` |
| Add disk | `mdadm /dev/md0 -a /dev/sdW1` |
| View status | `cat /proc/mdstat`, `mdadm -D /dev/md0` |
| Stop array | `mdadm -S /dev/md0` |

---

## 6. Critical Best Practices

### Before Deployment

1. **Use identical disks** (same size/model) to avoid wasted space.
2. **Save RAID config**:
    
    ```bash
    sudo mdadm --detail --scan | sudo tee /etc/mdadm.conf
    
    ```
    
3. **Size appropriately**:
    - For disks >2TB, consider **RAID-6** (dual parity) instead.

### During Operation

- **Monitor disk health**:
    
    ```bash
    sudo smartctl -a /dev/sdb    # Check for reallocated sectors
    
    ```
    
- **Avoid write-heavy workloads during rebuild**—increases failure risk.

### After Replacement

- **Verify data integrity**:
    
    ```bash
    sudo fsck -f /dev/md0    # After unmounting
    
    ```
    
- **Update initramfs** (for boot arrays):
    
    ```bash
    sudo dracut -f    # RHEL/CentOS
    
    ```
    

---

## 7. RAID-5 vs. Alternatives

| Scenario | Recommendation |
| --- | --- |
| **< 2TB disks, 3-5 disks** | RAID-5 |
| **> 2TB disks or >6 disks** | RAID-6 (dual parity) |
| **Max performance, no redundancy** | RAID-0 |
| **Max redundancy, small capacity** | RAID-1 |

> 💡 Modern Alternative:
> 
> 
> Consider **ZFS** or **Btrfs** for built-in RAID + checksums + snapshots.
> 

---

## Summary Workflow

### Create RAID-5

```bash
fdisk → t fd → mdadm --create /dev/md0 -l 5 -n 3 /dev/sdX1 /dev/sdY1 /dev/sdZ1 → mkfs → mount

```

### Replace Failed Disk

```bash
mdadm -f /dev/sdX1 → mdadm -r /dev/sdX1 → add new disk → mdadm -a /dev/sdW1

```

> 💡 Golden Rule:
> 
> 
> **RAID-5 = Performance + Single-Disk Redundancy**. Always:
> 
> - Monitor disk health proactively
> - Replace failed disks **immediately**
> - Avoid RAID-5 for large disks—use RAID-6 instead!