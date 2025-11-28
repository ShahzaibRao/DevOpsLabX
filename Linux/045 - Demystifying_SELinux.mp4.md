# 045: SELinux Contexts & Process Domain Transitions

## 1. Understanding SELinux Context Components

Every SELinux context has **4 components** (user:role:type:level):

| Component | Purpose | Example |
| --- | --- | --- |
| **SELinux User** | Maps Linux users to SELinux roles | `user_u`, `root`, `system_u` |
| **Role** | Defines which domains a user can enter | `object_r` (files), `system_r` (processes) |
| **Type** | **Most critical**—defines access permissions | `user_home_t`, `httpd_t`, `shadow_t` |
| **Level** | MLS/MCS security level (sensitivity:category) | `s0`, `s0-s15:c0.c1023` |

### Viewing Contexts

```bash
# File context
ls -Z file.txt

# Directory context (note -d flag)
ls -Zd /tmp

# Process context
ps -eZ

```

---

## 2. Deep Dive: SELinux Components

### SELinux User (`semanage login -l`)

- Maps **Linux users** → **SELinux users**
- Controls which roles a user can assume

```bash
# View user mappings
semanage login -l

# Example output:
# __default__               user_u
# root                      root
# zybi                      user_u

```

> 💡 Key Insight:
> 
> 
> `unconfined_u` = Can access almost anything (like traditional Linux)
> 
> `user_u` = Restricted user (common for regular accounts)
> 

---

### SELinux Role

- **Files**: Always `object_r` (no role-based access for files)
- **Processes**: `system_r` (system processes), `user_r` (user processes)
- **Role-Based Access Control (RBAC)**:
    - User → Role → Domain → Process
    - Prevents privilege escalation even if process is compromised

> 🔍 Example:
> 
> 
> Web server runs in `httpd_t` domain → Can only access `httpd_sys_content_t` files
> 
> Even if hacked, can't read `/etc/shadow` (`shadow_t` type)
> 

---

### SELinux Type (Most Important!)

- **Type Enforcement (TE)**: Core of SELinux policy
- Defines **what domains can access what types**

```bash
# Critical system files
ls -Z /etc/shadow        # -rw------- root root system_u:object_r:shadow_t:s0
ls -Z /usr/bin/passwd    # -rwsr-xr-x root root system_u:object_r:passwd_exec_t:s0

```

> ⚠️ Why this matters:
> 
> 
> `passwd` process must transition to `passwd_t` domain to access `shadow_t` files
> 

---

### SELinux Level (MLS/MCS)

- **MLS (Multi-Level Security)**: Government/military (Top Secret, Secret, etc.)
- **MCS (Multi-Category Security)**: Commercial (categories like Finance, HR)
- Format: `sensitivity:category` (e.g., `s0:c0.c10`)

> 💡 RHEL Default: targeted policy → Levels usually s0 (ignored)
> 

---

## 3. Domain Transitions: How Processes Change Contexts

### What is a Domain Transition?

- When a process **executes a program**, it may transition to a new domain
- Controlled by **type_transition rules** in policy

### Example: `passwd` Command

1. User runs `passwd` (in `user_t` domain)
2. `passwd` binary has type `passwd_exec_t`
3. Policy rule:`domain_auto_trans(user_t, passwd_exec_t, passwd_t)`
4. Process transitions to `passwd_t` domain
5. `passwd_t` can access `shadow_t` files

### Observe Domain Transition

```bash
# Terminal 1: Monitor processes
ps -eZ | grep passwd

# Terminal 2: Change password
passwd

# Terminal 1: See new process
# system_u:system_r:passwd_t:s0

```

> 🔍 Key Observation:
> 
> 
> Process context changes from `user_t` → `passwd_t` during execution
> 

---

## 4. Key Commands Reference

| Task | Command |
| --- | --- |
| View file context | `ls -Z filename` |
| View dir context | `ls -Zd directory` |
| View process context | `ps -eZ` |
| View user mappings | `semanage login -l` |
| Check SELinux status | `sestatus` |
| View current context | `id -Z` |
| Monitor transitions | `ps -eZ \| grep process` |

---

## 5. Practical Examples

### File Context Example

```bash
# Create file in /tmp
echo "test" > /tmp/testfile
ls -Z /tmp/testfile    # user_u:object_r:user_tmp_t:s0

# /tmp context
ls -Zd /tmp            # system_u:object_r:tmp_t:s0

```

### Process Context Example

```bash
# SSH process
ps -eZ | grep sshd     # system_u:system_r:sshd_t:s0

# Web server (if installed)
ps -eZ | grep httpd    # system_u:system_r:httpd_t:s0

```

---

## 6. Best Practices & Troubleshooting

### ✅ Do

- **Understand type enforcement**: 90% of SELinux is about types
- **Use `sealert`** for denial explanations:
    
    ```bash
    sudo sealert -a /var/log/audit/audit.log
    
    ```
    
- **Test transitions** with `ps -eZ` during command execution

### ❌ Don't

- **Ignore domain transitions**—they're why SELinux works
- **Assume all files in /tmp have same context** (they don't!)
- **Modify contexts without understanding policy**

### Common Issues

- **"Permission denied" despite 755 permissions**:
Check file type vs process domain:
    
    ```bash
    ls -Z /path/to/file
    ps -eZ | grep process_name
    
    ```
    
- **Web server can't read files**:
Files need `httpd_sys_content_t` type:
    
    ```bash
    sudo semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"
    sudo restorecon -R /web
    
    ```
    

---

## Summary Cheat Sheet

```bash
# Context components
user_u:object_r:user_home_t:s0
│       │         │          └── Level (sensitivity:category)
│       │         └───────────── Type (most important!)
│       └─────────────────────── Role
└─────────────────────────────── SELinux User

# Key commands
ls -Z file          # File context
ps -eZ              # Process contexts
semanage login -l   # User mappings
id -Z               # Current shell context

```

> 💡 Golden Rule:
> 
> 
> **Type Enforcement is SELinux**. Always:
> 
> - Check **file types** (`ls -Z`)
> - Check **process domains** (`ps -eZ`)
> - Understand **domain transitions** for executables!