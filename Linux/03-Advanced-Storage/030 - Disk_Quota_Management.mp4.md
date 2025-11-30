# 030: Disk Quota Management for Users & Groups on EXT4

## 1. Understanding Disk Quotas

### Key Concepts

- **Soft Limit**: Warning threshold (user can exceed temporarily).
- **Hard Limit**: Absolute maximum (blocks further writes).
- **Grace Period**: Time allowed to exceed soft limit (default: 7 days).
- **Block Quotas**: Limit disk space (in 1KB blocks).
- **Inode Quotas**: Limit number of files.

> ⚠️ Requirements:
> 
> - Filesystem must support quotas (`ext4`, `xfs`)
> - Mount options: `usrquota`, `grpquota`

---

## 2. Preparing the Quota Environment

### Create Dedicated Partition

```bash
# Create partition on /dev/sdb (not sda—avoid system disk!)
sudo fdisk /dev/sdb
# Commands: n → p → 1 → [Enter] → [Enter] → w
sudo mkfs.ext4 /dev/sdb1

```

### Install Quota Tools

```bash
# RHEL/CentOS
sudo yum install quota -y

# Verify
rpm -qa | grep quota

```

### Create Users and Groups

```bash
sudo useradd zybi
sudo useradd hamza
sudo useradd mazz
sudo groupadd quotagrp

# Assign users to group
sudo usermod -g quotagrp hamza
sudo usermod -g quotagrp mazz

```

---

## 3. Enabling Quotas on Filesystem

### Mount with Quota Options

```bash
# Create mount point
sudo mkdir /quotadir

# Temporary mount
sudo mount /dev/sdb1 /quotadir

# Make persistent in /etc/fstab
echo "/dev/sdb1 /quotadir ext4 defaults,usrquota,grpquota 0 0" | sudo tee -a /etc/fstab

# Remount with quota options
sudo mount -o remount /quotadir
mount | grep quotadir    # Verify usrquota,grpquota

```

> 💡 Critical:
> 
> 
> Use **`/dev/sdb1`** (not `/dev/sda1`) to avoid system disk conflicts.
> 

---

## 4. Initializing Quota Files

### Create Quota Database

```bash
# Generate aquota.user and aquota.group
sudo quotacheck -cug /quotadir

# Verify files created
ls -l /quotadir/aquota.*
# -rw------- 1 root root ... /quotadir/aquota.group
# -rw------- 1 root root ... /quotadir/aquota.user

```

> ⚠️ Flags:
> 
> - `c` = create files, `u` = user quotas, `g` = group quotas

### Enable Quotas

```bash
sudo quotaon /quotadir
sudo quotaon -p /quotadir    # Verify active

```

---

## 5. Setting User and Group Quotas

### Set User Quota (zybi)

```bash
sudo edquota zybi

```

Edit to:

```
/dev/sdb1      0      100000      200000          0       10       15
               ^         ^           ^             ^        ^        ^
           blocks     soft       hard         inodes    soft     hard

```

- **Blocks**: 100MB soft, 200MB hard (1 block = 1KB)
- **Inodes**: 10 files soft, 15 files hard

### Set Group Quota (quotagrp)

```bash
sudo edquota -g quotagrp

```

Edit to:

```
/dev/sdb1      0      400000      500000          0       200      300

```

- **Blocks**: 400MB soft, 500MB hard
- **Inodes**: 200 files soft, 300 files hard

### Set Grace Period

```bash
# Set 24-hour grace period for zybi
sudo edquota -t
# Set:
# Block grace time: 24 hours
# Inode grace time: 24 hours

```

> 💡 Note:
> 
> 
> Grace period applies to **all users/groups** (global setting).
> 

---

## 6. Testing Quotas

### Prepare Directory

```bash
sudo chmod 777 /quotadir
sudo chgrp quotagrp /quotadir

```

### Test User Quota (zybi)

```bash
su - zybi
cd /quotadir

# Create files (within inode limit)
touch zybi.user{1..10}    # OK
touch zybi.user{11..15}   # OK (soft limit = 10, but grace period active)
touch zybi.user{16..17}   # FAIL (hard limit = 15)

# Test block quota
dd if=/dev/zero of=zybiuser1.txt bs=1M count=100    # 100MB → OK
dd if=/dev/zero of=zybiuser2.txt bs=1M count=200    # 200MB → FAIL (hard limit)

```

### Test Group Quota (hamza)

```bash
su - hamza
cd /quotadir
touch hamza.user{1..100}    # OK (group inode soft=200)
touch hamza.user{101..300}  # OK (within hard=300)

```

### Check Quota Usage

```bash
# As user
quota                    # View own quota
quota -g quotagrp        # View group quota (if member)

# As root
sudo repquota -a         # Report all user quotas
sudo repquota -g         # Report all group quotas

```

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Install quota | `yum install quota` |
| Enable quotas | `quotaon /mountpoint` |
| Disable quotas | `quotaoff /mountpoint` |
| Edit user quota | `edquota username` |
| Edit group quota | `edquota -g groupname` |
| Set grace period | `edquota -t` |
| Check usage | `quota`, `repquota -a` |
| Scan filesystem | `quotacheck -avug` |

---

## 8. Critical Best Practices

### ✅ Do

- **Always test quotas** with real user accounts
- **Set grace periods** to allow cleanup time
- **Monitor with `repquota`** regularly
- Use **separate partition** for quota-enabled directories

### ❌ Don't

- Enable quotas on **root filesystem** (risk of system lockup)
- Set **hard limits = soft limits** (no grace period)
- Forget to **remount with quota options** after fstab edit

### Troubleshooting

- **"Quota not enabled"**:
    
    ```bash
    sudo quotaon /quotadir
    
    ```
    
- **Quota files missing**:
    
    ```bash
    sudo quotacheck -cug /quotadir
    
    ```
    
- **User can exceed hard limit**:
Check if filesystem was **remounted without quota options**

---

## Summary Workflow

```bash
# 1. Prepare
fdisk → mkfs.ext4 → install quota → create users/groups

# 2. Enable quotas
mount with usrquota,grpquota → quotacheck -cug → quotaon

# 3. Set limits
edquota user → edquota -g group → edquota -t

# 4. Test
su user → create files → quota → repquota -a

```

> 💡 Golden Rule:
> 
> 
> **Quotas protect shared storage**. Always:
> 
> - Set **soft < hard** limits
> - Give **grace period** for cleanup
> - Test **before enforcing** in production!