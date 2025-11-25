# 014: Filesystems & Mounting in Linux | Manage Filesystems

## 1. Understanding Linux Filesystems

### View Supported Filesystems

```bash
cat /proc/filesystems    # Kernel-supported filesystems
cat /etc/filesystems     # Filesystems tried by `mount` (if no type specified)
man fs                   # Manual: overview of filesystem types

```

> ✅ Common filesystems: ext4, xfs, ext2, vfat, ntfs
> 

---

## 2. Creating and Inspecting Filesystems

### Create ext2 Filesystem

```bash
sudo mke2fs /dev/sdb     # Format entire disk as ext2

```

> ⚠️ Warning: This destroys all data on /dev/sdb!
> 

### Verify Filesystem

```bash
sudo blkid /dev/sdb      # Show UUID, TYPE
lsblk -f                 # List disks with filesystem info

```

### Inspect Filesystem Details

```bash
sudo tune2fs -l /dev/sdb | grep -i "block count"  # Total blocks
sudo tune2fs -l /dev/sdb | grep -i "reserved"     # Reserved blocks for root

```

### Adjust Reserved Blocks

By default, 5% of space is reserved for root (prevents system crash when full).

Reduce to 10% (example—usually you **decrease** this, e.g., to 1%):

```bash
sudo tune2fs -m 10 /dev/sdb    # Set 10% reserved blocks
sudo tune2fs -l /dev/sdb | grep -i "reserved"  # Verify

```

> 💡 Tip: For large data disks, reduce reserved space:
> 
> 
> `sudo tune2fs -m 1 /dev/sdb` → only 1% reserved.
> 

---

## 3. Mounting Filesystems

### Temporary Mount

```bash
sudo mkdir /test
sudo mount -t ext2 /dev/sdb /test    # Mount as ext2
df -Th                               # Verify: shows type and usage

```

### Read-Only Mount

```bash
sudo mount -t ext2 -o ro /dev/sdb /test
mount | grep sdb                     # Shows "ro" (read-only)

```

### Mount with Special Options

| Option | Effect |
| --- | --- |
| `noexec` | Prevent executing binaries |
| `noacl` | Disable ACL support |
| `ro` | Read-only |
| `rw` | Read-write (default) |

### Example: `noexec`

```bash
sudo mount -t ext2 -o noexec /dev/sdb /test
cp /sbin/ifconfig /test/
cd /test
./ifconfig    # ❌ Fails: "Permission denied"

```

### Example: `noacl`

```bash
sudo mount -t ext2 -o noacl /dev/sdb /test
touch /test/a
setfacl -m u:zybi:7 /test/a    # ❌ Fails: ACLs disabled

```

---

## 4. Persistent Mounts with `/etc/fstab`

### Why `/etc/fstab`?

- Defines filesystems to mount at **boot**.
- Avoids manual mounting after reboot.

### View Current Mounts

```bash
cat /etc/mtab        # Currently mounted filesystems (deprecated; use `mount` instead)
mount                # Preferred way to see active mounts
df                   # Disk usage + mount points

```

### Configure Persistent Mount

1. Get UUID (more reliable than `/dev/sdb`):
    
    ```bash
    sudo blkid /dev/sdb
    # Output: /dev/sdb: UUID="a1b2c3d4..." TYPE="ext2"
    
    ```
    
2. Edit `/etc/fstab`:
    
    ```bash
    sudo vi /etc/fstab
    
    ```
    
    Add line:
    
    ```
    UUID=a1b2c3d4...  /test  ext2  defaults  0 2
    
    ```
    
    **Field Explanation**:
    
    | Field | Purpose |
    | --- | --- |
    | `UUID=...` | Device identifier |
    | `/test` | Mount point |
    | `ext2` | Filesystem type |
    | `defaults` | Mount options (`rw,suid,dev,exec,auto,nouser,async`) |
    | `0` | Dump (backup) frequency (0 = never) |
    | `2` | fsck order (1=root, 2=others, 0=skip) |
3. Test Configuration:
    
    ```bash
    sudo mount -a    # Mount all entries in /etc/fstab
    df               # Verify /test is mounted
    
    ```
    

> ⚠️ Critical:
> 
> - **Always test with `mount -a`** before rebooting.
> - A bad `/etc/fstab` can **prevent system boot**!

---

## 5. Filesystem Checks (`fsck`)

### Check Filesystem Integrity

```bash
sudo umount /test          # Must be unmounted!
sudo fsck /dev/sdb         # Check and repair

```

> 🔍 Note: Never run fsck on a mounted filesystem—it can cause corruption!
> 

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| List filesystems | `cat /proc/filesystems` |
| Create ext2 | `sudo mke2fs /dev/sdX` |
| Get UUID | `sudo blkid /dev/sdX` |
| Mount temporarily | `sudo mount -t ext2 /dev/sdX /mount` |
| Mount read-only | `sudo mount -o ro /dev/sdX /mount` |
| Mount noexec | `sudo mount -o noexec /dev/sdX /mount` |
| Edit fstab | `sudo vi /etc/fstab` |
| Test fstab | `sudo mount -a` |
| Check filesystem | `sudo fsck /dev/sdX` (unmounted!) |

### `/etc/fstab` Format

```
<device> <mount> <type> <options> <dump> <pass>
UUID=... /test   ext2   defaults   0       2

```

> 💡 Golden Rules:
> 
> - **Always use UUIDs** in `/etc/fstab` (device names like `/dev/sdb` can change).
> - **Unmount before running `fsck` or repartitioning**.
> - **Test `/etc/fstab` with `mount -a`** before rebooting.
> - Use `noexec`/`noacl` for security on shared/data partitions.