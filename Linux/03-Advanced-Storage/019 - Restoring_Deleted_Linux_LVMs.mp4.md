# 019: Restoring Removed LVM in Linux
---


## 1. Understanding LVM Metadata Backup

### How LVM Auto-Backups Work

- LVM **automatically saves metadata** in `/etc/lvm/archive/` after **every change** (e.g., `lvcreate`, `vgremove`).
- Files are named: `VG_NAME_XXXXX.vg` (e.g., `Linux_VG_00001-123456789.vg`).
- Use `vgcfgrestore` to revert to a previous state.

> ⚠️ Critical:
> 
> 
> This **only works if**:
> 
> - The **physical volume (PV)** still exists (disk/partition intact)
> - You **haven’t overwritten** the PV with new data

---

## 2. Lab Setup: Create and Delete LVM

### Create LVM

```bash
# Add 5GB disk (e.g., /dev/sdb)
sudo fdisk /dev/sdb
# Create partition: n → p → 1 → [Enter] → [Enter] → t → 8e → w
sudo partprobe

# Create LVM
sudo pvcreate /dev/sdb1
sudo vgcreate Linux_VG /dev/sdb1
sudo lvcreate -L 3G -n Linux_LV Linux_VG
sudo mkfs.ext4 /dev/Linux_VG/Linux_LV    # Fixed path!

# Mount and add data
sudo mkdir /data
sudo mount /dev/Linux_VG/Linux_LV /data
cd /data
touch test{1..5}
sudo umount /data

```

> 💡 Note: Your original command had reversed paths—correct is:
> 
> 
> `/dev/Linux_VG/Linux_LV` (not `/dev/Linux_LV/Linux_VG`)
> 

### Delete LVM (Accidentally)

```bash
sudo lvremove /dev/Linux_VG/Linux_LV
sudo lvs    # LV gone!

```

---

## 3. Restore Deleted LVM

### Step 1: Locate Backup Metadata

```bash
cd /etc/lvm/archive
ls -ltrh    # Find latest Linux_VG_*.vg file
# Example: Linux_VG_00002-123456789.vg

```

### Step 2: Test Restore (Dry Run)

```bash
sudo vgcfgrestore Linux_VG --test -f Linux_VG_00002-123456789.vg
# Output: "Test mode: Metadata will NOT be restored"

```

### Step 3: Perform Actual Restore

```bash
sudo vgcfgrestore Linux_VG -f Linux_VG_00002-123456789.vg
# Output: "Volume group Linux_VG has been restored"

```

### Step 4: Reactivate LVM

```bash
# Scan for VGs and LVs
sudo vgscan
sudo lvscan    # Shows LV as "inactive"

# Activate LV
sudo lvchange -ay /dev/Linux_VG/Linux_LV
sudo lvscan    # Now shows "active"

# Mount and verify
sudo mount /dev/Linux_VG/Linux_LV /data
ls /data       # Original files restored!

```

---

## 4. Key Commands Reference

| Task | Command |
| --- | --- |
| View LVM backups | `ls -ltr /etc/lvm/archive/` |
| Test restore | `vgcfgrestore VG_NAME --test -f backup_file` |
| Restore VG | `vgcfgrestore VG_NAME -f backup_file` |
| Scan VGs | `vgscan` |
| Scan LVs | `lvscan` |
| Activate LV | `lvchange -ay /dev/VG/LV` |

---

## 5. Critical Notes & Best Practices

### When Restoration Works

✅ **Works if**:

- PV (`/dev/sdb1`) still exists
- No new data written to PV after deletion

❌ **Fails if**:

- Disk/partition was deleted/reformatted
- New PV created on same disk

### Pro Tips

1. **Backup `/etc/lvm/` regularly**:
    
    ```bash
    sudo tar -czf lvm-backup-$(date +%F).tar.gz /etc/lvm/
    
    ```
    
2. **Increase archive retention** (edit `/etc/lvm/lvm.conf`):
    
    ```
    # Keep last 50 backups (default: 10)
    retention = 50
    
    ```
    
3. **Verify backups** after critical changes:
    
    ```bash
    sudo vgcfgbackup Linux_VG    # Manual backup
    
    ```
    

---

## Summary Workflow

```bash
# 1. Find backup
ls /etc/lvm/archive/Linux_VG_*.vg

# 2. Restore
sudo vgcfgrestore Linux_VG -f Linux_VG_00002-123456789.vg

# 3. Reactivate
sudo vgscan
sudo lvchange -ay /dev/Linux_VG/Linux_LV

# 4. Mount
sudo mount /dev/Linux_VG/Linux_LV /data

```

> 💡 Golden Rule:
> 
> 
> LVM metadata restore **recovers structure only**—your data is safe
> 
> as long as the **physical disk wasn’t overwritten**. Always verify
> 
> with `fsck` after restore if corruption is suspected!
>