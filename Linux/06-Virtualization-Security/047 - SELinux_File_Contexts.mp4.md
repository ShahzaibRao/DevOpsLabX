# 047: SELinux Contexts: Labeling and Relabeling

## 1. Understanding SELinux Contexts (Labels)

### Context Components

Every file/process has a **4-part context** (user:role:type:level):

```bash
ls -Z file.txt
# Output: unconfined_u:object_r:user_home_t:s0
#         │           │         │           └── Security Level
#         │           │         └────────────── Type (most important!)
#         │           └──────────────────────── Role
#         └──────────────────────────────────── SELinux User

```

### How Contexts Are Assigned

- **New files** inherit context from **parent directory**
- **System files** get contexts from **policy rules** (`/etc/selinux/targeted/contexts/files/`)
- **User files** get default contexts based on location (`/home` → `user_home_t`, `/tmp` → `user_tmp_t`)

---

## 2. Temporary Context Changes with `chcon`

### Basic Usage

```bash
# Create test file
touch file.txt
ls -Z file.txt  # unconfined_u:object_r:user_home_t:s0

# Change type temporarily
chcon -t httpd_sys_content_t file.txt
ls -Z file.txt  # unconfined_u:object_r:httpd_sys_content_t:s0

# Restore original context
restorecon -v file.txt
ls -Z file.txt  # Back to user_home_t

```

### Recursive Changes

```bash
# Create directory structure
mkdir -p /tmp/testdir
cal > /tmp/testdir/cal.txt

# Change context recursively
chcon -R -t httpd_sys_content_t /tmp/testdir/
ls -Z /tmp/testdir/cal.txt  # Now httpd_sys_content_t

```

> ⚠️ Critical Limitation:
> 
> 
> `chcon` changes are **temporary**—lost on:
> 
> - Reboot
> - `restorecon`
> - Filesystem relabel (`/.autorelabel`)

---

## 3. Permanent Context Changes with `semanage fcontext`

### Add Permanent Policy Rule

```bash
# Create system file
sudo touch /etc/myservice.conf

# Add policy rule (permanent)
sudo semanage fcontext -a -t samba_share_t "/etc/myservice.conf"

# Verify rule added
tail /etc/selinux/targeted/contexts/files/file_contexts.local
# Output: /etc/myservice.conf    system_u:object_r:samba_share_t:s0

# Apply context immediately
sudo restorecon -v /etc/myservice.conf
ls -Z /etc/myservice.conf  # Now samba_share_t

```

### Pattern-Based Rules

```bash
# Create web directory
mkdir ~/web
touch ~/web/file{1..3}

# Add recursive rule (note regex syntax)
semanage fcontext -a -t httpd_sys_content_t "$HOME/web(/.*)?"

# Apply to all files
restorecon -Rv ~/web/
ls -Z ~/web/file1  # httpd_sys_content_t

```

> 💡 Regex Syntax:
> 
> - `(/.*)?` = Optional subdirectories/files
> - Use **double quotes** to prevent shell expansion

---

## 4. Managing Policy Rules

### List Existing Rules

```bash
# List all custom rules
semanage fcontext -l

# Search specific rules
semanage fcontext -l | grep web

```

### Delete Policy Rules

```bash
# Remove directory rule
semanage fcontext -d "$HOME/web(/.*)?"

# Remove file rule
semanage fcontext -d "/etc/myservice.conf"

# Verify removal
tail /etc/selinux/targeted/contexts/files/file_contexts.local

# Restore original contexts
restorecon -Rv ~/web/
restorecon -v /etc/myservice.conf

```

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| View context | `ls -Z file` |
| Temporary change | `chcon -t TYPE file` |
| Recursive temp change | `chcon -R -t TYPE dir/` |
| Add permanent rule | `semanage fcontext -a -t TYPE "/path(/.*)?"` |
| Apply permanent rule | `restorecon -Rv /path` |
| List rules | `semanage fcontext -l` |
| Delete rule | `semanage fcontext -d "/path(/.*)?"` |
| Restore default | `restorecon -v file` |

---

## 6. Best Practices & Troubleshooting

### ✅ Do

- **Use `semanage fcontext` + `restorecon`** for permanent changes
- **Test with `restorecon -v`** to see what will change
- **Use regex patterns** for directories: `"/web(/.*)?"`
- **Check policy files** after adding rules:
    
    ```bash
    grep web /etc/selinux/targeted/contexts/files/file_contexts.local
    
    ```
    

### ❌ Don't

- **Rely on `chcon`** for production systems (changes lost on reboot)
- **Use absolute paths without quotes** (shell expansion breaks regex)
- **Forget `R` flag** for recursive directory changes

### Common Issues

- **"Rule not applying"**:
Check regex syntax—must use `(/.*)?` for directories
- **"Permission denied"**:
Run `semanage` and `restorecon` as **root**
- **"File context not changing"**:
Verify rule exists in policy file before running `restorecon`

### Verification Workflow

```bash
# 1. Add rule
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"

# 2. Verify rule exists
grep web /etc/selinux/targeted/contexts/files/file_contexts.local

# 3. Apply context
restorecon -Rv /web/

# 4. Confirm
ls -Z /web/file1

```

---

## Summary Cheat Sheet

```bash
# Temporary change (lost on reboot)
chcon -t httpd_sys_content_t file

# Permanent change
semanage fcontext -a -t httpd_sys_content_t "/path(/.*)?"
restorecon -Rv /path

# Remove permanent rule
semanage fcontext -d "/path(/.*)?"
restorecon -Rv /path

# View contexts
ls -Z file
ls -Zd directory

```

> 💡 Golden Rule:
> 
> 
> **`chcon` = Temporary, `semanage fcontext` = Permanent**. Always:
> 
> - Use **regex patterns** for directories
> - **Verify rules** before applying
> - **Restore contexts** after policy changes!