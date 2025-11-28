# 043: KVM: Kernel-Based Virtual Machine Setup 

## 1. Understanding KVM

### What is KVM?

- **Type-1 Hypervisor**: Runs directly on hardware (via Linux kernel)
- **Open Source**: Integrated into Linux kernel since 2.6.20
- **Enterprise Ready**: Used by AWS, Google Cloud, and OpenStack
- **Hardware Accelerated**: Requires CPU virtualization extensions

### Hardware Requirements

```bash
# Check Intel VT-x support
grep -E 'vmx' /proc/cpuinfo

# Check AMD-V support
grep -E 'svm' /proc/cpuinfo

```

> ✅ Output: If lines appear → CPU supports virtualization
> 
> 
> ❌ **No output**: Enable VT-x/AMD-V in BIOS
> 

---

## 2. Installing KVM Components

### Install Virtualization Packages

```bash
# Install Cockpit (for web management)
sudo dnf install -y cockpit cockpit-machines

# Enable Cockpit
sudo systemctl enable --now cockpit.socket
sudo systemctl status cockpit.socket

# Configure firewall for Cockpit
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --reload

# Install KVM virtualization stack
sudo dnf module install -y virt
sudo dnf install -y virt-install virt-viewer qemu-kvm libvirt

# Verify hardware support
sudo virt-host-validate

# Enable libvirtd service
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd

```

> ⚠️ Critical Notes:
> 
> - **`cockpit-machines`** = Cockpit plugin for VM management
> - **`virt-host-validate`** = Checks all KVM requirements
> - **`libvirtd`** = Core virtualization daemon

---

## 3. Configuring Network Bridge (for VMs)

### Create Bridge via Cockpit

1. Access Cockpit: `https://<server-ip>:9090`
2. Go to **Networking** → **Add Bridge**
3. Configure:
    - **Bridge name**: `br0`
    - **Ports**: Select physical interface (e.g., `ens192`)
    - **STP**: Disabled (for lab environments)
4. Click **Apply**

> 💡 Why Bridge?:
> 
> - VMs get **direct network access** (like physical machines)
> - Required for **public IP assignment** to VMs

### Verify Bridge

```bash
# Check bridge interface
ip a show br0

# List network interfaces
nmcli con show

```

---

## 4. Preparing VM Installation Media

### Upload ISO Image

```bash
# Copy ISO to libvirt directory
sudo cp ubuntu-20.04-desktop-amd64.iso /var/lib/libvirt/images/ubuntu.iso

# Set proper permissions
sudo chmod 664 /var/lib/libvirt/images/ubuntu.iso
sudo chown root:root /var/lib/libvirt/images/ubuntu.iso

```

> 🔍 ISO Location:
> 
> - **Default storage pool**: `/var/lib/libvirt/images/`
> - Permissions: `664` (readable by libvirt)

---

## 5. Creating Virtual Machines via Cockpit

### Step-by-Step VM Creation

1. In Cockpit → **Virtual Machines** → **Create VM**
2. Configure VM:
    - **Name**: `ubuntu-vm`
    - **Installation type**: Local install media
    - **OS**: Browse to `/var/lib/libvirt/images/ubuntu.iso`
    - **OS version**: Ubuntu 20.04
    - **Storage**: 20 GB (default)
    - **Memory**: 2048 MB
    - **CPUs**: 2
3. **Network**: Select `br0` (bridge interface)
4. Click **Create**

### Post-Creation Steps

- **Console**: Click **Console** to interact with VM installer
- **Start VM**: VM starts automatically after creation
- **Access**: Use VM's IP (visible in Cockpit) for SSH

---

## 6. Command-Line VM Management (Alternative)

### Create VM via CLI

```bash
# Create VM with virt-install
sudo virt-install \\
  --name ubuntu-vm \\
  --ram 2048 \\
  --vcpus 2 \\
  --disk path=/var/lib/libvirt/images/ubuntu-vm.qcow2,size=20 \\
  --os-variant ubuntu20.04 \\
  --network bridge=br0 \\
  --graphics vnc \\
  --cdrom /var/lib/libvirt/images/ubuntu.iso

```

### Manage VMs via virsh

```bash
# List VMs
sudo virsh list --all

# Start/stop VM
sudo virsh start ubuntu-vm
sudo virsh shutdown ubuntu-vm

# Delete VM (with storage)
sudo virsh undefine ubuntu-vm --remove-all-storage

```

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Check CPU support | `grep -E 'vmx |
| Install KVM | `sudo dnf module install -y virt` |
| Validate setup | `sudo virt-host-validate` |
| Start libvirtd | `sudo systemctl enable --now libvirtd` |
| Create VM (CLI) | `sudo virt-install [options]` |
| List VMs | `sudo virsh list --all` |
| Cockpit access | `https://<server-ip>:9090` |

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Enable nested virtualization** (for VMs inside VMs):
    
    ```bash
    echo 'options kvm-intel nested=1' | sudo tee /etc/modprobe.d/kvm-intel.conf
    
    ```
    
- **Use qcow2 format** for VM disks (supports snapshots)
- **Allocate sufficient resources**: 2+ vCPUs, 2GB+ RAM for desktop OS

### ❌ Don't

- **Run VMs as root** (libvirt handles permissions)
- **Store ISOs in home directories** (use `/var/lib/libvirt/images/`)
- **Disable SELinux** (causes permission issues)

### Common Issues

- **"No network connectivity in VM"**:
Verify bridge configuration: `nmcli con show br0`
- **"Permission denied on ISO"**:
Fix permissions: `sudo chmod 664 /var/lib/libvirt/images/*.iso`
- **"CPU doesn't support virtualization"**:
Enable VT-x/AMD-V in BIOS

### Storage Management

```bash
# List storage pools
sudo virsh pool-list

# Create new storage pool
sudo virsh pool-define-as iso-pool dir - - - - "/mnt/iso"
sudo virsh pool-build iso-pool
sudo virsh pool-start iso-pool

```

---

## Summary Workflow

```bash
# 1. Verify hardware
grep -E 'vmx|svm' /proc/cpuinfo

# 2. Install packages
dnf install -y cockpit cockpit-machines
dnf module install -y virt

# 3. Enable services
systemctl enable --now cockpit.socket libvirtd

# 4. Configure firewall
firewall-cmd --permanent --add-service=cockpit && firewall-cmd --reload

# 5. Create bridge (via Cockpit)

# 6. Upload ISO
cp ubuntu.iso /var/lib/libvirt/images/

# 7. Create VM (via Cockpit or CLI)

```

> 💡 Golden Rule:
> 
> 
> **KVM = Enterprise virtualization**. Always:
> 
> - Verify **hardware support** first
> - Use **bridged networking** for production
> - Manage via **Cockpit** (GUI) or **virsh** (CLI)
> - Store ISOs in **libvirt-managed directories**!