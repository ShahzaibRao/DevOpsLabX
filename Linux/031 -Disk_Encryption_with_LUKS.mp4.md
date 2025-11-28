# 031: Disk Encryption Using LUKS in Linux

## 1. Understanding LUKS (Linux Unified Key Setup)

### Key Concepts

- **Full-disk encryption**: Encrypts entire block device (not just files).
- **Multiple passphrases**: Up to 8 keys can unlock the same volume.
- **Key files**: Passwordless unlock using secure key files.
- **Use Cases**:
    - Laptops with sensitive data
    - External drives
    - Compliance requirements (HIPAA, GDPR)

> ⚠️ Critical Warning:
> 
> 
> **Lost passphrase = permanently lost data!** Always backup keys.
> 

---

## 2. Installing LUKS Tools

### Prerequisites

```bash
# Enable repositories
sudo dnf repolist all

# Install cryptsetup (RHEL/CentOS 8+)
sudo dnf install cryptsetup -y

# Verify
cryptsetup --version

```

> 💡 Note:
> 
> 
> Package name is **`cryptsetup`** (not `crypsetup`).
> 

---

## 3. Creating LUKS Encrypted Volume

### Encrypt Raw Disk

```bash
# WARNING: This erases ALL data on /dev/sdb!
sudo cryptsetup luksFormat /dev/sdb

# Confirm with "YES" (uppercase)
# Enter passphrase twice (min 8 chars)

```

> ⚠️ Critical:
> 
> - Use **dedicated disk/partition** (not system disk)
> - Passphrase is **case-sensitive**

### Open Encrypted Volume

```bash
# Map to /dev/mapper/myfiles
sudo cryptsetup luksOpen /dev/sdb myfiles

# Verify
sudo cryptsetup status myfiles
ls -l /dev/mapper/myfiles    # Should show block device

```

---

## 4. Using Encrypted Volume

### Format and Mount

```bash
# Create filesystem
sudo mkfs.ext4 /dev/mapper/myfiles

# Mount
sudo mkdir /myfiles
sudo mount /dev/mapper/myfiles /myfiles
df -hT    # Verify mounted as ext4

```

### Test Data

```bash
echo "cal" | sudo tee /myfiles/cal
echo "date" | sudo tee /myfiles/date
sudo chmod 777 /myfiles    # Allow user access

```

---

## 5. Setting Up Key File (Passwordless Unlock)

### Create Secure Key File

```bash
# Generate random 4KB key file
sudo dd if=/dev/urandom of=/etc/crypt_file bs=4096 count=1

# Set strict permissions
sudo chmod 600 /etc/crypt_file
sudo chown root:root /etc/crypt_file

```

### Add Key File to LUKS

```bash
# Add key file as second unlock method
sudo cryptsetup luksAddKey /dev/sdb /etc/crypt_file

# Verify (should show 2 key slots)
sudo cryptsetup luksDump /dev/sdb

```

### Configure Auto-Unlock

```bash
# Edit /etc/crypttab
echo "myfiles /dev/sdb /etc/crypt_file" | sudo tee /etc/crypttab

# Edit /etc/fstab
echo "/dev/mapper/myfiles /myfiles ext4 defaults 0 0" | sudo tee -a /etc/fstab

# Test
sudo mount -a
df -hT    # Should show /myfiles mounted

```

> 💡 File Paths:
> 
> - `/etc/crypttab`: `name device keyfile`
> - `/etc/fstab`: Use **`/dev/mapper/name`**

---

## 6. Managing LUKS Volumes

### Close Volume

```bash
# Unmount and close
sudo umount /myfiles
sudo cryptsetup luksClose myfiles

```

### Reopen Volume

```bash
# With passphrase
sudo cryptsetup luksOpen /dev/sdb myfiles

# Or with key file
sudo cryptsetup open --key-file /etc/crypt_file /dev/sdb myfiles

```

### Change Passphrase

```bash
# Update existing passphrase
sudo cryptsetup luksChangeKey /dev/sdb

# Enter old passphrase → new passphrase (twice)

```

---

## 7. Removing Encryption

### Steps to Clean Up

```bash
# 1. Unmount
sudo umount /myfiles

# 2. Remove from fstab
sudo sed -i '/myfiles/d' /etc/fstab

# 3. Close volume
sudo cryptsetup luksClose myfiles

# 4. Remove key file (optional)
sudo rm /etc/crypt_file

# 5. Wipe LUKS header (permanent!)
sudo cryptsetup erase /dev/sdb

```

> ⚠️ Final Warning:
> 
> 
> `cryptsetup erase` **permanently destroys** all data—no recovery possible!
> 

---

## 8. Key Commands Reference

| Task | Command |
| --- | --- |
| Encrypt disk | `cryptsetup luksFormat /dev/sdX` |
| Open volume | `cryptsetup luksOpen /dev/sdX name` |
| Close volume | `cryptsetup luksClose name` |
| Add key file | `cryptsetup luksAddKey /dev/sdX /path/keyfile` |
| Change passphrase | `cryptsetup luksChangeKey /dev/sdX` |
| View slots | `cryptsetup luksDump /dev/sdX` |
| Wipe header | `cryptsetup erase /dev/sdX` |

---

## 9. Critical Best Practices

### ✅ Do

- **Backup key files** to secure offline location
- Use **strong passphrases** (12+ chars, mixed case/numbers/symbols)
- Test **recovery procedures** before deploying
- Set **strict permissions** on key files (`600`)

### ❌ Don't

- Encrypt **system/boot partitions** without proper initramfs config
- Store key files on the **same system** as encrypted data (defeats purpose)
- Forget to **update initramfs** for root encryption:
    
    ```bash
    sudo dracut -f    # RHEL/CentOS
    
    ```
    

### Security Hardening

- **Use TPM** for key storage (advanced)
- **Enable FDE (Full Disk Encryption)** during OS install for system disks
- **Shred key files** when no longer needed:
    
    ```bash
    sudo shred -u /etc/crypt_file
    
    ```
    

---

## Summary Workflow

```bash
# 1. Encrypt
cryptsetup luksFormat /dev/sdb → luksOpen → mkfs → mount

# 2. Auto-unlock
dd keyfile → chmod 600 → luksAddKey → crypttab → fstab

# 3. Manage
luksChangeKey → mount -a → test

# 4. Remove
umount → luksClose → erase

```

> 💡 Golden Rule:
> 
> 
> **LUKS = Data Security**. Always:
> 
> - Treat passphrases like **master keys**
> - Backup keys **offline**
> - Test recovery **before trusting**!