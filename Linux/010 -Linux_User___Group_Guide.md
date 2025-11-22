# 010: Managing Users and Groups

## 1. Viewing Current User and Session Info

### Explanation

These commands help you identify **who you are**, **who is logged in**, and **your user/group IDs**.

### Commands

```bash
whoami          # Shows current username
who am i        # Shows your login session (terminal, time, etc.)
w               # Lists all logged-in users and their activities
id              # Shows your UID, GID, and groups
id username     # Shows UID/GID/groups for a specific user

```

### Examples

```bash
whoami
# Output: zybi

id zybi
# Output: uid=1001(zybi) gid=1001(zybi) groups=1001(zybi),1002(developer)

```

### Lab Steps

1. Run all four commands:
    
    ```bash
    whoami
    who am i
    w
    id
    
    ```
    

---

## 2. Switching Users with `su`

### Explanation

- `su` (**switch user**) lets you become another user (default: `root`).
- `su -` (or `su -l`) starts a **login shell**—loads the target user’s environment (`$HOME`, `$PATH`, etc.).

### Commands

```bash
su username        # Switch to user (non-login shell)
su - username      # Switch with full login environment
su                 # Switch to root (non-login)
su -               # Switch to root (login shell)
exit               # Return to previous user

```

### Examples

```bash
su - root          # Become root with root's environment
pwd                # Now in /root
exit               # Back to original user

```

### Tips & Warnings

- You need the **target user’s password** (except `root` can switch to anyone without password).
- Always use `su -` (not just `su`) to avoid path/environment issues.

---

## 3. Understanding Primary vs. Secondary Groups

### Explanation

- **Primary group**: Default group for files you create. Defined in `/etc/passwd`.
- **Secondary groups**: Additional groups for extra permissions. Defined in `/etc/group`.

### Key Files

| File | Purpose |
| --- | --- |
| `/etc/passwd` | User accounts (username, UID, primary GID, home dir, shell) |
| `/etc/group` | Group definitions (group name, GID, member list) |

### View Users & Groups

```bash
tail -4 /etc/passwd    # Last 4 user accounts
tail -4 /etc/group     # Last 4 groups

```

---

## 4. Creating Users and Groups

### Create a User

```bash
useradd amit           # Creates user 'amit'
tail -1 /etc/passwd    # Verify: amit:x:1002:1002::/home/amit:/bin/bash
tail -1 /etc/group     # A private group 'amit' is auto-created

```

> ✅ By default, useradd creates a primary group with the same name as the user.
> 

### Create a Group

```bash
groupadd developer     # Creates group 'developer'
tail -2 /etc/group     # Verify new group

```

### Lab Steps

1. Create user and group:
    
    ```bash
    sudo useradd amit
    sudo groupadd developer
    
    ```
    
2. Check files:
    
    ```bash
    grep amit /etc/passwd
    grep developer /etc/group
    
    ```
    

---

## 5. Managing Group Memberships

### Add User to Secondary Group

```bash
usermod -a -G developer zybi    # -a = append, -G = secondary groups
id zybi                         # Verify: now in 'developer' group
groups zybi                     # Alternative view

```

> ⚠️ Never omit -a! usermod -G removes existing secondary groups.
> 

### Change Primary Group

```bash
usermod -g developer zybi       # -g = set primary group
grep zybi /etc/passwd           # Verify GID changed

```

### Rename a Group

```bash
groupmod -n tester developer    # Rename 'developer' → 'tester'
tail -3 /etc/group              # Verify

```

### Delete a Group

```bash
groupdel tester                 # Only works if no user has it as primary group

```

> ❌ Cannot delete a group that is someone’s primary group.
> 

---

## 6. Group Administrators (Delegated Management)

### Explanation

A **group admin** can add/remove members **without root access**.

### Commands

```bash
# Create group and add admin
groupadd unix
gpasswd -A zybi unix        # Make 'zybi' admin of 'unix' group

# Admin adds user (as zybi)
su - zybi
gpasswd -a hamza unix       # Add 'hamza' to 'unix'
tail -3 /etc/group          # Verify

# Remove user from group
gpasswd -d hamza unix       # Remove 'hamza'

# Remove all admins
su -
gpasswd -A "" unix          # Clear admin list

```

### Lab Steps

1. As root:
    
    ```bash
    groupadd unix
    gpasswd -A zybi unix
    
    ```
    
2. Switch to `zybi`:
    
    ```bash
    su - zybi
    gpasswd -a hamza unix
    exit
    
    ```
    
3. Verify:
    
    ```bash
    grep unix /etc/group
    
    ```
    

---

## 7. Temporary Group Switching with `newgrp`

### Explanation

`newgrp` starts a **new shell** with a different **primary group** (useful for file creation).

### Example

```bash
groups                  # Current groups
newgrp developer        # Switch primary group to 'developer'
echo $SHLVL             # Shell level increases (e.g., 2 → 3)
touch testfile
ls -l testfile          # Group = developer
exit                    # Return to original shell

```

### Lab Steps

1. Add yourself to a group:
    
    ```bash
    sudo usermod -a -G developer $USER
    
    ```
    
2. Switch group:
    
    ```bash
    newgrp developer
    touch mydevfile
    ls -l mydevfile    # Group should be 'developer'
    exit
    
    ```
    

---

## Summary Cheat Sheet

| Task | Command |
| --- | --- |
| View current user | `whoami`, `id` |
| Switch to root | `su -` |
| Create user | `sudo useradd username` |
| Create group | `sudo groupadd groupname` |
| Add to secondary group | `sudo usermod -a -G group user` |
| Change primary group | `sudo usermod -g group user` |
| Set group admin | `sudo gpasswd -A user group` |
| Add user to group (as admin) | `gpasswd -a user group` |
| Remove user from group | `gpasswd -d user group` |
| Temporarily switch group | `newgrp groupname` |

> 💡 Golden Rules:
> 
> - Always use `usermod -a -G` (never omit `a`).
> - Use `su -` (not `su`) for proper environment.
> - Primary groups can’t be deleted—change user’s primary group first.