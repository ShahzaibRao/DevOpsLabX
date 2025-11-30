# 056: NFS Server Configuration 
## 1. Understanding NFS (Network File System)

### Core Concepts

- **NFS**: Allows sharing directories/files over a network
- **Server**: Hosts shared directories
- **Client**: Mounts remote directories as local filesystems
- **Use Cases**:
    - Shared storage for web servers
    - Centralized home directories
    - Application data sharing

### Key NFS Services

| Service | Port | Purpose |
| --- | --- | --- |
| **nfs** | 2049 | Main NFS service |
| **rpc-bind** | 111 | Port mapper |
| **mountd** | Variable | Mount requests |

---

## 2. NFS Server Configuration

### Initial Setup

```bash
# Set hostname
sudo hostnamectl set-hostname nfsserver.shahzaibrao.local

# Install NFS utilities
sudo dnf install nfs-utils -y

# Enable and start services
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server

```

### Create Shared Directory

```bash
# Create directory structure
sudo mkdir -p /mnt/nfs_shares/docs

# Set proper ownership and permissions
sudo chown -R nobody:nobody /mnt/nfs_shares/docs
sudo chmod -R 777 /mnt/nfs_shares/docs

```

> ⚠️ Critical Fixes from Your Notes:
> 
> - **`chwon`** → **`chown`**
> - **`/mnt/nfs_server`** → **`/mnt/nfs_shares`** (path consistency)

### Configure Exports

```bash
sudo vim /etc/exports

```

```
/mnt/nfs_shares/docs 192.168.10.0/24(rw,sync,no_root_squash)

```

> 🔍 Export Options Explained:
> 
> - **`rw`**: Read-write access
> - **`sync`**: Commit changes to disk before acknowledging
> - **`no_root_squash`**: Allow root access (use cautiously!)
> - **`root_squash`**: Map root to nobody (more secure)

### Apply Export Configuration

```bash
# Export all shares
sudo exportfs -arv

# Verify exports
sudo exportfs -s
sudo showmount -e localhost

```

### Configure Firewall

```bash
# Allow NFS services
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload

```

> 💡 Note:
> 
> - **`mount`** → **`mountd`** (correct service name)
> - All three services are required for NFSv4

---

## 3. NFS Client Configuration

### Install NFS Client

```bash
# On client machine
sudo hostnamectl set-hostname nfsclient.shahzaibrao.local
sudo dnf install nfs-utils -y

```

### Discover and Mount NFS Share

```bash
# Discover available shares
showmount -e 192.168.10.112

# Create mount point
sudo mkdir -p /mnt/client_shares

# Mount NFS share
sudo mount -t nfs 192.168.10.112:/mnt/nfs_shares/docs /mnt/client_shares

# Verify mount
mount | grep -i nfs
df -hT | grep nfs

```

> ⚠️ Critical Fixes:
> 
> - **`clinet_shares`** → **`client_shares`**
> - Use **server IP** (192.168.10.112), not network range

---

## 4. Testing NFS Functionality

### Test File Sharing

```bash
# On SERVER: Create file
sudo touch /mnt/nfs_shares/docs/server_nfs_file.txt

# On CLIENT: Verify file exists
ls -l /mnt/client_shares/

# On CLIENT: Create file
sudo touch /mnt/client_shares/client_nfs_file.txt

# On SERVER: Verify file exists
ls -l /mnt/nfs_shares/docs/

```

### Auto-Mount on Boot (Client)

```bash
# Add to /etc/fstab
echo "192.168.10.112:/mnt/nfs_shares/docs /mnt/client_shares nfs defaults 0 0" | sudo tee -a /etc/fstab

```

> 💡 fstab Options:
> 
> - **`_netdev`**: Wait for network before mounting
> - **`soft`**: Fail quickly on server issues
> - **`hard`**: Retry indefinitely (default)

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Install NFS | `dnf install nfs-utils` |
| Start server | `systemctl enable --now nfs-server` |
| Configure exports | `/etc/exports` |
| Apply exports | `exportfs -arv` |
| Show exports | `showmount -e SERVER_IP` |
| Mount share | `mount -t nfs SERVER:/path /mount` |
| Firewall rules | `firewall-cmd --add-service={nfs,rpc-bind,mountd}` |

---

## 6. Best Practices & Troubleshooting

### ✅ Do

- **Use `root_squash`** for security (default)
- **Test with `showmount -e`** before mounting
- **Monitor with `nfsstat`** for performance issues
- **Use NFSv4** (enabled by default in RHEL 8)

### ❌ Don't

- **Use `no_root_squash`** in production (security risk)
- **Forget firewall rules** (causes "connection refused")
- **Skip `exportfs -arv`** after config changes

### Common Issues

- **"Permission denied"**:
Check export options and directory permissions
- **"Connection timed out"**:
Verify firewall: `firewall-cmd --list-services | grep -E 'nfs|rpc|mount'`
- **"Stale file handle"**:
Restart NFS services on server: `systemctl restart nfs-server`
- **Mount fails on boot**:
Add `_netdev` to fstab: `SERVER:/path /mount nfs defaults,_netdev 0 0`

### Security Hardening

```bash
# Restrict to specific clients
/mnt/nfs_shares/docs 192.168.10.50(rw,sync,root_squash)

# Use secure ports only
# In /etc/sysconfig/nfs:
RPCNFSDARGS="-N 2 -N 3"  # Disable NFSv2/v3

```

---

## Summary Workflow

```bash
# SERVER
mkdir -p /mnt/nfs_shares/docs
chown nobody:nobody /mnt/nfs_shares/docs
echo "/mnt/nfs_shares/docs 192.168.10.0/24(rw,sync,root_squash)" > /etc/exports
exportfs -arv
firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd}

# CLIENT
mkdir -p /mnt/client_shares
mount -t nfs 192.168.10.112:/mnt/nfs_shares/docs /mnt/client_shares

```

> 💡 Golden Rule:
> 
> 
> **Always use `root_squash`** for security!
> 
> Test with `showmount -e` before mounting, and **never skip firewall configuration**.
>