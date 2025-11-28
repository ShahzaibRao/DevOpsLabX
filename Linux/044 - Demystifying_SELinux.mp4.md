# 044: Introduction to Security-Enhanced Linux

## 1. What is SELinux?

### Core Concept

- **Mandatory Access Control (MAC)**: Enforces system-wide security policies (unlike traditional DAC)
- **Kernel-level security**: Built into Linux kernel since 2.6
- **Policy-driven**: Defines exactly what processes/users can access

### How SELinux Enhances Security

| Traditional Linux (DAC) | SELinux (MAC) |
| --- | --- |
| Owner decides permissions | **System policy** decides permissions |
| "Root can do anything" | **Even root restricted** by policy |
| File permissions only | **Context-based** (user:role:type:level) |

> 💡 Key Insight:
> 
> 
> SELinux prevents **privilege escalation**—even if an attacker gains root access, they're confined by policy.
> 

---

## 2. SELinux Modes

### Three Operational Modes

| Mode | Behavior | Use Case |
| --- | --- | --- |
| **Enforcing** | **Blocks** unauthorized access + logs denials | Production systems |
| **Permissive** | **Logs** denials but **allows** access | Troubleshooting |
| **Disabled** | **SELinux completely off** | Not recommended |

### Check and Change Modes

```bash
# Check current mode
getenforce

# Temporarily set to Permissive
sudo setenforce 0

# Temporarily set to Enforcing
sudo setenforce 1

# Permanent configuration
cat /etc/selinux/config

```

> ⚠️ Critical Note:
> 
> 
> **Never disable SELinux**—use Permissive mode for debugging instead.
> 

---

## 3. SELinux Contexts

### File Contexts (`ls -Z`)

```bash
# Create test file
cal > /tmp/cal.txt

# View SELinux context
ls -lZ /tmp/cal.txt
# Output: -rw-r--r--. user_u object_r user_home_t:s0 cal.txt

```

- **user_u**: SELinux user
- **object_r**: Role
- **user_home_t**: **Type** (most important for access control)
- **s0**: Sensitivity level (MLS/MCS)

### Process Contexts (`ps -Z`)

```bash
# View SSH process context
ps -efZ | grep sshd
# Output: system_u:system_r:sshd_t:s0 ...

```

- **sshd_t**: Process type (must match file types for access)

---

## 4. Practical Example: Access Control

### Scenario: Restrict File Access

```bash
# As root
echo "secret" > /tmp/secret.txt
chmod 600 /tmp/secret.txt

# As user zybi
su - zybi
cat /tmp/secret.txt    # Fails (traditional DAC)

# But what if permissions were 644?
chmod 644 /tmp/secret.txt
cat /tmp/secret.txt    # Still might fail if SELinux context blocks it!

```

### SELinux in Action

- Even with `644` permissions, access depends on:
    - **Process type** (e.g., `user_t` vs `httpd_t`)
    - **File type** (e.g., `user_home_t` vs `httpd_sys_content_t`)

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Check mode | `getenforce` |
| Set mode (temp) | `sudo setenforce 0` (Permissive) / `1` (Enforcing) |
| View file context | `ls -lZ filename` |
| View process context | `ps -eZ` or `ps -Z -p PID` |
| Check config | `cat /etc/selinux/config` |
| View denials | `sudo ausearch -m avc -ts recent` |

---

## 6. SELinux Configuration File

### `/etc/selinux/config`

```
# This file controls the state of SELinux on the system.
SELINUX=enforcing    # enforcing | permissive | disabled
SELINUXTYPE=targeted # targeted | minimum | mls

```

### Policy Types

| Type | Description |
| --- | --- |
| **targeted** | **Default**—only targeted processes confined (e.g., httpd, sshd) |
| **minimum** | Basic confinement for selected processes |
| **mls** | Multi-Level Security (government/military) |

> 💡 Recommendation:
> 
> 
> Always use **`SELINUX=enforcing`** and **`SELINUXTYPE=targeted`** in production.
> 

---

## 7. Best Practices & Troubleshooting

### ✅ Do

- **Keep SELinux Enforcing** in production
- **Use Permissive mode** for debugging (not Disabled)
- **Check contexts** when access is denied: `ls -lZ`
- **Review denials**: `sudo ausearch -m avc`

### ❌ Don't

- **Disable SELinux** ("I'll fix it later" → never happens)
- **Assume chmod fixes everything** (SELinux contexts matter more)
- **Ignore audit logs** (`/var/log/audit/audit.log`)

### Common Workflow for Access Issues

1. **Reproduce issue** in Enforcing mode
2. **Check denials**:
    
    ```bash
    sudo ausearch -m avc -ts recent
    
    ```
    
3. **Temporarily set Permissive**:
    
    ```bash
    sudo setenforce 0
    
    ```
    
4. **Test if issue resolves** → confirms SELinux cause
5. **Fix properly** (don't just disable SELinux!)

---

## Summary Cheat Sheet

```bash
# Check status
getenforce
ls -lZ file
ps -eZ

# Temporary mode change
setenforce 0    # Permissive
setenforce 1    # Enforcing

# Permanent config
vi /etc/selinux/config
# SELINUX=enforcing
# SELINUXTYPE=targeted

```

> 💡 Golden Rule:
> 
> 
> **SELinux is your friend**—not your enemy!
> 
> Always:
> 
> - Keep it **Enforcing**
> - Use **Permissive for debugging**
> - Fix **contexts** (not disable security)!