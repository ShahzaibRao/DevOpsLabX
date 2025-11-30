# 018: LVM Snapshot Feature in Linux
---


## 1. Understanding LVM Snapshots

### Purpose

- **Point-in-time backup** of a Logical Volume (LV).
- Allows you to:
    - Safely backup data while system is running
    - Roll back to a previous state after accidental deletion/corruption
- **Copy-on-write (COW)**: Only changed blocks are stored in snapshot.

> ⚠️ Critical Notes:
> 
> - Snapshot size must accommodate **expected changes** during its lifetime.
> - If snapshot fills up → **becomes invalid** (data loss risk!).
> - **Cannot merge** while origin LV is mounted (in older RHEL versions).

---

## 2. Lab Setup: Create LVM with Snapshot

### Prepare Disk and Partitions

```bash
# Add 10GB disk (e.g., /dev/sdb)
sudo fdisk /dev/sdb
# Create two partitions:
#   n → p → 1 → [Enter] → +5G → t → 8e
#   n → p → 2 → [Enter] → +5G → t → 8e
#   w
sudo partprobe
lsblk    # Verify sdb1, sdb2

```

### Create LVM

```bash
# Create PV and VG
sudo pvcreate /dev/sdb1
sudo vgcreate system /dev/sdb1

# Create LV (3GB)
sudo lvcreate -L 3G -n data system
sudo mkfs.ext4 /dev/system/data

# Mount and add test data
sudo mkdir /data
sudo mount /dev/system/data /data
cd /data
touch test{1..5}
cal > cal.txt
mkdir aa bb cc
cd
du -sh /data    # Note size (e.g., 20K)

```

---

## 3. Create and Use Snapshot

### Create Snapshot

```bash
# Create 1GB snapshot (name: snap_data)
sudo lvcreate -L 1G -s -n snap_data /dev/system/data
sudo lvs    # Verify: snap_data (snapshot), data (origin)

```

> 💡 Snapshot Size Rule:
> 
> 
> Allocate **20-30% of origin LV size** for moderate changes.
> 
> For 3GB LV → 1GB snapshot is reasonable for short-term use.
> 

### Simulate Data Loss

```bash
cd /data
sudo rm -rf *    # Delete all files
ls               # Empty!
cd

```

---

## 4. Restore from Snapshot

### Merge Snapshot (Rollback)

```bash
# Unmount origin LV FIRST
sudo umount /data

# Merge snapshot into origin
sudo lvconvert --merge /dev/system/snap_data

# Reactivate LV (if needed)
sudo lvchange -an /dev/system/data    # Deactivate
sudo lvchange -ay /dev/system/data    # Activate

# Remount and verify
sudo mount /dev/system/data /data
ls /data    # Original files restored!

```

> ⚠️ Key Requirements for Merge:
> 
> - Origin LV **must be unmounted**.
> - System **must reboot** if origin was active during snapshot creation (in RHEL 7/8).*(Your lab skips reboot—works in some cases but not guaranteed)*

---

## 5. Alternative: Access Snapshot Directly (Read-Only)

If you can’t merge (e.g., origin must stay mounted):

```bash
# Mount snapshot read-only
sudo mkdir /snap_backup
sudo mount -o ro /dev/system/snap_data /snap_backup

# Copy files manually
sudo cp -a /snap_backup/* /data/

# Unmount and remove snapshot
sudo umount /snap_backup
sudo lvremove /dev/system/snap_data

```

---

## 6. Cleanup

### Remove Snapshot (If Not Merged)

```bash
sudo lvremove /dev/system/snap_data

```

### Full Cleanup

```bash
sudo umount /data
sudo lvremove /dev/system/data
sudo vgremove system
sudo pvremove /dev/sdb1

```

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| Create snapshot | `lvcreate -L 1G -s -n snap_name /dev/VG/LV` |
| View snapshots | `lvs` (look for "snap" in Attr column) |
| Merge snapshot | `lvconvert --merge /dev/VG/snap_name` |
| Deactivate LV | `lvchange -an /dev/VG/LV` |
| Activate LV | `lvchange -ay /dev/VG/LV` |
| Mount snapshot RO | `mount -o ro /dev/VG/snap_name /mnt` |

### Critical Best Practices

1. **Size snapshots properly**: Monitor with `lvs -o +data_percent`.
2. **Unmount origin** before merging.
3. **Delete snapshots** after use—they consume space!
4. For production: Use snapshots for **short-term backups only**.

> 💡 Golden Rule:
> 
> 
> LVM snapshots are **not backups**—they protect against user errors,
> 
> but **won’t survive disk failure**. Always pair with real backups!
>