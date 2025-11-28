# 042: Cockpit: Web-Based Server Management Tool

## 1. What is Cockpit?

### Core Features

- **Web-based GUI** for Linux server management
- **Real-time monitoring**: CPU, memory, disk, network
- **Multi-server management**: Control multiple servers from one dashboard
- **Container management**: Docker/Podman support
- **User-friendly**: Accessible to sysadmins and developers alike

### Key Advantages

- **No client software needed**—works in any modern browser
- **Secure by default**: Uses HTTPS (port 9090)
- **Minimal resource usage**: Only runs when accessed
- **Integrated with systemd**: View logs, manage services

> 💡 Critical Insight:
> 
> 
> Cockpit uses a **socket activation** model—service starts only when accessed (port 9090).
> 

---

## 2. Installing Cockpit

### Basic Installation

```bash
# Install Cockpit core
sudo dnf install -y cockpit

# Optional: Add virtual machine management
sudo dnf install -y cockpit-machines

# Enable and start (socket activation)
sudo systemctl enable --now cockpit.socket
sudo systemctl status cockpit.socket

```

> ⚠️ Why Socket?:
> 
> - **Efficiency**: Service starts only when accessed
> - **Security**: No persistent process running
> - **Scalability**: Handles multiple connections efficiently

---

## 3. Configuring Firewall for Cockpit

### Allow Cockpit Through Firewall

```bash
# Check current firewall rules
firewall-cmd --list-all

# Add Cockpit service permanently
firewall-cmd --permanent --add-service=cockpit

# Reload firewall
firewall-cmd --reload

```

> 🔍 Cockpit Service Details:
> 
> - Predefined in firewalld as `cockpit`
> - Uses **TCP port 9090**
> - No need to manually open port—use service name

---

## 4. Accessing Cockpit Web Interface

### Connect via Browser

1. Get server IP address:
    
    ```bash
    ip a
    # Note IP (e.g., 192.168.100.4)
    
    ```
    
2. Open browser and navigate to:
    
    ```
    <https://192.168.100.4:9090>
    
    ```
    
3. **Accept self-signed certificate** (first time only)
4. Login with **system credentials** (root or sudo user)

> ⚠️ Security Note:
> 
> - Always use **HTTPS** (not HTTP)
> - Cockpit uses **system authentication**—same as SSH/console login

---

## 5. Key Cockpit Features

### Dashboard Overview

- **System metrics**: Real-time CPU, memory, disk usage
- **Network activity**: Bandwidth usage per interface
- **Storage**: Disk usage and mount points
- **Services**: Running systemd services

### Management Capabilities

| Feature | Description |
| --- | --- |
| **Terminal** | Built-in web terminal (like SSH) |
| **Services** | Start/stop/restart systemd services |
| **Logs** | View systemd journal logs |
| **Accounts** | Create/manage user accounts |
| **Networking** | Configure IP addresses, firewall |
| **Storage** | Manage filesystems, LVM, RAID |
| **Containers** | Manage Podman/Docker containers (with cockpit-machines) |

---

## 6. Multi-Server Management

### Add Additional Servers

1. In Cockpit dashboard → **"Add server"**
2. Enter IP address of another Cockpit-enabled server
3. Authenticate with credentials for that server

> 💡 Requirement:
> 
> 
> Target server must have **Cockpit installed and accessible** on port 9090.
> 

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Install Cockpit | `sudo dnf install -y cockpit` |
| Install VM support | `sudo dnf install -y cockpit-machines` |
| Enable service | `sudo systemctl enable --now cockpit.socket` |
| Check status | `systemctl status cockpit.socket` |
| Open firewall | `firewall-cmd --permanent --add-service=cockpit && firewall-cmd --reload` |
| Get IP | `ip a` |

---

## 8. Best Practices & Security

### ✅ Do

- **Use strong passwords** (Cockpit uses system auth)
- **Restrict access** via firewall (only trusted IPs):
    
    ```bash
    firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="cockpit" accept'
    
    ```
    
- **Keep Cockpit updated**: `sudo dnf update cockpit`
- **Use HTTPS exclusively** (never expose to public internet)

### ❌ Don't

- **Disable TLS/SSL** (Cockpit requires HTTPS)
- **Use root account** for daily tasks (create sudo user instead)
- **Expose to public internet** without reverse proxy + auth

### Hardening Tips

- **Change default port** (advanced):
Add:
    
    ```bash
    sudo semanage port -a -t websm_port_t -p tcp 9191
    sudo firewall-cmd --permanent --add-port=9191/tcp
    sudo systemctl edit cockpit.socket
    
    ```
    
    ```
    [Socket]
    ListenStream=9191
    
    ```
    

---

## Summary Workflow

```bash
# 1. Install
sudo dnf install -y cockpit cockpit-machines

# 2. Enable
sudo systemctl enable --now cockpit.socket

# 3. Open firewall
firewall-cmd --permanent --add-service=cockpit
firewall-cmd --reload

# 4. Access
# Browser: https://<server-ip>:9090

```

> 💡 Golden Rule:
> 
> 
> **Cockpit = Secure remote management**. Always:
> 
> - Use **firewall restrictions**
> - Keep **system updated**
> - Never expose to **public internet** without additional security layers!