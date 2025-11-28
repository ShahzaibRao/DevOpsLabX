# 046: SELinux Targeted Policy: Confined vs. Unconfined Processes

## 1. Understanding Targeted Policy

### Core Concept

- **Default policy** in RHEL/Fedora
- **Two domains**:
    - **Confined**: Restricted processes (network-facing services)
    - **Unconfined**: Traditional Linux behavior (users, most system processes)

### How It Works

| Process Type | Domain | Access Control |
| --- | --- | --- |
| **Network services** (httpd, sshd) | Confined (`httpd_t`, `sshd_t`) | Strict type enforcement |
| **Users & most system processes** | Unconfined (`unconfined_t`) | Only traditional DAC (chmod) |

> 💡 Key Insight:
> 
> 
> Targeted policy **only confines high-risk processes**—everything else works like traditional Linux.
> 

---

## 2. Confined Processes: Web Server Example

### Setup & Breakage

```bash
# Install and start Apache
sudo dnf install httpd -y
sudo systemctl enable --now httpd

# Create test page
echo "Hello" | sudo tee /var/www/html/index.html

# Allow through firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

# Test (works)
curl localhost

```

### Break with Wrong Context

```bash
# Change file context to Samba type
sudo chcon -t samba_share_t /var/www/html/index.html
ls -Z /var/www/html/index.html  # Now samba_share_t

# Restart Apache (fails to serve)
sudo systemctl restart httpd
curl localhost  # Permission denied!

```

### Diagnose & Fix

```bash
# Check denials
sudo tail /var/log/audit/audit.log | grep avc

# Temporarily disable SELinux (confirm issue)
sudo setenforce 0
curl localhost  # Now works!
sudo setenforce 1

# Proper fix: Restore correct context
sudo restorecon -v /var/www/html/index.html
curl localhost  # Works with SELinux enforcing!

```

> 🔍 Why this happens:
> 
> 
> `httpd_t` domain can only read `httpd_sys_content_t` files—not `samba_share_t`.
> 

---

## 3. Unconfined Processes

### Characteristics

- **Users**: Run in `unconfined_t` domain
- **System processes**: Most run unconfined unless specifically targeted
- **Behavior**: Only restricted by traditional file permissions (`chmod`)

### Process Context Examples

```bash
# User process
ps -Z -p $$  # unconfined_u:unconfined_r:unconfined_t:s0

# Kernel process
ps -Z -C kthreadd  # system_u:system_r:kernel_t:s0

# Unconfined service
ps -Z -C crond  # system_u:system_r:unconfined_service_t:s0

```

---

## 4. Managing Confined Users

### View Current User Mappings

```bash
# List SELinux user mappings
semanage login -l

# List all SELinux users
seinfo -u

```

### Create Confined Users

```bash
# Install setools for seinfo
sudo dnf install setools-console -y

# Create user with confined SELinux user
sudo useradd -Z staff_u example.user1
sudo passwd example.user1

# Verify
ssh example.user1@localhost
id -Z  # staff_u:staff_r:staff_t:s0

```

### Change Default User Mapping

```bash
# Make ALL new users confined
sudo semanage login -m -s user_u -r s0 __default__

# Create new user (now confined)
sudo useradd example.user2
ssh example.user2@localhost
id -Z  # user_u:user_r:user_t:s0

```

---

## 5. Admin Users with Confined Access

### Create Confined Admin

```bash
# Enable SSH for sysadm_u
sudo setsebool -P ssh_sysadm_login on

# Create admin user
sudo useradd -G wheel -Z sysadm_u example.user3
sudo passwd example.user3

# Verify
ssh example.user3@localhost
id -Z  # sysadm_u:sysadm_r:sysadm_t:s0

# Switch to root (stays confined)
sudo su -
id -Z  # sysadm_u:sysadm_r:sysadm_t:s0

```

### Revert Changes

```bash
# Disable SSH for sysadm_u
sudo setsebool -P ssh_sysadm_login off

# Change user back to unconfined
sudo semanage login -m -s unconfined_u -r s0 example.user3

```

---

## 6. SELinux Booleans: Runtime Policy Tuning

### What Are Booleans?

- **On/Off switches** for SELinux policy rules
- Allow customization **without writing policy**

### Manage Booleans

```bash
# List all booleans
getsebool -a
semanage boolean -l

# Check specific boolean
getsebool httpd_can_network_connect_db

# Temporary change (lost on reboot)
setsebool httpd_can_network_connect_db on

# Permanent change
setsebool -P httpd_can_network_connect_db on

```

### Common Booleans

| Boolean | Purpose |
| --- | --- |
| `httpd_can_network_connect` | Allow Apache to connect to network |
| `ftp_home_dir` | Allow FTP access to user home dirs |
| `ssh_sysadm_login` | Allow sysadm_u users to SSH |

---

## 7. SELinux Modes & Configuration

### Temporary Mode Changes

```bash
getenforce        # Check current mode
sudo setenforce 0 # Permissive mode
sudo setenforce 1 # Enforcing mode

```

### Permanent Configuration

```bash
# Edit main config file
sudo vim /etc/selinux/config

```

```
SELINUX=enforcing    # enforcing | permissive | disabled
SELINUXTYPE=targeted

```

> ⚠️ Critical Warning:
> 
> 
> **Never set `SELINUX=disabled`**—use `permissive` for debugging instead.
> 

---

## 8. Key Commands Reference

| Task | Command |
| --- | --- |
| View user mappings | `semanage login -l` |
| List SELinux users | `seinfo -u` |
| Create confined user | `useradd -Z staff_u username` |
| Change default mapping | `semanage login -m -s user_u __default__` |
| List booleans | `getsebool -a` |
| Set boolean (permanent) | `setsebool -P boolean_name on` |
| Restore file context | `restorecon -v /path` |
| Check denials | `sudo ausearch -m avc -ts recent` |

---

## 9. Best Practices & Troubleshooting

### ✅ Do

- **Use `restorecon`** instead of `chcon` for permanent fixes
- **Enable booleans** instead of disabling SELinux
- **Test in Permissive mode** before making changes
- **Use confined users** for shared systems

### ❌ Don't

- **Disable SELinux** (`SELINUX=disabled`)
- **Use `chcon` permanently** (lost on relabel)
- **Ignore audit logs** (`/var/log/audit/audit.log`)

### Troubleshooting Workflow

1. **Reproduce issue** in Enforcing mode
2. **Check denials**:
    
    ```bash
    sudo ausearch -m avc -ts recent
    
    ```
    
3. **Use `sealert`** for solutions:
    
    ```bash
    sudo sealert -a /var/log/audit/audit.log
    
    ```
    
4. **Fix properly**:
    - Restore contexts: `restorecon`
    - Enable booleans: `setsebool -P`
    - Create custom policies (last resort)

---

## Summary Cheat Sheet

```bash
# Confined vs Unconfined
httpd_t → confined (restricted)
unconfined_t → unconfined (traditional)

# Fix file contexts
restorecon -v /var/www/html/index.html

# Manage users
useradd -Z staff_u username
semanage login -m -s user_u __default__

# Booleans
setsebool -P httpd_can_network_connect on

# Modes
setenforce 0    # Permissive (debugging)
# /etc/selinux/config → SELINUX=enforcing (production)

```

> 💡 Golden Rule:
> 
> 
> **SELinux targeted policy = Confine the risky, free the rest**. Always:
> 
> - Keep **Enforcing** mode
> - Use **booleans** for customization
> - **Restore contexts** (don't disable security)!