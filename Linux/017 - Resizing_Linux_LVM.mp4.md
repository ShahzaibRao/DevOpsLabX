# 017: Extending and Reducing LVM in Linux
---


## 1. Initial Setup: Create LVM with Two Disks

### Add Disks and Partition

```bash
# Partition /dev/sda (3GB)
sudo fdisk /dev/sda
# Commands: n → p → 1 → [Enter] → +3G → t → 8e → w

# Partition /dev/sdb (2GB)
sudo fdisk /dev/sdb
# Commands: n → p → 1 → [Enter] → +2G → t → 8e → w

sudo partprobe    # Reload partition table
lsblk             # Verify sda1, sdb1

```

### Create Physical Volumes (PVs)

```bash
sudo pvcreate /dev/sda1 /dev/sdb1
sudo pvs          # Should show both PVs (3G + 2G = 5G total)

```

### Create Volume Group (VG)

```bash
sudo vgcreate zybi_vg /dev/sda1 /dev/sdb1
sudo vgs          # Verify VG size (~4.99G usable)

```

### Create Logical Volume (LV)

```bash
sudo lvcreate -L 4G -n zybi_lv zybi_vg
sudo lvs          # Verify 4G LV

```

> 💡 Note: Your original command had reversed paths—correct LV path is:
> 
> 
> `/dev/zybi_vg/zybi_lv` (not `/dev/zybi_lv/zybi_vg`)
> 

---

## 2. Format and Mount LV

### Correct Commands

```bash
# Format
sudo mkfs.ext4 /dev/zybi_vg/zybi_lv

# Mount
sudo mkdir /testdir
sudo mount /dev/zybi_vg/zybi_lv /testdir

# Test data
cd /testdir
touch test{1..20}
cal > cal.txt
cd

```

---

## 3. Extend LVM (Online Resize)

### Extend by 1000MB (1GB)

```bash
# Check current size
df -hT /testdir

# Extend LV (+1GB)
sudo lvextend -L +1000M /dev/zybi_vg/zybi_lv

# Resize filesystem (ext4)
sudo resize2fs /dev/zybi_vg/zybi_lv

# Verify
df -hT /testdir    # Should show ~5G total

```

> ✅ Key Point:
> 
> 
> `resize2fs` works **online** for ext4—no unmount needed!
> 

---

## 4. Extend LVM Further (Add New Disk)

### Add Third Partition

```bash
# Create new partition on /dev/sda
sudo fdisk /dev/sda
# Commands: n → p → 2 → [Enter] → +3G → t → 8e → w
sudo partprobe

```

### Add to Volume Group

```bash
sudo pvcreate /dev/sda2
sudo vgextend zybi_vg /dev/sda2
sudo vgs    # VG size now ~7.99G

```

### Extend LV to 5G (with Auto-Resize)

```bash
# -r flag = auto-resize filesystem after extending LV
sudo lvextend -L 5G -r /dev/zybi_vg/zybi_lv
df -hT /testdir    # Confirms 5G with data intact

```

> 💡 Pro Tip:
> 
> 
> Use `-r` with `lvextend` to **automatically resize** ext2/3/4 or XFS filesystems!
> 

---

## 5. Reducing LVM (Shrinking)

> ⚠️ Critical:
> 
> 
> **ext4 cannot be shrunk online!** You **MUST unmount first**.
> 

### Steps to Reduce LV

```bash
# 1. Unmount
sudo umount /testdir

# 2. Check filesystem (required before resize)
sudo e2fsck -f /dev/zybi_vg/zybi_lv

# 3. Shrink filesystem FIRST (to 3G)
sudo resize2fs /dev/zybi_vg/zybi_lv 3G

# 4. Shrink LV to match filesystem
sudo lvreduce -L 3G /dev/zybi_vg/zybi_lv

# 5. Remount
sudo mount /dev/zybi_vg/zybi_lv /testdir
df -hT /testdir    # Verify 3G

```

> ❌ Never skip e2fsck and filesystem resize—data loss risk!
> 

---

## 6. Key Commands Reference

| Task | Command |
| --- | --- |
| Create PV | `pvcreate /dev/sdX1` |
| Create VG | `vgcreate VG_NAME /dev/sdX1` |
| Extend VG | `vgextend VG_NAME /dev/sdX2` |
| Create LV | `lvcreate -L 4G -n LV_NAME VG_NAME` |
| Extend LV (+1G) | `lvextend -L +1G /dev/VG/LV` |
| Auto-resize LV | `lvextend -L 5G -r /dev/VG/LV` |
| Shrink filesystem | `resize2fs /dev/VG/LV 3G` |
| Shrink LV | `lvreduce -L 3G /dev/VG/LV` |
| Check filesystem | `e2fsck -f /dev/VG/LV` |

---

## Summary Workflow

### Extend LVM (Online)

```bash
lvextend -L +1G -r /dev/VG/LV    # One command!

```

### Reduce LVM (Offline)

```bash
umount /mount
e2fsck -f /dev/VG/LV
resize2fs /dev/VG/LV 3G
lvreduce -L 3G /dev/VG/LV
mount /dev/VG/LV /mount

```

> 💡 Golden Rules:
> 
> - **Always extend LV before filesystem** (or use `r`).
> - **Always shrink filesystem before LV**.
> - **Never reduce without `e2fsck`**.
> - **XFS can grow but NOT shrink**—plan accordingly!