# 01: LAB-Setup
# **Proxmox installation - step-by-step tutorial for beginners**

# 🧩 Build Ubuntu Cloud-Init Template in Proxmox

This guide explains how to create a **reusable Ubuntu Cloud-Init VM template** in Proxmox VE.

You can use this template to quickly deploy pre-configured VMs with unique hostnames, IPs, SSH keys, and passwords — all automated via **Cloud-Init**.

---

## 📘 What is Cloud-Init?

**Cloud-Init** is a tool that automatically configures a virtual machine when it first boots.

It handles setup tasks such as:

- Setting the **hostname**
- Creating users and **passwords**
- Adding **SSH keys**
- Applying **network settings**
- Running custom scripts on first boot

In **Proxmox**, Cloud-Init lets you create **templates** that can be cloned easily — each clone booting up with its own configuration, without manual setup.

---

## ⚙️ Steps to Build the Template

---

### **1. Download the Ubuntu Cloud Image**

Use the official Ubuntu cloud image for your chosen release (here we use Ubuntu 22.04 “Jammy”).

```bash
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

```

This `.img` file is preconfigured for Cloud-Init and optimized for virtualization platforms like Proxmox.

---

### **2. Create and Configure the VM**

Replace placeholders with your own values:

- `<VM-ID>` → Example: `100`
- `<username>` → Your preferred default user
- `<password>` → A secure password
- `<path/to/public-key>` → Path to your SSH public key file (e.g., `~/.ssh/id_rsa.pub`)

```bash
# Create a new VM
qm create <VM-ID> --name ubuntu-cloud --cpu host --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

# Import the downloaded disk image
qm importdisk <VM-ID> jammy-server-cloudimg-amd64.img local-lvm

# Attach the disk to the VM
qm set <VM-ID> --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-<VM-ID>-disk-0

# Configure boot order
qm set <VM-ID> --boot c --bootdisk scsi0

# Add a Cloud-Init drive
qm set <VM-ID> --ide2 local-lvm:cloudinit

# Set Cloud-Init parameters
qm set <VM-ID> --ciuser <username> --cipassword "<password>" --sshkeys <path/to/public-key> --ipconfig0 ip=dhcp

# Enable QEMU Guest Agent and serial console
qm set <VM-ID> --agent 1 --serial0 socket --vga serial0

# (Optional) Resize disk to desired size
qm resize <VM-ID> scsi0 20G

# Start the VM
qm start <VM-ID>

```

---

### **3. Verify and Update**

After the VM boots, ensure:

- You can log in via SSH using your provided credentials or key.
- The network configuration is correct (`ip a` or `hostname -I`).
- QEMU Guest Agent is running (`systemctl status qemu-guest-agent`).

Once verified, **shut down** the VM before creating a template.

---

### **4. Convert the VM to a Template**

Convert your configured VM into a reusable template:

```bash
qm template <VM-ID>

```

Now this VM can be cloned quickly for future deployments.

---

### **5. Clone Template to Create New VMs**

When you need a new VM, clone it from your template.

Replace `<NEW-VM-ID>` with a new number (e.g., `101`):

```bash
# Clone the template
qm clone <VM-ID> <NEW-VM-ID> --name ubuntu-clone --full

# Configure Cloud-Init settings for the new VM
qm set <NEW-VM-ID> --ciuser <username> --cipassword "<password>" --sshkeys <path/to/public-key> --ipconfig0 ip=dhcp

# Start the new VM
qm start <NEW-VM-ID>

```

Cloud-Init will automatically:

- Set the new hostname
- Apply the new SSH keys and credentials
- Configure the network

---

## 🧠 Notes & Best Practices

- Replace all `<placeholders>` with your actual values.
- Always use **unique VM IDs** for each VM.
- To assign static IPs, modify `-ipconfig0` (e.g. `ip=192.168.1.50/24,gw=192.168.1.1`).
- The **QEMU Guest Agent** improves interaction between Proxmox and the VM (shutdowns, IP reporting, etc.).
- Cloud-Init VMs are best used for **Linux distributions** that support it (Ubuntu, Debian, CentOS, Rocky, etc.).
- For **Windows VMs**, Cloud-Init is not used; instead, install the `virtio-win` drivers.

---

## 🚀 Benefits of Using Cloud-Init Templates in Proxmox

✅ **Fast deployments** — spin up new VMs in seconds.

✅ **Consistent configurations** — every VM follows the same base image.

✅ **Automated setup** — users, passwords, and network handled automatically.

✅ **Easily scalable** — perfect for labs, clusters, or cloud-like environments.

---