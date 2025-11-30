# 048: SELinux Information Gathering & Multi-Level Security (MLS/MCS)

## 1. SELinux Information Gathering Tools

### Core Diagnostic Commands

| Command | Purpose |
| --- | --- |
| `sestatus` | Show SELinux status, mode, policy type |
| `id -Z` | Display current user's SELinux context |
| `ls -Z` | Show file/directory SELinux context |
| `ps -eZ` | List all processes with SELinux contexts |
| `seinfo` | Query SELinux policy components |
| `sesearch` | Search SELinux policy rules |
| `avcstat` | Show SELinux denial statistics |

### Advanced Policy Analysis

```bash
# Count total domains in policy
seinfo -adomain -x | wc -l

# List all domains
seinfo -adomain

# Search allow rules
sesearch --allow

# Search dontaudit rules (denials that aren't logged)
sesearch --dontaudit

# Search role transitions
sesearch --role_allow -t unconfined_r

```

> 💡 Key Insight:
> 
> 
> These tools help **diagnose denials** and **understand policy structure** without reading raw policy files.
> 

---

## 2. Multi-Level Security (MLS) & Multi-Category Security (MCS)

### Core Concepts

- **MLS**: Implements **Bell-LaPadula model** (no read up, no write down)
- **MCS**: Commercial variant using **categories** instead of classification levels
- **Security Level Format**: `sensitivity:category` (e.g., `s0:c0.c1023`)

### Sensitivity Levels (MLS)

| Level | Numeric | Access |
| --- | --- | --- |
| **Unclassified** | `s0` | Everyone |
| **Confidential** | `s1` | Restricted |
| **Secret** | `s2-s14` | Highly restricted |
| **Top Secret** | `s15` | Extremely restricted |

### Category Ranges (MCS)

- **Categories**: `c0` to `c1023` (1024 possible categories)
- **Used for**: Departmental separation (Finance=c0, HR=c1, etc.)

> 🔍 Context Example:
> 
> 
> `user_u:staff_r:staff_t:s2:c0.c10` = Secret clearance with Finance+HR access
> 

---

## 3. Enabling MLS Policy

### Step-by-Step Setup

```bash
# 1. Install MLS policy package
sudo dnf install selinux-policy-mls -y

# 2. Configure SELinux for MLS
sudo vim /etc/selinux/config

```

```
SELINUX=permissive
SELINUXTYPE=mls

```

```bash
# 3. Trigger full relabel
sudo touch /.autorelabel
echo "-F" >> /.autorelabel  # Force full relabel

# 4. Reboot to apply
sudo reboot

```

> ⚠️ Critical Notes:
> 
> - **Always test in Permissive mode first**
> - **Full relabel takes 10-30 minutes**
> - **Never enable Enforcing MLS without testing**

---

## 4. MLS User Management

### Create MLS Users

```bash
# Create user with default MLS context
sudo useradd -Z user_u jhon
sudo passwd jhon

# Verify context
su - jhon
id -Z  # user_u:staff_r:staff_t:s0

# Root context in MLS
sudo su -
id -Z  # root:sysadm_r:sysadm_t:s0-s15:c0.c1023

```

### User Mappings

```bash
# View SELinux user mappings
semanage login -l

```

---

## 5. MLS Enforcement Behavior

### Key Restrictions

1. **No Read Up**: User at `s0` cannot read `s1` files
2. **No Write Down**: User at `s1` cannot write to `s0` files
3. **Category Isolation**: User with `c0` cannot access `c1` files

### Practical Example

```bash
# After enabling Enforcing MLS
sudo setenforce 1

# Root (s0-s15) can access everything
# Regular user (s0) cannot:
# - Read /etc/shadow (typically s15)
# - Write to system logs (s15)

```

> 💡 Why root is restricted:
> 
> 
> Even root operates within **MLS boundaries**—prevents accidental data leaks between classification levels.
> 

---

## 6. Key Commands Reference

| Task | Command |
| --- | --- |
| Install MLS policy | `dnf install selinux-policy-mls` |
| Check status | `sestatus` |
| View user context | `id -Z` |
| List domains | `seinfo -adomain` |
| Search policy rules | `sesearch --allow` |
| Force relabel | `touch /.autorelabel` |
| Create MLS user | `useradd -Z user_u username` |

---

## 7. Best Practices & Warnings

### ✅ Do

- **Test thoroughly in Permissive mode** before Enforcing
- **Document user clearance levels** before deployment
- **Use MCS for commercial environments** (simpler than MLS)
- **Monitor denials** after enabling MLS:
    
    ```bash
    sudo ausearch -m avc -ts recent
    
    ```
    

### ❌ Don't

- **Enable MLS on production systems** without training
- **Assume root bypasses MLS** (it doesn't!)
- **Skip the relabel step** (causes inconsistent contexts)

### Common Issues

- **"Permission denied" after reboot**:
Normal during relabel—wait 15-30 minutes
- **Users can't log in**:
Verify user mappings: `semanage login -l`
- **Services fail to start**:
Check if service supports MLS (most don't by default)

---

## Summary Workflow

```bash
# 1. Install MLS
dnf install selinux-policy-mls -y

# 2. Configure
vim /etc/selinux/config
# SELINUX=permissive
# SELINUXTYPE=mls

# 3. Relabel
touch /.autorelabel
reboot

# 4. Test
sestatus
useradd -Z user_u testuser
setenforce 1  # Only after thorough testing

```

> 💡 Golden Rule:
> 
> 
> **MLS = Military-grade security**. Always:
> 
> - Start with **Permissive mode**
> - Understand **sensitivity/category model**
> - **Never enable Enforcing** without full testing!
> - Use **MCS for commercial needs** (not full MLS)