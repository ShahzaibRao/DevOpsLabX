# 022: Migrating LVM Data Disk Between Machines

## 1. Understanding Cross-Machine LVM Migration

### Purpose

- Move an entire LVM setup (VG + LVs) from one machine to another.
- Common for hardware upgrades, server replacements, or disaster recovery.

### Key Concepts

- **`vgexport`**: Marks a Volume Group as **inactive and exportable** (prevents auto-activation).
- **`vgimport`**: Makes an exported VG **available** on the new machine.
- **No data copying needed**—just physically move the disk!

> ✅ Critical:
> 
> 
> The disk must be **physically moved** (or attached via SAN/iSCSI).
> 
> Never run both machines with the same LVM disk attached simultaneously!
> 

---

## 2. Lab Setup: Prepare LVM on Source Machine

### Create LVM

```bash
# Create partition on /dev/sdb (5GB+ disk)
sudo fdisk /dev/sdb
# Commands: n → p → 1 → [Enter] → [Enter] → t → 8e → w
sudo partprobe

# Create LVM
sudo pvcreate /dev/sdb1
sudo vgcreate VolGrp /dev/sdb1
sudo lvcreate -L 2G -n LogVol VolGrp    # Fixed: -L for size (not -l)
sudo mkfs.ext4 /dev/VolGrp/LogVol

# Mount and add data
sudo mkdir /data
sudo mount /dev/VolGrp/LogVol /data
sudo touch /data/{1..11}
sudo umount /data

```

> ⚠️ Note:
> 
> - `l 2000` = 2000 **extents** (size depends on VG extent size).
> 
> Use `-L 2G` for predictable 2GB size.
> 

---

## 3. Export LVM on Source Machine

### Steps to Safely Export

```bash
# 1. Ensure filesystem is unmounted
df -hT    # Verify /data not mounted

# 2. Deactivate VG
sudo vgchange -an VolGrp    # -an = deactivate all LVs

# 3. Export VG
sudo vgexport VolGrp

# 4. Verify
sudo pvs    # Shows PV but no VG
sudo vgs    # Empty
sudo lvscan # Shows "inactive" (but won't list exported VG)

```

### Shutdown Source Machine

```bash
sudo init 0    # Power off

```

> 🔍 What vgexport does:
> 
> - Removes VG from LVM's active configuration
> - Preserves metadata on disk for import later

---

## 4. Import LVM on Target Machine

### Attach Disk & Boot

1. Physically move disk to new machine (or attach via SAN).
2. Boot target machine (adjust BIOS boot order if needed to avoid booting from moved disk).

### Import LVM

```bash
# 1. Scan for PVs/VGs
sudo pvscan    # Should detect /dev/sdb1
sudo vgscan    # Detects exported VG "VolGrp"

# 2. Import VG
sudo vgimport VolGrp

# 3. Activate VG
sudo vgchange -ay VolGrp    # -ay = activate all LVs

# 4. Verify
sudo vgs    # VolGrp appears
sudo lvs    # LogVol listed

```

### Mount and Verify Data

```bash
sudo mkdir /data
sudo mount /dev/VolGrp/LogVol /data
ls /data    # Original files {1..11} intact!

```

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Deactivate VG | `vgchange -an VG_NAME` |
| Export VG | `vgexport VG_NAME` |
| Scan PVs | `pvscan` |
| Scan VGs | `vgscan` |
| Import VG | `vgimport VG_NAME` |
| Activate VG | `vgchange -ay VG_NAME` |

---

## 6. Critical Notes & Best Practices

### Before Migration

- **Unmount filesystems** to prevent corruption.
- **Deactivate VG** (`vgchange -an`) before export.
- **Backup LVM config**:
    
    ```bash
    sudo vgcfgbackup VolGrp
    
    ```
    

### After Migration

- **Update `/etc/fstab`** on target machine (if auto-mounting):
    
    ```bash
    echo "UUID=$(sudo blkid -s UUID -o value /dev/VolGrp/LogVol) /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
    
    ```
    
- **Never attach the same LVM disk to two machines at once**—causes metadata corruption!

### Troubleshooting

- **"Volume group not found" after import?**
Run `sudo vgscan` first.
- **LVs not activating?**
Check with `sudo lvscan` and activate manually:
    
    ```bash
    sudo lvchange -ay /dev/VolGrp/LogVol
    
    ```
    

---

## Summary Workflow

### Source Machine

```bash
umount /data
vgchange -an VolGrp
vgexport VolGrp
init 0

```

### Target Machine

```bash
pvscan
vgscan
vgimport VolGrp
vgchange -ay VolGrp
mount /dev/VolGrp/LogVol /data

```

> 💡 Golden Rule:
> 
> 
> **Always deactivate and export** on the source machine.
> 
> **Always scan and import** on the target machine.
> 
> Never skip `vgchange -an`/`-ay`—it prevents metadata conflicts!
>