# 033: How to Recover the /boot Directory 


## **Scenario Overview**

- The entire `/boot` directory has been accidentally deleted (`rm -rf /boot/*`)
- System fails to boot → drops to **GRUB rescue prompt**
- Recovery requires **CentOS 8 installation media**

> ⚠️ Critical: This procedure only works if:
> 
> - Root filesystem (`/`) is **intact**
> - You have **physical/console access** or **rescue ISO**

---

## **Step-by-Step Recovery Procedure**

### **1. Boot from CentOS 8 Installation Media**

1. Attach CentOS 8 ISO to VM/physical machine
2. Reboot and boot from ISO
3. At installation menu:
    - Select **"Troubleshooting"**
    - Choose **"Rescue a CentOS Linux system"**
    - Select **Option 1: Continue** (mounts system to `/mnt/sysimage`)
    - Press **Enter** to get shell prompt

---

### **2. Reinstall Kernel Packages**

```bash
# Navigate to BaseOS packages
cd /mnt/install/repo/BaseOS/Packages

# Find exact kernel-core version (match your system)
ls kernel-core-*.rpm

# Reinstall kernel-core (replace with your version)
rpm -ivh --root=/mnt/sysimage --replacepkgs kernel-core-4.18.0-80.el8.x86_64.rpm

```

> 💡 Note:
> 
> - Ignore `grub2-editenv` error—it will be fixed later
> - Use `-replacepkgs` to force reinstall over missing files

---

### **3. Rebuild GRUB Bootloader**

```bash
# Chroot into your system
chroot /mnt/sysimage

# Reinstall GRUB to MBR
grub2-install /dev/sda    # Replace sda with your boot disk

# Regenerate GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg

```

> 🔍 Verify disk name:
> 
> 
> Run `lsblk` before `grub2-install` to confirm boot disk (usually `/dev/sda`)
> 

---

### **4. Fix SELinux Contexts**

```bash
# Trigger SELinux relabel on next boot
touch /.autorelabel

```

> ⚠️ Critical Step:
> 
> 
> Without this, you’ll get **login failures** after reboot due to incorrect SELinux labels
> 

---

### **5. Reboot and Verify**

```bash
# Exit chroot and reboot
exit
reboot

```

1. Remove ISO from boot order
2. System should now show **GRUB menu** and boot normally

---

## **Key Commands Reference**

| Task | Command |
| --- | --- |
| Reinstall kernel | `rpm -ivh --root=/mnt/sysimage --replacepkgs kernel-core-*.rpm` |
| Reinstall GRUB | `grub2-install /dev/sda` |
| Regenerate GRUB config | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| Trigger SELinux relabel | `touch /.autorelabel` |
| Verify boot disk | `lsblk` |

---

## **Critical Notes & Best Practices**

### ✅ **Do**

- **Always backup `/boot`** before major updates:
    
    ```bash
    tar -czf /backup/boot-$(date +%F).tar.gz /boot
    
    ```
    
- **Verify kernel version** matches your system:
    
    ```bash
    # From rescue shell (before chroot)
    cat /mnt/sysimage/etc/redhat-release
    ls /mnt/sysimage/lib/modules/
    
    ```
    
- **Use exact package version** from ISO’s `BaseOS/Packages`

### ❌ **Don't**

- Skip `/.autorelabel` (causes post-recovery login issues)
- Assume disk name is `/dev/sda` (verify with `lsblk`)
- Reboot without removing ISO (system may boot from ISO again)

### **Troubleshooting**

- **"No space left on device"**:
Clean up `/tmp` in rescue environment:
    
    ```bash
    rm -rf /tmp/*
    
    ```
    
- **GRUB still not working**:
Check BIOS boot mode (UEFI vs Legacy) matches original setup
- **Kernel version mismatch**:
Use `rpm -qa --root=/mnt/sysimage kernel-core` to find exact version

---

## **Prevention Strategy**

### **Regular Backups**

```bash
# Backup /boot weekly
0 2 * * 0 tar -czf /backup/boot-$(date +\\%F).tar.gz /boot

```

### **Test Recovery Plan**

- Practice this procedure in a **non-production VM**
- Keep **rescue ISO** readily available

### **Monitoring**

- Alert on `/boot` usage:
    
    ```bash
    # Add to monitoring
    df /boot | awk '$5+0 > 80 {print "WARNING: /boot >80% full"}'
    
    ```
    

---

> 💡 Golden Rule:
> 
> 
> **`/.autorelabel` is non-optional** in SELinux systems.
> 
> Without it, your system **will appear broken** after successful GRUB recovery!
>