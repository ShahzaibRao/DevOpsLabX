# 021: Migrating Physical Volumes (PVs) & LVM Between Disks

## 1. Understanding PV Migration

### Purpose

- Move data from one disk to another **without downtime**.
- Replace failing disks or upgrade storage.
- **`pvmove`** relocates data at the **Logical Extent (LE)** level—filesystems stay mounted!

> ✅ Key Benefit:
> 
> 
> Migration happens **online**—users/apps keep working during the move.
> 

---

## 2. Lab Setup: Create LVM on Source Disk

### Prepare Source Disk (`/dev/sda`)

```bash
# Create partition on /dev/sda
sudo fdisk /dev/sda
# Commands: n → p → 1 → [Enter] → [Enter] → t → 8e → w
sudo partprobe

# Create LVM
sudo pvcreate /dev/sda1
sudo vgcreate dataVG /dev/sda1
sudo lvcreate -L 1000M -n dataLV dataVG    # Fixed: -L for size (not -l)
sudo mkfs.ext4 /dev/dataVG/dataLV

# Mount and add data
sudo mkdir /data
sudo mount /dev/dataVG/dataLV /data
sudo touch /data/test{a..z}

```

> ⚠️ Note:
> 
> - `l` = **extents** (not MB). Use `L 1000M` for 1000MB.

---

## 3. Migrate LVM to New Disk (`/dev/sdb`)

### Prepare Target Disk

```bash
# Create partition on /dev/sdb
sudo fdisk /dev/sdb
# Commands: n → p → 1 → [Enter] → [Enter] → t → 8e → w
sudo partprobe

# Add to Volume Group
sudo pvcreate /dev/sdb1
sudo vgextend dataVG /dev/sdb1
sudo vgs    # Verify both PVs in VG

```

### Migrate Data with `pvmove`

```bash
# Check current LV location
sudo dmsetup deps /dev/dataVG/dataLV    # Shows: sda1

# Migrate LV from sda1 → sdb1
sudo pvmove -n dataLV /dev/sda1 /dev/sdb1

# Verify
sudo pvs    # sda1 should show 0 free, sdb1 used
sudo dmsetup deps /dev/dataVG/dataLV    # Now shows: sdb1

```

> 💡 How pvmove Works:
> 
> - Copies data from source PV → target PV
> - Updates LVM metadata
> - **No unmount needed!**

---

## 4. Remove Old Disk from LVM

### Cleanup Source PV

```bash
# Remove PV from VG (after migration)
sudo vgreduce dataVG /dev/sda1
sudo pvremove /dev/sda1

# Verify data integrity
ls /data    # Files still accessible!

```

---

## 5. Migrating Root and Swap LVs (Advanced)

> ⚠️ Critical:
> 
> 
> Root (`/`) and swap **cannot be unmounted**—but `pvmove` works online!
> 

### Steps for Root/Swap Migration

```bash
# 1. Add new PV to existing VG (e.g., "rhel" for RHEL)
sudo pvcreate /dev/sda1
sudo vgextend rhel /dev/sda1

# 2. Migrate root LV
sudo pvmove -n root /dev/sdb1 /dev/sda1

# 3. Migrate swap LV
sudo pvmove -n swap /dev/sdb1 /dev/sda1

# 4. Remove old PV
sudo vgreduce rhel /dev/sdb1
sudo pvremove /dev/sdb1

```

### Verify Critical LVs

```bash
# Check root LV
df /                # Should show new PV
lsblk               # Verify root LV on /dev/sda1

# Check swap
swapon --show       # Should show new swap location

```

> 🔍 Note:
> 
> - Root VG name is usually `rhel` (RHEL) or `ubuntu-vg` (Ubuntu).
> - LV names: `root`, `swap` (check with `lvs`).

---

## 6. Key Commands Reference

| Task | Command |
| --- | --- |
| Create PV | `pvcreate /dev/sdX1` |
| Extend VG | `vgextend VG_NAME /dev/sdX1` |
| Migrate LV | `pvmove -n LV_NAME /dev/OLD_PV /dev/NEW_PV` |
| Reduce VG | `vgreduce VG_NAME /dev/OLD_PV` |
| Remove PV | `pvremove /dev/sdX1` |
| Check LV deps | `dmsetup deps /dev/VG/LV` |

---

## 7. Critical Best Practices

### Before Migration

1. **Check free space**:
    
    ```bash
    sudo pvs    # Ensure target PV has enough space
    
    ```
    
2. **Backup LVM config**:
    
    ```bash
    sudo vgcfgbackup dataVG
    
    ```
    

### During Migration

- **Monitor progress**:
    
    ```bash
    sudo pvmove    # Shows % complete
    
    ```
    
- **Don’t interrupt**—`pvmove` is atomic (safe to resume if interrupted).

### After Migration

1. **Update `/etc/fstab`** if using **device names** (use **UUIDs** instead!):
    
    ```bash
    sudo blkid /dev/dataVG/dataLV    # Get UUID
    
    ```
    
2. **Reinstall bootloader** if migrating `/boot`:
    
    ```bash
    sudo grub2-install /dev/sda      # For BIOS
    
    ```
    

---

## Summary Workflow

```bash
# 1. Prepare new disk
fdisk /dev/sdb → create sdb1 → pvcreate /dev/sdb1

# 2. Extend VG
vgextend dataVG /dev/sdb1

# 3. Migrate data
pvmove -n dataLV /dev/sda1 /dev/sdb1

# 4. Remove old disk
vgreduce dataVG /dev/sda1
pvremove /dev/sda1

```

> 💡 Golden Rule:
> 
> 
> **Always use UUIDs in `/etc/fstab`**—device names (`/dev/sda1`) can change after disk migration!
> 
> Verify with `sudo blkid` and update `/etc/fstab` if needed.
>