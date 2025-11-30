# 013: Disk Management

## 1. Discovering Disks and Hardware

### Explanation

Before managing storage, identify available disks and their details.

### Key Commands

```bash
lsblk                     # List block devices (disks & partitions)
fdisk -l                  # List partition tables (MBR only)
dmesg | tail -20          # View recent kernel messages (e.g., disk detection)
lsscsi                    # List SCSI/SATA/NVMe devices
lshw -class disk          # Detailed hardware info (install with `sudo yum install lshw`)
cat /proc/scsi/scsi       # Legacy SCSI device info (may be empty on modern systems)

```

### Lab Steps

1. List current disks:
    
    ```bash
    lsblk
    
    ```
    
2. Check for new hardware:
    
    ```bash
    dmesg | grep -i sd
    
    ```
    

---

## 2. Rescanning for New Disks (Hot-Add)

### Explanation

When attaching a disk to a **running system** (e.g., VM or hot-swap server), rescan the SCSI/SATA bus to detect it.

### Rescan Command

```bash
# Rescan all SCSI hosts
ls /sysclass/scsi_host/ | while read host; do
  echo "- - -" > /sysclass/scsi_host/$host/scan
done

```

> ✅ Note: - - - means: channel, target, LUN (wildcard = scan all).
> 

### Verify Detection

```bash
lsblk    # New disk (e.g., /dev/sdb) should appear

```

---

## 3. Checking Disk Health (Optional)

> ⚠️ Warning: These commands can destroy data—use only on unused disks!
> 

### Bad Blocks Scan

```bash
badblocks -ws /dev/sdX    # Read/write test (destructive!)

```

### Wipe Disk (Emergency Only)

```bash
dd if=/dev/zero of=/dev/sdX bs=1M status=progress  # Erase entire disk

```

> ❌ Never run on system disks!
> 

---

## 4. Partitioning Disks

### MBR vs. GPT

| Feature | MBR | GPT |
| --- | --- | --- |
| Max partitions | 4 primary (or 3 primary + 1 extended) | 128+ |
| Max disk size | 2 TB | 9.4 ZB |
| Tools | `fdisk` | `gdisk`, `parted` |

---

### Creating MBR Partitions with `fdisk`

### Step-by-Step Lab

```bash
# 1. Start fdisk on disk (e.g., /dev/sdb)
sudo fdisk /dev/sdb

# 2. Create primary partition
Command (m for help): n
Partition type: p (primary)
Partition number: 1
First sector: [Enter] (default)
Last sector: +2G (2GB size)

# 3. Set partition type (8e = Linux LVM)
Command: t
Hex code: 8e

# 4. Write changes
Command: w

```

### Notify OS of Changes

```bash
sudo partprobe    # Reload partition table without reboot
lsblk             # Verify new partition (e.g., /dev/sdb1)

```

> 💡 Partition Types:
> 
> - `83` = Linux filesystem (default)
> - `8e` = Linux LVM
> - `82` = Linux swap

---

### Creating Extended Partitions (MBR Only)

```bash
fdisk /dev/sdb
n → e (extended) → [Enter] → +1G → w
partprobe

```

> ✅ Extended partitions act as containers for logical partitions.
> 

---

## 5. Formatting and Mounting Filesystems

### Create Filesystem

```bash
# XFS (RHEL 8 default)
sudo mkfs.xfs /dev/sdb1

# ext4 (alternative)
sudo mkfs.ext4 /dev/sdb1

```

### Mount Temporarily

```bash
sudo mkdir /Data
sudo mount /dev/sdb1 /Data
df -h /Data    # Verify mount

```

### Test Filesystem

```bash
cd /Data
touch test{a..z}    # Create files testa, testb, ..., testz
ls

```

### Unmount

```bash
sudo umount /Data

```

> ⚠️ Unmount before repartitioning—or you’ll get "device busy" errors.
> 

---

## 6. Key Files and Verification

### Partition Info

```bash
cat /proc/partitions    # Kernel’s view of partitions
fdisk -l /dev/sdb       # Detailed partition table
lsblk -f                # Show filesystems & mount points

```

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| List disks | `lsblk`, `fdisk -l` |
| Rescan for new disk | `echo "- - -" > /sys/class/scsi_host/host*/scan` |
| Create MBR partition | `fdisk /dev/sdX` → `n` → `p` → size → `w` |
| Create GPT partition | `gdisk /dev/sdX` |
| Reload partition table | `partprobe` |
| Format as XFS | `mkfs.xfs /dev/sdX1` |
| Mount | `mount /dev/sdX1 /mountpoint` |
| Unmount | `umount /mountpoint` |
| Verify mounts | `df -h`, `lsblk -f` |

> 💡 Golden Rules:
> 
> - **Always** run `partprobe` after partitioning.
> - **Never** partition/mount a disk that’s in use.
> - Use **GPT** for disks >2TB or >4 partitions.
> - **Backup data** before partitioning!