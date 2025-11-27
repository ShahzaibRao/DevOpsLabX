# 016: Logical Volume Management (LVM) in Linux
---


## 1. Overview of LVM Components

| Component | Purpose | Example |
| --- | --- | --- |
| **Physical Volume (PV)** | Raw storage (disk/partition) | `/dev/sda1`, `/dev/sdb` |
| **Volume Group (VG)** | Pool of PVs | `LinuxVG` |
| **Logical Volume (LV)** | Usable "partition" from VG | `/dev/LinuxVG/LinuxLV` |

> ✅ Benefits: Resize volumes online, combine disks, snapshot support.
> 

---

## 2. Prepare Disks for LVM

### Create LVM Partitions

```bash
# Partition /dev/sda
sudo fdisk /dev/sda

# In fdisk:
# n → p → 1 → [Enter] → +2G → t → 8e (Linux LVM)
# n → p → 2 → [Enter] → +3G → t → 8e
# w

sudo partprobe    # Reload partition table
lsblk             # Verify sda1, sda2

```

> 💡 Partition Type 8e: Marks partition as LVM (required for safety).
> 

---

## 3. Create Physical Volumes (PVs)

### Initialize PVs

```bash
sudo pvcreate /dev/sda1 /dev/sda2    # Initialize both partitions
sudo pvs                             # Brief PV info
sudo pvdisplay                       # Detailed PV info

```

> ⚠️ Warning: pvcreate erases data on the partition!
> 

---

## 4. Create Volume Group (VG)

### Pool PVs into a VG

```bash
sudo vgcreate LinuxVG /dev/sda1      # Start with sda1
sudo vgs                             # Brief VG info
sudo vgdisplay                       # Detailed VG info

```

### Extend VG with Additional PV

```bash
sudo vgextend LinuxVG /dev/sda2      # Add sda2 to VG
sudo vgs                             # Verify increased size

```

---

## 5. Create and Use Logical Volumes (LVs)

### Create LV

```bash
# Create 3GB LV
sudo lvcreate -L 3G -n LinuxLV LinuxVG

# OR create 2000MB LV
sudo lvcreate -L 2000M -n LinuxLV2 LinuxVG

```

### Verify LVs

```bash
sudo lvs          # Brief LV info
sudo lvdisplay    # Detailed LV info
lsblk             # Shows LV under /dev/mapper/

```

### Format and Mount LV

```bash
# Format as ext4
sudo mkfs.ext4 /dev/LinuxVG/LinuxLV

# Mount
sudo mkdir /testdir
sudo mount /dev/LinuxVG/LinuxLV /testdir
df -Th            # Verify mount and type

# Test
cd /testdir
touch test{1..10}
cal > cal.txt

```

> ✅ Device Paths:
> 
> - `/dev/LinuxVG/LinuxLV` = Symbolic link
> - `/dev/mapper/LinuxVG-LinuxLV` = Actual device

---

## 6. Extend Logical Volume (Online Resize)

### Steps to Grow LV

```bash
# 1. Extend LV size (+2GB)
sudo lvextend -L +2G /dev/LinuxVG/LinuxLV

# 2. Resize filesystem to use new space
sudo resize2fs /dev/LinuxVG/LinuxLV    # For ext4/ext3/ext2
# For XFS: xfs_growfs /testdir

```

> ⚠️ Critical:
> 
> - **ext4**: Use `resize2fs` (can grow online).
> - **XFS**: Use `xfs_growfs MOUNT_POINT` (must be mounted).

---

## 7. Remove LVM Setup (Cleanup)

### Step-by-Step Removal

```bash
# 1. Unmount LV
sudo umount /dev/mapper/LinuxVG-LinuxLV

# 2. Remove LV
sudo lvremove /dev/mapper/LinuxVG-LinuxLV

# 3. Remove VG
sudo vgremove LinuxVG

# 4. Remove PVs
sudo pvremove /dev/sda1 /dev/sda2

```

### Optional: Delete Partitions

```bash
sudo fdisk /dev/sda
# In fdisk: d → 1 → d → 2 → w
sudo partprobe

```

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| Create PV | `pvcreate /dev/sdX1` |
| Create VG | `vgcreate VG_NAME /dev/sdX1` |
| Extend VG | `vgextend VG_NAME /dev/sdX2` |
| Create LV | `lvcreate -L 5G -n LV_NAME VG_NAME` |
| Format LV | `mkfs.ext4 /dev/VG_NAME/LV_NAME` |
| Mount LV | `mount /dev/VG_NAME/LV_NAME /mount` |
| Extend LV | `lvextend -L +2G /dev/VG_NAME/LV_NAME` |
| Resize ext4 | `resize2fs /dev/VG_NAME/LV_NAME` |
| Remove LV | `lvremove /dev/mapper/VG_NAME-LV_NAME` |
| Remove VG | `vgremove VG_NAME` |
| Remove PV | `pvremove /dev/sdX1` |

### Key Notes

- **Always unmount** before removing LVs.
- **Resize filesystem** after extending LV (`resize2fs` for ext4, `xfs_growfs` for XFS).
- Use **`L`** for absolute size (`5G`), **`l`** for % of VG (`l 50%FREE`).

> 💡 Golden Rule:
> 
> 
> LVM gives **flexibility**—but always:
> 
> 1. Backup data before major changes.
> 2. Verify with `pvs`/`vgs`/`lvs` at each step.
> 3. Resize filesystem **after** extending LV.