# 020: Merge and Split Volume Groups in LVM
---


## 1. Understanding Volume Group Operations

### Key Concepts

- **Merge VGs**: Combine two Volume Groups into one (all LVs/PVs become part of the target VG).
- **Split VG**: Extract a Physical Volume (PV) from a VG to create a new VG.
- **Requirements**:
    - All LVs must be **unmounted and deactivated**.
    - PVs must be **unused** (no LVs allocated on them for split).

---

## 2. Lab Setup: Create Two Volume Groups

### Prepare Disk Partitions

```bash
# Partition /dev/sdb (assuming 10GB+ disk)
sudo fdisk /dev/sdb
# Commands:
#   n → p → 1 → [Enter] → +5G → t → 8e
#   n → p → 2 → [Enter] → [Enter] → t → 8e
#   w
sudo partprobe
lsblk    # Verify sdb1, sdb2

```

### Create Physical Volumes (PVs)

```bash
sudo pvcreate /dev/sdb1 /dev/sdb2
sudo pvs

```

### Create Two Volume Groups (VGs)

```bash
sudo vgcreate vg1 /dev/sdb1
sudo vgcreate vg2 /dev/sdb2
sudo vgs    # Verify two separate VGs

```

### Create Logical Volumes (LVs)

```bash
# Create LVs using 1000 extents (not MB!)
sudo lvcreate -l 1000 -n lv1 vg1
sudo lvcreate -l 1000 -n lv2 vg2
sudo lvs

```

> 💡 Note: -l = extents (not MB). Check extent size with vgdisplay vg1.
> 

### Format and Mount

```bash
sudo mkfs.ext4 /dev/vg1/lv1
sudo mkfs.ext4 /dev/vg2/lv2

sudo mkdir /test1 /test2
sudo mount /dev/vg1/lv1 /test1
sudo mount /dev/vg2/lv2 /test2

# Add test data
echo "cal" | sudo tee /test1/cal
echo "date" | sudo tee /test2/date
sudo touch /test1/{1..10}
sudo touch /test2/{a..z}

```

---

## 3. Merge Volume Groups

### Steps to Merge `vg2` into `vg1`

```bash
# 1. Unmount filesystems
sudo umount /test1 /test2

# 2. Deactivate source VG (vg2)
sudo vgchange -an vg2    # -an = deactivate all LVs in VG

# 3. Merge vg2 into vg1
sudo vgmerge vg1 vg2     # vg1 = target, vg2 = source

# 4. Verify
sudo vgs    # Only vg1 remains
sudo lvs    # Both lv1 and lv2 now in vg1

# 5. Activate and mount
sudo lvchange -ay /dev/vg1/lv2    # Activate lv2 (lv1 auto-activates)
sudo mount /dev/vg1/lv1 /test1
sudo mount /dev/vg1/lv2 /test2

# 6. Verify data
ls /test1 /test2    # Data intact!

```

> ⚠️ Critical:
> 
> - **Source VG (`vg2`) must be deactivated** before merge.
> - All LVs from `vg2` become part of `vg1`.

---

## 4. Split Volume Group

### Steps to Split `vg1` Back into `vg1` + `vg2`

```bash
# 1. Unmount and deactivate
sudo umount /test1 /test2
sudo vgchange -an vg1    # Deactivate entire VG

# 2. Split PV /dev/sdb2 into new VG "vg2"
sudo vgsplit vg1 vg2 /dev/sdb2

# 3. Verify
sudo vgs    # vg1 and vg2 restored
sudo pvs    # /dev/sdb1 → vg1, /dev/sdb2 → vg2

# 4. Activate and mount
sudo vgchange -ay vg1 vg2
sudo mount /dev/vg1/lv1 /test1
sudo mount /dev/vg2/lv2 /test2

# 5. Verify data
ls /test1 /test2    # Data still intact!

```

> 💡 Key Rule for Split:
> 
> 
> The PV you split (`/dev/sdb2`) **must not contain active LVs** from the original VG.
> 
> LVM automatically moves data if needed (but requires free space).
> 

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Deactivate VG | `vgchange -an VG_NAME` |
| Activate VG | `vgchange -ay VG_NAME` |
| Merge VGs | `vgmerge TARGET_VG SOURCE_VG` |
| Split VG | `vgsplit SOURCE_VG NEW_VG /dev/PV` |
| Scan LVs | `lvscan` |
| View PVs | `pvs` |

---

## 6. Critical Notes & Best Practices

### Merge Requirements

- Source VG must be **deactivated** (`vgchange -an`).
- Both VGs must use **same metadata format** (usually automatic).

### Split Requirements

- The PV to split **must be free of LVs** (or LVM moves data automatically).
- Ensure **enough free space** in source VG to relocate data if needed.

### Safety Tips

1. **Always unmount filesystems** before VG operations.
2. **Backup LVM config** before major changes:
    
    ```bash
    sudo vgcfgbackup vg1
    sudo vgcfgbackup vg2
    
    ```
    
3. Use **`v` (verbose)** flag for debugging:
    
    ```bash
    sudo vgmerge -v vg1 vg2
    
    ```
    

---

## Summary Workflow

### Merge VGs

```bash
umount /mounts
vgchange -an vg2
vgmerge vg1 vg2
vgchange -ay vg1
mount /dev/vg1/lv1 /test1
mount /dev/vg1/lv2 /test2

```

### Split VG

```bash
umount /mounts
vgchange -an vg1
vgsplit vg1 vg2 /dev/sdb2
vgchange -ay vg1 vg2
mount /dev/vg1/lv1 /test1
mount /dev/vg2/lv2 /test2

```

> 💡 Golden Rule:
> 
> 
> **Always deactivate VGs** before merge/split operations.
> 
> Verify with `pvs`/`vgs`/`lvs` at every step!
>