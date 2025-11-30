# 024: RAID Concepts & Managing RAID-0 in Linux
---


## 1. Understanding RAID

### What is RAID?

**RAID (Redundant Array of Independent Disks)** combines multiple physical disks into a single logical unit for:

- **Performance** (RAID 0)
- **Redundancy** (RAID 1, 5, 6)
- **Capacity** (RAID 0)

### RAID Levels Overview

| Level | Name | Pros | Cons | Use Case |
| --- | --- | --- | --- | --- |
| **RAID 0** | Striping | ⚡ Fastest read/write | ❌ **No redundancy** (1 disk fail = total data loss) | Video editing, gaming, temp data |
| RAID 1 | Mirroring | ✅ Full redundancy | 💾 50% capacity loss | Critical system disks |
| RAID 5 | Striping + Parity | ✅ 1-disk fault tolerance | ⏳ Write penalty | General-purpose storage |
| RAID 6 | Striping + Double Parity | ✅ 2-disk fault tolerance | ⏳ High write penalty | Large archival storage |

> ⚠️ RAID ≠ Backup: RAID protects against disk failure, not data deletion/corruption.
> 

---

## 2. Creating RAID-0 (Striping)

### Prepare Disks

```bash
# Create partitions on 2 disks (e.g., /dev/sdb, /dev/sdc)
sudo fdisk /dev/sdb
# Commands: n → p → 1 → [Enter] → [Enter] → t → fd (Linux RAID) → w

sudo fdisk /dev/sdc
# Same steps → /dev/sdc1

sudo partprobe    # Reload partition table
lsblk             # Verify sdb1, sdc1

```

> 💡 Partition Type fd: Marks partition for RAID (required for safety).
> 

### Create RAID-0 Array

```bash
# Create RAID-0 with 2 disks
sudo mdadm -C /dev/md0 -l 0 -n 2 /dev/sdb1 /dev/sdc1

# Verify
cat /proc/mdstat    # Shows active RAID-0
sudo mdadm -D /dev/md0    # Detailed RAID info
lsblk               # Shows /dev/md0

```

### Format and Use RAID

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir /raid0
sudo mount /dev/md0 /raid0
df -hT              # Verify mounted as ext4

# Test performance
cd /raid0
touch {1..20}
cal > cal.txt

```

> ✅ RAID-0 Benefit:
> 
> 
> Write speed ≈ **sum of all disks** (e.g., 2x disks = 2x speed).
> 

---

## 3. Removing RAID-0

### Steps to Clean Up

```bash
# 1. Unmount
sudo umount /raid0

# 2. Stop RAID array
sudo mdadm -S /dev/md0

# 3. Verify removal
cat /proc/mdstat    # Should show no active arrays

# 4. Remove partitions (optional)
sudo fdisk /dev/sdb → d → w
sudo fdisk /dev/sdc → d → w

```

> ⚠️ Critical:
> 
> 
> Always stop RAID with `mdadm -S` before deleting partitions!
> 

---

## 4. Using RAID-0 as LVM Physical Volume

### Convert RAID to LVM

```bash
# Create larger RAID-0 (3 disks for demo)
sudo mdadm -C /dev/md1 -l 0 -n 3 /dev/sdb1 /dev/sdc1 /dev/sdd1

# Use RAID as PV for LVM
sudo pvcreate /dev/md1
sudo vgcreate raid_vg /dev/md1
sudo lvcreate -L 2G -n raid_lv raid_vg

# Format and mount LV
sudo mkfs.ext4 /dev/raid_vg/raid_lv
sudo mkdir /test
sudo mount /dev/raid_vg/raid_lv /test
df -hT

```

> 💡 Why Combine RAID + LVM?
> 
> - RAID provides **performance**
> - LVM provides **flexibility** (resize, snapshots)

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Create RAID-0 | `mdadm -C /dev/md0 -l 0 -n 2 /dev/sdX1 /dev/sdY1` |
| View RAID status | `cat /proc/mdstat` |
| Detailed RAID info | `mdadm -D /dev/md0` |
| Stop RAID | `mdadm -S /dev/md0` |
| Create PV on RAID | `pvcreate /dev/md0` |
| Monitor rebuild | `watch cat /proc/mdstat` |

---

## 6. Critical Best Practices

### For RAID-0

- **Never use for critical data**—always pair with backups.
- **Use identical disks** (same size/speed) for optimal performance.
- **Monitor disk health**:
    
    ```bash
    sudo smartctl -a /dev/sdb    # Check disk health
    
    ```
    

### General RAID Tips

1. **Save RAID config**:
    
    ```bash
    sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
    
    ```
    
2. **Auto-assemble on boot**:
    
    ```bash
    sudo dracut -f    # RHEL/CentOS (updates initramfs)
    
    ```
    
3. **Test failure recovery** (for redundant RAID levels):
    
    ```bash
    echo 1 | sudo tee /sys/block/sdb/device/delete    # Simulate disk failure
    
    ```
    

---

## Summary Workflow

### Create RAID-0

```bash
fdisk → t fd → mdadm -C /dev/md0 -l 0 -n 2 /dev/sdX1 /dev/sdY1 → mkfs → mount

```

### RAID-0 + LVM

```bash
mdadm -C /dev/md1 → pvcreate /dev/md1 → vgcreate → lvcreate → mkfs → mount

```

> 💡 Golden Rule:
> 
> 
> **RAID-0 = Performance + Risk**. Only use when:
> 
> - You have **backups**
> - Data is **recreatable** (e.g., cache, temp files)
> - **Speed is critical** (video rendering, AI datasets)