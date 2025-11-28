# 034: Installing a New Kernel in RHEL


## 1. Understanding Kernel Management

### Key Concepts

- **Default Kernel**: RHEL/CentOS ships with a **stable, tested kernel** (e.g., `4.18.0-448.el8.x86_64`)
- **Mainline Kernel (ML)**: Latest upstream kernel from [kernel.org](http://kernel.org/) (via ELRepo)
- **GRUB Management**: Never edit `/boot/grub2/grub.cfg` directly—use configuration tools

> ⚠️ Critical Warning:
> 
> 
> Mainline kernels **lack RHEL-specific patches** and may cause stability issues.
> 
> Only use for testing or specific hardware support.
> 

---

## 2. Installing ELRepo Repository

### Add ELRepo (RHEL 8 Compatible)

```bash
# Install ELRepo GPG key
sudo rpm --import <https://www.elrepo.org/RPM-GPG-KEY-elrepo.org>

# Install ELRepo for RHEL 8 (NOT RHEL 7!)
sudo dnf install <https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm> -y

# Verify repositories
sudo dnf repolist all | grep elrepo

```

> 💡 Note:
> 
> 
> Your original command used **RHEL 7 repo** (`elrepo-release-7.el7...`)—this causes errors on RHEL 8!
> 

---

## 3. Installing Mainline Kernel

### Install Latest Mainline Kernel

```bash
# List available kernels
sudo dnf --enablerepo=elrepo-kernel list available | grep kernel-ml

# Install mainline kernel
sudo dnf --enablerepo=elrepo-kernel install kernel-ml -y

# Verify installation
ls /boot/vmlinuz-*    # Should show new kernel (e.g., vmlinuz-6.1.0-1.el8.elrepo.x86_64)

```

> 🔍 Kernel Naming:
> 
> - `kernel-ml` = Mainline kernel
> - `kernel-lt` = Long-term support kernel

---

## 4. Managing GRUB Boot Entries

### View Current GRUB Configuration

```bash
# Check default boot entry
cat /boot/grub2/grubenv | grep saved_entry

# View GRUB settings
cat /etc/default/grub

```

### Set Default Kernel

```bash
# List boot entries (0 = first entry, 1 = second, etc.)
sudo grep -P "submenu|menuentry" /boot/grub2/grub.cfg | cat -n

# Set default to latest kernel (usually entry 0)
sudo grub2-set-default 0

# Verify
cat /boot/grub2/grubenv

```

### Customize GRUB Timeout

```bash
# Edit GRUB config
sudo vim /etc/default/grub

```

Modify:

```
GRUB_TIMEOUT=30
GRUB_TIMEOUT_STYLE=menu    # Always show menu

```

### Regenerate GRUB Configuration

```bash
# For BIOS systems
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# For UEFI systems
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

```

> ⚠️ Never edit /boot/grub2/grub.cfg directly—changes lost on kernel updates!
> 

---

## 5. Reboot and Verify

### Reboot to New Kernel

```bash
sudo systemctl reboot

```

### Verify Kernel Version

```bash
uname -r    # Should show new mainline version (e.g., 6.1.0-1.el8.elrepo.x86_64)

```

---

## 6. Key Commands Reference

| Task | Command |
| --- | --- |
| Add ELRepo (RHEL 8) | `dnf install <https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm`> |
| Install mainline kernel | `dnf --enablerepo=elrepo-kernel install kernel-ml` |
| List boot entries | `grep -P "submenu |
| Set default kernel | `grub2-set-default N` |
| Regenerate GRUB config | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| Verify kernel | `uname -r` |

---

## 7. Critical Best Practices

### ✅ Do

- **Keep old kernels**: Never remove default RHEL kernel until new one is tested
- **Test in non-production**: Mainline kernels may break drivers/modules
- **Backup GRUB config**:
    
    ```bash
    sudo cp /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak
    
    ```
    

### ❌ Don't

- Use **RHEL 7 repos on RHEL 8** (causes dependency conflicts)
- Edit **`/boot/grub2/grub.cfg`** directly
- Remove **all old kernels** (keep at least 2)

### Rollback Plan

If new kernel fails:

1. Reboot → Select old kernel from GRUB menu
2. Remove problematic kernel:
    
    ```bash
    sudo dnf remove kernel-ml-6.1.0*
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg
    
    ```
    

---

## 8. Troubleshooting

### Issue: "No package kernel-ml available"

**Cause**: Wrong ELRepo version (RHEL 7 vs 8)

**Fix**:

```bash
# Remove wrong repo
sudo rm /etc/yum.repos.d/elrepo*.repo

# Install correct RHEL 8 repo
sudo dnf install <https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm>

```

### Issue: System won't boot new kernel

**Solution**:

1. Reboot → Select working kernel from GRUB menu
2. Check for missing initramfs:
    
    ```bash
    ls /boot/initramfs-$(uname -r).img    # Should exist for new kernel
    
    ```
    
3. Rebuild if missing:
    
    ```bash
    sudo dracut -f /boot/initramfs-6.1.0-1.el8.elrepo.x86_64.img 6.1.0-1.el8.elrepo.x86_64
    
    ```
    

---

## Summary Workflow

```bash
# 1. Add correct ELRepo
rpm --import <https://www.elrepo.org/RPM-GPG-KEY-elrepo.org>
dnf install <https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm>

# 2. Install kernel
dnf --enablerepo=elrepo-kernel install kernel-ml

# 3. Configure GRUB
grub2-set-default 0
vim /etc/default/grub  # Set GRUB_TIMEOUT=30
grub2-mkconfig -o /boot/grub2/grub.cfg

# 4. Reboot and verify
reboot
uname -r

```

> 💡 Golden Rule:
> 
> 
> **Mainline kernels = bleeding edge**. Always:
> 
> - Keep **RHEL default kernel** as fallback
> - Test **thoroughly** before production use
> - Use **ELRepo RHEL 8 repo** (not RHEL 7)!