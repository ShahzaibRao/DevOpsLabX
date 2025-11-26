# 015: Converting EXT2 to EXT3/EXT4 & Managing Journaling 
---


## 1. Prepare a New Disk (Lab Setup)

### Rescan for New Disk (Hot-Add)

```bash
# Fix syntax from your notes (corrected)
for host in /sysclass/scsi_host/host*/; do
  echo "- - -" > "${host}scan"
done
lsblk    # Verify new disk (e.g., /dev/sdb)

```

### Create Partition

```bash
sudo fdisk /dev/sdb
# Commands in fdisk:
# n → p → [Enter] → [Enter] → w
sudo partprobe    # Reload partition table
lsblk             # Verify /dev/sdb1

```

---

## 2. Create EXT2 Filesystem

### Format and Mount

```bash
sudo mke2fs /dev/sdb1          # Create EXT2 (no journaling)
sudo blkid /dev/sdb1           # Note UUID and TYPE="ext2"
sudo mkdir /test
sudo mount /dev/sdb1 /test
df -hT                         # Confirm TYPE=ext2

# Add test data
cd /test
touch test{1..20}
cal > cal.txt
cd

```

---

## 3. Convert EXT2 → EXT3 (Enable Journaling)

### Steps

```bash
# 1. Unmount
sudo umount /test

# 2. Add journal (converts to EXT3)
sudo tune2fs -j /dev/sdb1

# 3. Verify
sudo blkid /dev/sdb1           # TYPE="ext3"
sudo tune2fs -l /dev/sdb1 | grep "Filesystem features"
# Output should include: has_journal

# 4. Remount and test
sudo mount /dev/sdb1 /test
cat /test/cal.txt              # Data intact

```

> ✅ Key Point:
> 
> - `j` adds an **internal journal** (default for EXT3).

---

## 4. Convert EXT3 → EXT4 (Add Modern Features)

### Steps

```bash
# 1. Unmount
sudo umount /test

# 2. Enable EXT4 features
sudo tune2fs -O extents,uninit_bg,dir_index /dev/sdb1

# 3. Verify
sudo blkid /dev/sdb1           # TYPE="ext4"
lsblk -f                       # Confirm ext4

# 4. Remount and test
sudo mount /dev/sdb1 /test
ls -l /test                    # Data intact
cat /test/cal.txt

```

### Key EXT4 Features Enabled

| Feature | Benefit |
| --- | --- |
| `extents` | Replaces block maps → better performance for large files |
| `uninit_bg` | Faster `fsck` |
| `dir_index` | Faster directory operations |

> ⚠️ Critical: After converting to EXT4, run e2fsck to fix on-disk structures:
> 
> 
> ```bash
> sudo e2fsck -f /dev/sdb1
> 
> ```
> 

---

## 5. Enable/Disable Journaling

### Check Current Journal Status

```bash
sudo tune2fs -l /dev/sdb1 | grep "Filesystem features"
# Look for: has_journal

```

### Disable Journaling (EXT4 → EXT2-like)

```bash
sudo umount /test
sudo tune2fs -O ^has_journal /dev/sdb1    # ^ = disable feature
sudo tune2fs -l /dev/sdb1 | grep "has_journal"  # Should be gone

```

### Re-enable Journaling

```bash
sudo tune2fs -O has_journal /dev/sdb1
sudo tune2fs -l /dev/sdb1 | grep "has_journal"  # Now present

```

> ⚠️ Warning:
> 
> - Disabling journaling **increases risk of corruption** on crash.
> - EXT4 **requires** `extents`—don’t disable it!

---

## 6. Persistent Mount with `/etc/fstab`

### Add Entry for New Filesystem

1. Get UUID:
    
    ```bash
    sudo blkid /dev/sdb1
    # Output: UUID="a1b2c3d4..." TYPE="ext4"
    
    ```
    
2. Edit `/etc/fstab`:
    
    ```bash
    sudo vi /etc/fstab
    
    ```
    
    Add line:
    
    ```
    UUID=a1b2c3d4...  /test  ext4  defaults  0 2
    
    ```
    
3. Test:
    
    ```bash
    sudo mount -a
    df -hT    # Verify mounted as ext4
    
    ```
    

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| Create EXT2 | `mke2fs /dev/sdX1` |
| EXT2 → EXT3 | `tune2fs -j /dev/sdX1` |
| EXT3 → EXT4 | `tune2fs -O extents,uninit_bg,dir_index /dev/sdX1` |
| Disable journal | `tune2fs -O ^has_journal /dev/sdX1` |
| Enable journal | `tune2fs -O has_journal /dev/sdX1` |
| Verify features | `tune2fs -l /dev/sdX1 \| grep features` |
| Fix after EXT4 convert | `e2fsck -f /dev/sdX1` |

### Critical Notes

- **Always unmount** before converting filesystems.
- **Run `e2fsck -f`** after enabling EXT4 features.
- **Never disable `extents`** on EXT4—it’s required.
- Use **UUIDs** in `/etc/fstab` for reliability.

> 💡 Golden Rule:
> 
> 
> Journaling protects against corruption—only disable it for **specialized workloads** (e.g., read-only media). For general use, **keep journaling enabled**.
>