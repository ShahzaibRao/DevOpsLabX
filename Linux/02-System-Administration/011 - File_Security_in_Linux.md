# 011: File Security in Linux: Standard Permissions, Special Bits & ACLs

## 1. Understanding `umask` (Default Permissions)

### Explanation

`umask` defines the **default permissions** for newly created files and directories by **masking** (subtracting) permissions from the maximum.

- **Files**: Max `666` (rw-rw-rw-)
- **Directories**: Max `777` (rwxrwxrwx)

### Common `umask` Values

| umask | File Permissions | Dir Permissions | Use Case |
| --- | --- | --- | --- |
| `0022` | `644` (rw-r--r--) | `755` (rwxr-xr-x) | Default for most systems |
| `0077` | `600` (rw-------) | `700` (rwx------) | Private files (no group/others access) |

### Commands

```bash
umask -S                # Show symbolic form: u=rwx,g=rx,o=rx
umask 0077              # Set restrictive default
umask 0022              # Set standard default

```

### Lab Steps

1. Check current `umask`:
    
    ```bash
    umask -S
    
    ```
    
2. Create a file and check permissions:
    
    ```bash
    umask 0077
    touch private.txt
    ls -l private.txt    # Should be -rw-------
    
    ```
    

### Tips & Warnings

- `umask` is **per-session**. Set it in `~/.bashrc` for persistence.
- Never set `umask 0000`—it gives full access to everyone!

---

## 2. Special Permission Bits

### Sticky Bit (`+t`)

**Purpose**: On directories, only the **file owner** (or root) can delete files—even if others have write permission.

### Lab: Sticky Bit

```bash
# Create shared directory
sudo mkdir /test
sudo chmod 777 /test

# User zybi creates a file
su - zybi
cal > /test/cal.txt
exit

# User hamza can delete it (no sticky bit)
su - hamza
rm /test/cal.txt    # SUCCESS (problem!)
exit

# Add sticky bit
sudo chmod +t /test
ls -ld /test        # Should show '...rwxrwxrwt'

# Now hamza CANNOT delete zybi's file
su - hamza
rm /test/cal.txt    # Permission denied!

```

> ✅ Common use: /tmp (always has sticky bit).
> 

---

### SetGID Bit (`g+s`)

**Purpose**: On directories, new files inherit the **directory’s group** (not the user’s primary group).

### Lab: SetGID

```bash
# Create project directory
sudo groupadd proj55
sudo mkdir /tmp/proj55
sudo chown root:proj55 /tmp/proj55
sudo chmod 777 /tmp/proj55

# User zybi creates a file (without SetGID)
su - zybi
cal > /tmp/proj55/cal.txt
ls -l /tmp/proj55/cal.txt  # Group = zybi's primary group

# Add SetGID
sudo chmod 2777 /tmp/proj55  # '2' = SetGID
ls -ld /tmp/proj55           # Should show 'drwxrwsrwx'

# New file inherits directory's group
su - zybi
cal > /tmp/proj55/cal22.txt
ls -l /tmp/proj55/cal22.txt  # Group = proj55

```

### Manage SetGID

```bash
# Add
chmod g+s /path/to/dir

# Remove
chmod g-s /path/to/dir

# Find all SetGID directories
find / -type d -perm -2000 2>/dev/null

```

---

### SetUID Bit (`u+s`)

**Purpose**: On **executables**, runs with the **owner’s privileges** (not the user’s).

⚠️ **Rarely used on files**—security risk!

### Example

```bash
# Create script (not recommended for real use!)
touch /tmp/cal.sh
chmod u+s /tmp/cal.sh    # Set SetUID
ls -l /tmp/cal.sh        # Shows 'rwsr-xr-x'

# Remove SetUID
chmod u-s /tmp/cal.sh

```

> ❌ Never set SetUID on scripts—only on trusted binaries (e.g., passwd).
> 

---

## 3. Access Control Lists (ACLs)

### Explanation

ACLs allow **fine-grained permissions** beyond standard user/group/other (e.g., grant access to specific users/groups).

### View ACLs

```bash
getfacl filename    # Show detailed permissions
ls -l filename      # Shows '+' if ACL exists: -rw-rw-r--+

```

### Lab: Manage ACLs

```bash
# Create file
cal > cal.txt

# Grant read to user 'zybi'
setfacl -m u:zybi:4 cal.txt
getfacl cal.txt

# Grant full access to group 'unix'
setfacl -m g:unix:7 cal.txt

# Test access
su - zybi
cat cal.txt    # Works (read-only)
vi cal.txt     # Fails (no write permission)
exit

# Remove user ACL
setfacl -x u:zybi cal.txt

# Remove ALL ACLs
setfacl -b cal.txt

```

### Key ACL Commands

| Task | Command |
| --- | --- |
| Add user ACL | `setfacl -m u:username:perms file` |
| Add group ACL | `setfacl -m g:groupname:perms file` |
| Remove user ACL | `setfacl -x u:username file` |
| Remove all ACLs | `setfacl -b file` |
| View ACLs | `getfacl file` |

### Permissions in ACLs

- `4` = read (`r`)
- `6` = read + write (`rw`)
- `7` = read + write + execute (`rwx`)

---

## Summary Cheat Sheet

### Special Bits

| Bit | Symbol | Command | Purpose |
| --- | --- | --- | --- |
| Sticky | `t` | `chmod +t dir` | Only owner can delete files in dir |
| SetGID | `s` (group) | `chmod g+s dir` | New files inherit dir's group |
| SetUID | `s` (user) | `chmod u+s file` | Run file as owner (use cautiously!) |

### ACLs

| Command | Purpose |
| --- | --- |
| `getfacl file` | View ACLs |
| `setfacl -m u:user:7 file` | Grant rwx to user |
| `setfacl -x u:user file` | Remove user ACL |
| `setfacl -b file` | Remove all ACLs |

### Critical Checks

```bash
ls -ld /shared_dir    # Look for 't' (sticky) or 's' (SetGID)
ls -l file            # Look for '+' (ACLs exist)

```

> 💡 Golden Rules:
> 
> - Use **sticky bit** on shared directories (`/tmp`, project folders).
> - Use **SetGID** on collaborative project directories.
> - Use **ACLs** for exceptions (e.g., "user X needs access to Y’s file").
> - **Avoid SetUID** on scripts—huge security risk!