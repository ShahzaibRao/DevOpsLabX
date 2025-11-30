# 029: Creating and Managing Virtual Data Optimizer (VDO) 
---


## 1. Understanding VDO (Virtual Data Optimizer)

### Key Concepts

- **Block-level deduplication**: Identical data blocks stored **once** (saves space).
- **Compression**: LZ4 compression reduces block size.
- **Thin provisioning**: Logical size can be **larger than physical disk** (e.g., 300GB logical on 10GB disk).
- **Use Cases**:
    - VM/image repositories (many identical OS files)
    - Backup targets
    - File servers with duplicate data

> ⚠️ Critical Notes:
> 
> - **Not for databases** (random I/O hurts performance)
> - **Not for encrypted data** (deduplication fails on unique ciphertext)
> - Requires **RHEL 8+** or **CentOS 8+**

---

## 2. Installing VDO

### Prerequisites

```bash
# Enable repositories (if needed)
sudo dnf repolist all

# Install VDO packages
sudo dnf install kmod-kvdo vdo -y

# Start and enable service
sudo systemctl start vdo
sudo systemctl enable vdo
sudo systemctl status vdo    # Verify active

```

> 💡 Note:
> 
> 
> `kmod-kvdo` = kernel module, `vdo` = user-space tools.
> 

---

## 3. Creating a VDO Volume

### Using Raw Disk (No Partition)

```bash
# Create VDO on /dev/sdb (10GB physical)
sudo vdo create --name=vdo1 --device=/dev/sdb --vdoLogicalSize=300G

# Verify
lsblk    # Shows /dev/mapper/vdo1 (300G logical)
sudo vdostats --human-readable    # Shows physical vs logical usage

```

> ⚠️ Critical Flags:
> 
> - `-vdoLogicalSize=300G`: Logical size (can be 10x physical)
> - **No partition needed**—VDO uses raw disk

### If Creation Fails

```bash
# Reboot if kernel module issues
sudo init 0
# After reboot, retry creation

```

---

## 4. Using VDO with LVM

### Create LVM on VDO

```bash
# Create PV on VDO device
sudo pvcreate /dev/mapper/vdo1

# Create VG and LVs
sudo vgcreate vdo1vg /dev/mapper/vdo1
sudo lvcreate -n vdo1lv1 -L 50G vdo1vg
sudo lvcreate -n vdo1lv2 -L 50G vdo1vg

# Format with -K (disable lazy init for VDO)
sudo mkfs.ext4 -K /dev/vdo1vg/vdo1lv1
sudo mkfs.ext4 -K /dev/vdo1vg/vdo1lv2

```

> 💡 Why -K?
> 
> 
> Disables `lazy_itable_init`—prevents background initialization from interfering with VDO.
> 

---

## 5. Mounting VDO Volumes

### Mount with `discard` Option

```bash
sudo mkdir /v01 /v02
sudo mount -o discard /dev/vdo1vg/vdo1lv1 /v01
sudo mount -o discard /dev/vdo1vg/vdo1lv2 /v02
df -hT

```

> ✅ discard Option:
> 
> 
> Allows filesystem to **notify VDO** of unused blocks (improves space reclamation).
> 

---

## 6. Testing Deduplication & Compression

### Generate Duplicate Data

```bash
# Copy same data to both volumes
cd /v01
cal > cal.txt
cp -r /usr/sbin/* .    # ~100MB of binaries

cd /v02
cp -r /usr/sbin/* .    # Identical data!

# Check space savings
sudo vdostats --human-readable

```

### Expected Output

```bash
$ sudo vdostats --human-readable
Device           1K-blocks   Used  Available  Use% Space saving%
/dev/mapper/vdo1   10485760  20480   10465280    0%          75%

```

- **Physical used**: ~20MB (not 200MB!)
- **Space saving**: ~75% (due to deduplication + compression)

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Create VDO | `vdo create --name=vdo1 --device=/dev/sdX --vdoLogicalSize=300G` |
| View stats | `vdostats --human-readable` |
| Detailed stats | `vdostats --verbose /dev/mapper/vdo1` |
| Stop VDO | `vdo stop --name=vdo1` |
| Remove VDO | `vdo remove --name=vdo1` |
| Mount with trim | `mount -o discard /dev/vg/lv /mnt` |

---

## 8. Critical Best Practices

### ✅ Do

- Use for **duplicate-heavy workloads** (VMs, logs, backups)
- Always mount with **`discard`**
- Format with **`K`** (disable lazy init)
- Monitor with **`vdostats`**

### ❌ Don't

- Use for **databases** (MySQL, PostgreSQL)
- Use for **encrypted data** (LUKS, etc.)
- Expect performance gains on **random I/O workloads**

### Performance Tuning

- **Disable atime** (reduces writes):
    
    ```bash
    mount -o discard,noatime /dev/vg/lv /mnt
    
    ```
    
- **Use XFS** for large files:
    
    ```bash
    mkfs.xfs -K /dev/vg/lv
    
    ```
    

---

## 9. Troubleshooting

### Issue: "VDO not starting after reboot"

**Fix**:

```bash
sudo systemctl enable vdo
sudo vdo start --all

```

### Issue: "No space left on device" despite free space

**Cause**: Physical disk full (logical size > physical)
**Fix**:

```bash
# Check physical usage
sudo vdostats

# Delete data or extend physical disk

```

### Issue: Poor deduplication ratio

**Check**:

- Data is **truly duplicate** (same files, not similar)
- Files are **not encrypted/compressed** already

---

## Summary Workflow

```bash
# 1. Install
dnf install kmod-kvdo vdo -y

# 2. Create VDO
vdo create --name=vdo1 --device=/dev/sdb --vdoLogicalSize=300G

# 3. Use with LVM
pvcreate /dev/mapper/vdo1 → vgcreate → lvcreate → mkfs.ext4 -K

# 4. Mount with discard
mount -o discard /dev/vg/lv /mnt

# 5. Monitor
vdostats --human-readable

```

> 💡 Golden Rule:
> 
> 
> **VDO = Space Savings for Duplicate Data**. Always:
> 
> - Use for **VMs/backups**, not databases
> - Monitor **physical usage** (not logical!)
> - Mount with **`discard`** for optimal space reclamation!