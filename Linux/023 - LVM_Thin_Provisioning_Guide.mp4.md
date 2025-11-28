# 023: Setting Up Thin Provisioning Volumes in LVM



## 1. Understanding Thin Provisioning

### Key Concepts

- **Thin Pool**: A shared storage pool that allocates space **on-demand**.
- **Thin Volumes**: Virtual volumes that **overcommit** space (e.g., create 20GB of thin volumes on a 15GB pool).
- **Benefits**:
    - Efficient storage utilization
    - Instant volume creation
    - Snapshots with minimal overhead

> ⚠️ Critical Warning:
> 
> 
> Thin provisioning risks **pool exhaustion**—monitor usage closely!
> 

---

## 2. Lab Setup: Create Thin Pool

### Prepare Disk

```bash
# Create 20GB partition on /dev/sdb
sudo fdisk /dev/sdb
# Commands: n → p → 1 → [Enter] → +20G → w
sudo partprobe

# Create PV and VG with custom extent size (32MB)
sudo pvcreate /dev/sdb1
sudo vgcreate -s 32M vg_thin /dev/sdb1    # -s = physical extent size
sudo vgdisplay vg_thin

```

> 💡 Why 32MB extents?
> 
> 
> Larger extents reduce metadata overhead for large thin pools.
> 

### Create Thin Pool

```bash
# Create 15GB thin pool
sudo lvcreate -L 15G --thinpool thin_pool vg_thin
sudo lvs    # Shows thin_pool (type: thin-pool)

```

---

## 3. Create Thin Volumes

### Create Three 5GB Thin Volumes

```bash
# Note: Correct VG name is "vg_thin" (not "vg-thin")
sudo lvcreate -V 5G --thin -n thin_vol_client1 vg_thin/thin_pool
sudo lvcreate -V 5G --thin -n thin_vol_client2 vg_thin/thin_pool
sudo lvcreate -V 5G --thin -n thin_vol_client3 vg_thin/thin_pool
sudo lvs    # Shows thin volumes (type: thin)

```

> ✅ Key Point:
> 
> - `V` = **virtual size** (logical size visible to OS)
> 
> Actual space used = only what’s written
> 

### Format and Mount

```bash
sudo mkdir -p /mnt/client{1..3}

# Format
sudo mkfs.ext4 /dev/vg_thin/thin_vol_client1
sudo mkfs.ext4 /dev/vg_thin/thin_vol_client2
sudo mkfs.ext4 /dev/vg_thin/thin_vol_client3

# Mount
sudo mount /dev/vg_thin/thin_vol_client1 /mnt/client1
sudo mount /dev/vg_thin/thin_vol_client2 /mnt/client2
sudo mount /dev/vg_thin/thin_vol_client3 /mnt/client3

```

---

## 4. Test Thin Provisioning

### Write Data and Monitor Usage

```bash
# Client1: Write 1GB
sudo fallocate -l 1G /mnt/client1/test.img

# Client2: Write 1GB
sudo fallocate -l 1G /mnt/client2/test.img

# Client3: Write 1GB
sudo fallocate -l 1G /mnt/client3/test.img

# Check actual usage
sudo lvs    # Look at "Data%" for thin_pool
sudo vgs    # Shows pool usage

```

> 🔍 Observe:
> 
> - Each thin volume shows **5GB virtual size**
> - Thin pool shows **~3GB actual usage** (1GB per client)

---

## 5. Extend Thin Pool

### Add More Space to Pool

```bash
# Create second partition on /dev/sdb
sudo fdisk /dev/sdb
# Commands: n → p → 2 → [Enter] → [Enter] → t → 8e → w
sudo partprobe

# Add to VG and extend pool
sudo pvcreate /dev/sdb2
sudo vgextend vg_thin /dev/sdb2
sudo lvextend -L +15G /dev/vg_thin/thin_pool    # Pool now 30GB
sudo lvs    # Verify thin_pool size

```

### Create Additional Thin Volume

```bash
sudo lvcreate -V 5G --thin -n thin_vol_client4 vg_thin/thin_pool
sudo mkfs.ext4 /dev/vg_thin/thin_vol_client4
sudo mkdir /mnt/client4
sudo mount /dev/vg_thin/thin_vol_client4 /mnt/client4

```

---

## 6. Monitoring Thin Provisioning

### Critical Commands

```bash
# View pool usage
sudo lvs -o +data_percent,metadata_percent vg_thin/thin_pool

# View all thin volumes
sudo lvs -a -o +pool_lv,origin vg_thin

# Check for pool exhaustion warnings
sudo dmesg | grep thin

```

### Pool Exhaustion Symptoms

- Writes hang or fail with "No space left on device"
- `dmesg` shows: `thin pool full`

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Create thin pool | `lvcreate -L 15G --thinpool POOL_NAME VG` |
| Create thin volume | `lvcreate -V 5G --thin -n VOL_NAME VG/POOL` |
| Extend pool | `lvextend -L +15G /dev/VG/POOL` |
| Add PV to VG | `vgextend VG /dev/sdX2` |
| Monitor usage | `lvs -o +data_percent VG/POOL` |

---

## 8. Best Practices & Warnings

### ⚠️ Critical Rules

1. **Never overcommit excessively**:
Total virtual size ≤ 2x physical pool size (adjust based on workload).
2. **Monitor pool usage**:
Set alerts at 80% usage:
    
    ```bash
    # Add to cron
    lvs -o data_percent --noheadings vg_thin/thin_pool | awk '{if ($1 > 80) print "ALERT: Pool >80% full!"}'
    
    ```
    
3. **Extend pool proactively**:
Don’t wait for exhaustion—extend when at 70% usage.

### 💡 Pro Tips

- **Use larger extents** (`s 32M` or `64M`) for pools >100GB.
- **Thin snapshots** are instant and space-efficient:
    
    ```bash
    lvcreate -s -n snap1 /dev/vg_thin/thin_vol_client1
    
    ```
    
- **Auto-extend pools** with `lvm.conf` (advanced):
    
    ```
    thin_pool_autoextend_threshold = 80
    thin_pool_autoextend_percent = 20
    
    ```
    

---

## Summary Workflow

```bash
# 1. Create thin pool
vgcreate -s 32M vg_thin /dev/sdb1
lvcreate -L 15G --thinpool thin_pool vg_thin

# 2. Create thin volumes
lvcreate -V 5G --thin -n client1 vg_thin/thin_pool

# 3. Monitor and extend
lvs -o +data_percent
vgextend vg_thin /dev/sdb2
lvextend -L +15G /dev/vg_thin/thin_pool

```

> 💡 Golden Rule:
> 
> 
> Thin provisioning is **powerful but dangerous**—always monitor pool usage!
> 
> Treat thin pools like a shared resource: **plan for growth, watch usage, extend early**.
>