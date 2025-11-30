# 038: Network Management


## 1. Understanding Network Management Tools

### Key Tools Overview

| Tool | Purpose | RHEL 8 Status |
| --- | --- | --- |
| `ifconfig` | Legacy interface config | **Deprecated** (use `ip`) |
| `ip` | Modern network configuration | **Recommended** |
| `nmcli` | Command-line NetworkManager | **Primary tool** |
| `nmtui` | Text-based UI for NetworkManager | **User-friendly alternative** |

> 💡 Critical Note:
> 
> 
> RHEL 9 uses **NetworkManager** as the default network service—**not** traditional network scripts.
> 

---

## 2. Basic Network Inspection

### View Network Devices

```bash
# NetworkManager device status
nmcli dev status

# Detailed device info
nmcli dev show

# IP addresses (modern way)
ip addr
# OR
ip a

# Legacy (deprecated)
ifconfig

```

### View Connections

```bash
# List all connections
nmcli connection show
# OR
nmcli con show

# Show active connections
nmcli con show --active

```

---

## 3. Managing Network Connections with `nmcli`

### Create DHCP Connection

```bash
# Add new connection (DHCP)
nmcli con add type ethernet con-name ens224 ifname ens224

# Verify
nmcli con show
ip a

```

### Create Static IP Connection

```bash
# Add static IP connection
nmcli con add type ethernet con-name static2 ifname ens256 \\
  ip4 192.168.0.50/24 gw4 192.168.0.1

# Add DNS
nmcli con mod static2 ipv4.dns "8.8.8.8"

# Activate connection
nmcli con down static2
nmcli con up static2

```

### Connection Configuration Files

```bash
# Location of config files
ls /etc/sysconfig/network-scripts/

# View connection config
cat /etc/sysconfig/network-scripts/ifcfg-static2

```

> ⚠️ Note:
> 
> 
> NetworkManager **owns** these files—edit via `nmcli` or `nmtui`, not directly.
> 

---

## 4. Advanced Connection Management

### Control Auto-Connect

```bash
# Disable auto-connect on boot
nmcli con mod static2 connection.autoconnect no

# Re-enable auto-connect
nmcli con mod static2 connection.autoconnect yes

```

### Set User Permissions

```bash
# Allow specific users to manage connection
nmcli con mod static2 connection.permissions user:zybi,arslan

# Remove permissions
nmcli con mod static2 connection.permissions ""

```

### Modify Connection Settings

```bash
# Edit interactively
nmcli con edit static2

# Remove DNS entry
nmcli con mod static2 -ipv4.dns 8.8.8.8

# Add multiple DNS servers
nmcli con mod static2 ipv4.dns "8.8.8.8,8.8.4.4"

```

### Delete Connection

```bash
nmcli con del static2

```

---

## 5. Network Service Management

### Restart NetworkManager

```bash
# Reload all connections
sudo systemctl restart NetworkManager

# Check status
systemctl status NetworkManager

```

### Traditional Commands (Limited Use)

```bash
# Bring interface down/up (temporary)
sudo ifdown ens224
sudo ifup ens224

# Note: These use legacy scripts—prefer nmcli

```

---

## 6. Using `nmtui` (Text-Based UI)

### Launch Network Manager TUI

```bash
sudo nmtui

```

### Key Features

- **Edit connections**: Modify IP, DNS, gateway
- **Activate connections**: Bring interfaces up/down
- **Set system hostname**

> 💡 When to Use:
> 
> - Quick configuration without memorizing `nmcli` syntax
> - Teaching environments
> - Systems without GUI

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| List devices | `nmcli dev status` |
| List connections | `nmcli con show` |
| Create DHCP conn | `nmcli con add type ethernet con-name NAME ifname IFACE` |
| Create static conn | `nmcli con add type ethernet con-name NAME ifname IFACE ip4 IP/GW gw4 GW` |
| Set DNS | `nmcli con mod NAME ipv4.dns "8.8.8.8"` |
| Disable auto-connect | `nmcli con mod NAME connection.autoconnect no` |
| Set user permissions | `nmcli con mod NAME connection.permissions user:zybi` |
| Delete connection | `nmcli con del NAME` |
| Restart NetworkManager | `systemctl restart NetworkManager` |
| Launch TUI | `nmtui` |

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Use `nmcli` for scripts**, `nmtui` for interactive config
- **Always specify `con-name`** (not just interface name)
- **Verify with `ip a`** after changes
- **Use `nmcli con show NAME`** to check settings

### ❌ Don't

- **Edit `/etc/sysconfig/network-scripts/ifcfg-*` directly** (NetworkManager may overwrite)
- **Mix `ifup`/`ifdown` with NetworkManager** (causes state conflicts)
- **Use `ifconfig` in production scripts** (deprecated)

### Common Issues

- **"Connection activation failed"**:
Check interface name: `ip link show`
- **No IP after `nmcli con up`**:
Verify DHCP server or static IP settings
- **DNS not working**:
Check: `nmcli con show static2 | grep dns`

### Verification Commands

```bash
# Test connectivity
ping -c 3 8.8.8.8

# Check DNS resolution
nslookup google.com

# View routing table
ip route show

```

---

## Summary Workflow

### Create Static Connection

```bash
nmcli con add type ethernet con-name static2 ifname ens256 \\
  ip4 192.168.0.50/24 gw4 192.168.0.1
nmcli con mod static2 ipv4.dns "8.8.8.8"
nmcli con up static2

```

### Manage Connection

```bash
nmcli con mod static2 connection.autoconnect no
nmcli con mod static2 connection.permissions user:zybi
nmcli con del static2

```

> 💡 Golden Rule:
> 
> 
> **NetworkManager is king in RHEL 8**. Always:
> 
> - Use `nmcli` or `nmtui`—not legacy tools
> - Verify with `ip a` (not `ifconfig`)
> - Restart `NetworkManager` service after major changes