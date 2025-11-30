# 025: Creating and Managing RAID-1 (Mirroring) in Linux

## 1. Understanding RAID-1

### Key Concepts

- **Mirroring**: Data is **duplicated** across all disks in the array.
- **Fault Tolerance**: Survives **failure of 1 disk** (in a 2-disk setup).
- **Capacity**: Usable space = **size of smallest disk** (50% efficiency for 2 disks).
- **Use Cases**:
    - Critical system disks (`/`, `/boot`)
    - Database servers
    - Any workload requiring **high availability**

> ✅ RAID-1 Benefit:
> 
> 
> Read performance ≈ **2x single disk** (can read from both disks simultaneously).
> 
> Write performance ≈ **single disk** (must write to both disks).
> 

---

## 2. Creating RAID-1 Array

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

### Create RAID-1

```bash
# Create mirror with 2 disks
sudo mdadm -C /dev/md0 -l 1 -n 2 /dev/sdb1 /dev/sdc1

# Verify
cat /proc/mdstat    # Shows [UU] (both disks active)
sudo mdadm -D /dev/md0    # Detailed status
lsblk -f            # Shows /dev/md0

```

> 💡 Note:
> 
> 
> Initial sync may take time (check `/proc/mdstat` for `[=>...]` progress).
> 

### Format and Mount

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir /raid1
sudo mount /dev/md0 /raid1
df -hT              # Verify mounted

# Add test data
echo "cal" | sudo tee /raid1/cal
echo "date" | sudo tee /raid1/date

```

---

## 3. Simulating Disk Failure

### Mark Disk as Faulty

```bash
# Check current status
cat /proc/mdstat    # Should show [UU]

# Fail /dev/sdb1
sudo mdadm /dev/md0 -f /dev/sdb1

# Verify failure
cat /proc/mdstat    # Shows [U_] (one disk failed)
sudo mdadm -D /dev/md0    # Shows /dev/sdb1 as "faulty"

```

> ⚠️ Data remains accessible! RAID-1 continues operating in degraded mode.
> 

---

## 4. Replacing Failed Disk

### Steps to Replace

```bash
# 1. Unmount filesystem (optional but recommended)
sudo umount /raid1

# 2. Remove faulty disk from array
sudo mdadm /dev/md0 -r /dev/sdb1

# 3. Prepare new disk (e.g., /dev/sdd1)
sudo fdisk /dev/sdd
# → n → p → 1 → [Enter] → [Enter] → t → fd → w
sudo partprobe

# 4. Add new disk to array
sudo mdadm /dev/md0 -a /dev/sdd1

# 5. Verify rebuild
cat /proc/mdstat    # Shows [U_] → [UU] (rebuilding)
sudo mdadm -D /dev/md0    # Shows "rebuilding" status

```

> 🔍 Rebuild Process:
> 
> - New disk syncs data from healthy disk
> - Array remains **usable during rebuild** (but vulnerable to second failure)

### Remount and Verify

```bash
sudo mount /dev/md0 /raid1
cat /raid1/cal      # Data intact!

```

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Create RAID-1 | `mdadm -C /dev/md0 -l 1 -n 2 /dev/sdX1 /dev/sdY1` |
| Fail disk | `mdadm /dev/md0 -f /dev/sdX1` |
| Remove disk | `mdadm /dev/md0 -r /dev/sdX1` |
| Add disk | `mdadm /dev/md0 -a /dev/sdZ1` |
| View status | `cat /proc/mdstat`, `mdadm -D /dev/md0` |
| Stop array | `mdadm -S /dev/md0` |

---

## 6. Critical Best Practices

### Before Failure

1. **Save RAID config**:
    
    ```bash
    sudo mdadm --detail --scan | sudo tee /etc/mdadm.conf
    
    ```
    
2. **Monitor disk health**:
    
    ```bash
    sudo smartctl -a /dev/sdb    # Check for reallocated sectors
    
    ```
    

### During Failure

- **Never remove healthy disk first**—always fail/replace the faulty one.
- **Rebuild during low-usage periods** (rebuilds stress disks).

### After Replacement

- **Verify data integrity**:
    
    ```bash
    sudo fsck -f /dev/md0    # After unmounting
    
    ```
    
- **Update bootloader** (if RAID-1 is for `/boot`):
    
    ```bash
    sudo grub2-install /dev/sdb /dev/sdd    # Install to both disks
    
    ```
    

---

## 7. Troubleshooting Common Issues

### Issue: "Device or resource busy" when removing disk

**Solution**: Unmount filesystem first:

```bash
sudo umount /raid1

```

### Issue: New disk not syncing

**Check**:

1. Partition type = `fd` (Linux RAID)
2. Disk size ≥ failed disk
3. No I/O errors in `dmesg`

### Issue: Array won't auto-assemble on boot

**Fix**:

```bash
# Ensure config exists
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf

# Update initramfs (RHEL/CentOS)
sudo dracut -f

```

---

## Summary Workflow

### Create RAID-1

```bash
fdisk → t fd → mdadm -C /dev/md0 -l 1 -n 2 /dev/sdX1 /dev/sdY1 → mkfs → mount

```

### Replace Failed Disk

```bash
mdadm -f /dev/sdX1 → mdadm -r /dev/sdX1 → add new disk → mdadm -a /dev/sdZ1

```

> 💡 Golden Rule:
> 
> 
> **RAID-1 = Redundancy + Read Performance**. Always:
> 
> - Monitor disk health proactively
> - Test failure/recovery procedures
> - Keep spare disks ready for quick replacement!