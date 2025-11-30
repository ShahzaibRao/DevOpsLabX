# 027: Creating and Managing RAID-6 in Linux
---


## 1. Understanding RAID-6

### Key Concepts

- **Striping with Dual Distributed Parity**: Data + **two parity blocks** spread across all disks.
- **Fault Tolerance**: Survives **failure of up to 2 disks simultaneously**.
- **Capacity**: Usable space = **(N-2) × smallest disk** (e.g., 4×2TB → 4TB usable).
- **Minimum Disks**: **4 disks required**.
- **Use Cases**:
    - Large storage arrays (>10TB)
    - Archival systems
    - Critical data where **rebuild time is long** (large disks)
    - Any environment where **two-disk failure protection** is essential

> ⚠️ Trade-offs:
> 
> - **Write penalty**: Slower writes (must calculate two parity blocks)
> - **CPU overhead**: Higher than RAID-5 (but negligible on modern systems)

---

## 2. Creating RAID-6 Array

### Prepare Disks

```bash
# Create partitions on 4+ disks (e.g., /dev/sdb-sde)
for disk in sdb sdc sdd sde; do
  sudo fdisk /dev/$disk <<EOF
n
p
1

t
fd
w
EOF
done

sudo partprobe    # Reload partition table
lsblk             # Verify sdb1, sdc1, sdd1, sde1

```

### Create RAID-6

```bash
# Create RAID-6 with 4 disks
sudo mdadm --create /dev/md0 -l 6 -n 4 /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1

# Verify
cat /proc/mdstat    # Shows [UUUU] (all disks active)
sudo mdadm -D /dev/md0    # Detailed status (note "Active Devices: 4")

```

> 💡 Initial Sync:
> 
> 
> RAID-6 initializes dual parity—check `/proc/mdstat` for `[=>...]` progress (can take many hours).
> 

### Format and Mount

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir /raid6
sudo mount /dev/md0 /raid6
df -hT              # Verify mounted

# Add test data
echo "cal" | sudo tee /raid6/cal
echo "date" | sudo tee /raid6/date

```

---

## 3. Simulating Dual Disk Failure

### Fail First Disk

```bash
# Check current status
cat /proc/mdstat    # Should show [UUUU]

# Fail /dev/sdb1
sudo mdadm /dev/md0 -f /dev/sdb1

# Verify degraded state (1 disk failed)
cat /proc/mdstat    # Shows [U_UU]
sudo mdadm -D /dev/md0    # Shows /dev/sdb1 as "faulty"

```

### Fail Second Disk

```bash
# Fail /dev/sdc1
sudo mdadm /dev/md0 -f /dev/sdc1

# Verify still operational (2 disks failed)
cat /proc/mdstat    # Shows [U__U]
ls /raid6           # Data still accessible!

```

> ✅ RAID-6 survives dual failure:
> 
> - Reads work (using dual parity to reconstruct data)
> - Writes work (but very slow—parity recalculated on every write)

---

## 4. Replacing Failed Disks

### Steps to Replace Both Disks

```bash
# 1. Unmount (recommended)
sudo umount /raid6

# 2. Remove both faulty disks
sudo mdadm /dev/md0 -r /dev/sdb1 /dev/sdc1

# 3. Prepare new disks (e.g., /dev/sdf1, /dev/sdg1)
for disk in sdf sdg; do
  sudo fdisk /dev/$disk <<EOF
n
p
1

t
fd
w
EOF
done
sudo partprobe

# 4. Add new disks to array
sudo mdadm /dev/md0 -a /dev/sdf1 /dev/sdg1

# 5. Verify rebuild (happens sequentially)
cat /proc/mdstat    # Shows [U__U] → [UU_U] → [UUUU]
sudo mdadm -D /dev/md0    # Shows "rebuilding" status

```

> 🔍 Rebuild Process:
> 
> - Disks rebuild **one at a time** (not simultaneously)
> - Array remains **usable during rebuild**
> - Total rebuild time = time for **two full disk syncs**

### Remount and Verify

```bash
sudo mount /dev/md0 /raid6
cat /raid6/cal      # Data intact!

```

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Create RAID-6 | `mdadm --create /dev/md0 -l 6 -n 4 /dev/sdX1 /dev/sdY1 /dev/sdZ1 /dev/sdW1` |
| Fail disk(s) | `mdadm /dev/md0 -f /dev/sdX1 [/dev/sdY1]` |
| Remove disk(s) | `mdadm /dev/md0 -r /dev/sdX1 [/dev/sdY1]` |
| Add disk(s) | `mdadm /dev/md0 -a /dev/sdA1 [/dev/sdB1]` |
| View status | `cat /proc/mdstat`, `mdadm -D /dev/md0` |
| Stop array | `mdadm -S /dev/md0` |

---

## 6. Critical Best Practices

### Before Deployment

1. **Use enterprise-grade disks** (TLER/ERC enabled) to prevent false failures during rebuilds.
2. **Save RAID config**:
    
    ```bash
    sudo mdadm --detail --scan | sudo tee /etc/mdadm.conf
    
    ```
    
3. **Size appropriately**:
    - Ideal for **large arrays** (6+ disks)
    - Avoid for small setups (RAID-1 or RAID-10 better for <4 disks)

### During Operation

- **Monitor disk health aggressively**:
    
    ```bash
    sudo smartctl -a /dev/sdb | grep -i "reallocated\\|pending"
    
    ```
    
- **Schedule regular scrubs** (detect latent errors):
    
    ```bash
    echo check | sudo tee /sys/block/md0/md/sync_action
    
    ```
    

### After Replacement

- **Verify data integrity**:
    
    ```bash
    sudo fsck -f /dev/md0    # After unmounting
    
    ```
    
- **Update initramfs** (for boot arrays):
    
    ```bash
    sudo dracut -f    # RHEL/CentOS
    
    ```
    

---

## 7. RAID-6 vs. Alternatives

| Scenario | Recommendation |
| --- | --- |
| **4-8 disks, critical data** | RAID-6 |
| **>8 disks or >10TB total** | RAID-6 |
| **Max performance + redundancy** | RAID-10 (but 50% capacity loss) |
| **Budget storage, non-critical** | RAID-5 (with <2TB disks) |

> 💡 Modern Alternative:
> 
> 
> **ZFS RAID-Z2** provides similar dual-parity protection + built-in checksums + snapshots.
> 

---

## Summary Workflow

### Create RAID-6

```bash
fdisk → t fd → mdadm --create /dev/md0 -l 6 -n 4 /dev/sdX1..sdW1 → mkfs → mount

```

### Replace Failed Disks

```bash
mdadm -f /dev/sdX1 /dev/sdY1 → mdadm -r /dev/sdX1 /dev/sdY1 → add new disks → mdadm -a /dev/sdA1 /dev/sdB1

```

> 💡 Golden Rule:
> 
> 
> **RAID-6 = Dual-Disk Fault Tolerance**. Always:
> 
> - Use for **large arrays** where rebuild time is long
> - Replace failed disks **immediately** (third failure = total data loss!)
> - Monitor disk health **aggressively**—RAID-6 is your last line of defense!