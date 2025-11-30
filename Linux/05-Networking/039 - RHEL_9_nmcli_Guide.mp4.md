# 039: Assigning IP Addresses 


## 1. Understanding IP Assignment Methods

### Key Concepts

- **DHCP**: Automatic IP assignment (default for new connections)
- **Static**: Manual IP configuration (required for servers)
- **Multiple IPs**: Assign multiple addresses to one interface (virtual IPs)
- **IPv6**: Next-generation addressing (128-bit vs IPv4's 32-bit)

> 💡 Critical Note:
> 
> 
> RHEL 9 uses **NetworkManager**—all changes should be made via `nmcli` or `nmtui`.
> 

---

## 2. Creating Static IPv4 Connections

### Basic Static IP Setup

```bash
# Create static connection
nmcli con add type ethernet con-name ens224 ifname ens224 \\
  ipv4.addresses 192.168.0.145/24 \\
  ipv4.gateway 192.168.0.1 \\
  ipv4.dns 8.8.8.8 \\
  ipv4.method manual

# Activate connection
nmcli con up ens224

# Verify
ip a show ens224

```

> ⚠️ Fix from Your Notes:
> 
> - Correct parameter: `ipv4.addresses` (not `ipv4.Address`)
> - Correct parameter: `ipv4.gateway` (not `gw4`)
> - Must specify `ipv4.method manual` for static IPs

---

## 3. Adding Multiple IP Addresses

### Assign Secondary IP

```bash
# Add second IP to existing connection
nmcli con modify ens256 +ipv4.addresses 192.168.0.155/24

# Add additional DNS
nmcli con modify ens256 +ipv4.dns 8.8.4.4

# Reload and activate
sudo nmcli con reload
nmcli con up ens256

# Verify
ip a show ens256

```

### Remove Secondary IP

```bash
# Remove specific IP
nmcli con modify ens256 -ipv4.addresses 192.168.0.155/24

# Remove DNS entry
nmcli con modify ens256 -ipv4.dns 8.8.4.4

# Reload and activate
sudo nmcli con reload
nmcli con up ens256

```

> 💡 Key Syntax:
> 
> - **`+`** = Add value to list
> - = Remove specific value
> - Always use **`sudo nmcli con reload`** after manual config file edits

---

## 4. Managing Connections

### View and Delete Connections

```bash
# List all connections
nmcli con show

# Delete connection
nmcli con delete ens224

```

### Using nmtui for Interactive Setup

```bash
# Launch text UI
sudo nmtui

# Steps in nmtui:
# 1. Edit a connection
# 2. Select interface (ens256)
# 3. Set IPv4 to "Manual"
# 4. Add addresses, gateway, DNS
# 5. OK → Back → Quit

```

---

## 5. Configuring IPv6 Addresses

### Prerequisites

```bash
# Set static hostname (required for IPv6)
sudo hostnamectl set-hostname nehraclasses.local
hostnamectl status

```

### Create IPv6 Connection

```bash
# Create new connection (or modify existing)
nmcli con add type ethernet con-name ens224 ifname ens224

# Configure IPv6
nmcli con modify ens224 \\
  ipv6.addresses 'dfdd:df34:abc1::c0ad:8/64' \\
  ipv6.method manual

# Activate
nmcli con up ens224

# Verify
ip a show ens224

```

> ⚠️ Fix from Your Notes:
> 
> - Correct parameter: `ipv6.addresses` (not `ipv6,addresses`)
> - Correct parameter: `ipv6.method` (not `ipvd.method`)

---

## 6. Configuration File Location

### Network Script Files

```bash
# Location of config files
ls /etc/sysconfig/network-scripts/

# View connection config
cat /etc/sysconfig/network-scripts/ifcfg-ens224

```

> 🔍 File Structure Example:
> 
> 
> ```
> TYPE=Ethernet
> BOOTPROTO=none
> IPADDR=192.168.0.145
> PREFIX=24
> GATEWAY=192.168.0.1
> DNS1=8.8.8.8
> DEFROUTE=yes
> IPV4_FAILURE_FATAL=no
> IPV6INIT=yes
> IPV6ADDR=dfdd:df34:abc1::c0ad:8/64
> NAME=ens224
> DEVICE=ens224
> ONBOOT=yes
> 
> ```
> 

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Create static IPv4 | `nmcli con add type ethernet con-name NAME ifname IFACE ipv4.addresses IP/GW ipv4.gateway GW ipv4.dns DNS ipv4.method manual` |
| Add secondary IP | `nmcli con modify NAME +ipv4.addresses IP/GW` |
| Remove secondary IP | `nmcli con modify NAME -ipv4.addresses IP/GW` |
| Create IPv6 | `nmcli con modify NAME ipv6.addresses 'ADDR/PREFIX' ipv6.method manual` |
| Set hostname | `hostnamectl set-hostname FQDN` |
| Reload configs | `sudo nmcli con reload` |
| Activate connection | `nmcli con up NAME` |
| Delete connection | `nmcli con delete NAME` |

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Always specify `ipv4.method manual`** for static IPs
- **Use `+` and**  for multiple IP management
- **Set static hostname** before IPv6 configuration
- **Verify with `ip a`** after changes

### ❌ Don't

- **Edit config files directly** without `nmcli con reload`
- **Mix DHCP and static settings** in same connection
- **Use deprecated parameters** (`gw4`, `ip4.Address`)

### Common Issues

- **"Error: connection activation failed"**:
Check interface name with `ip link show`
- **Secondary IP not showing**:
Verify with `nmcli con show NAME | grep ipv4.addresses`
- **IPv6 not working**:
Ensure `IPV6INIT=yes` in config file

### Verification Commands

```bash
# Test IPv4 connectivity
ping -c 3 8.8.8.8

# Test IPv6 connectivity
ping6 -c 3 dfdd:df34:abc1::1

# Check DNS resolution
nslookup google.com

```

---

## Summary Workflow

### Static IPv4 with Multiple IPs

```bash
nmcli con add type ethernet con-name ens224 ifname ens224 \\
  ipv4.addresses 192.168.0.145/24 \\
  ipv4.gateway 192.168.0.1 \\
  ipv4.dns 8.8.8.8 \\
  ipv4.method manual

nmcli con modify ens224 +ipv4.addresses 192.168.0.155/24
nmcli con up ens224

```

### IPv6 Setup

```bash
hostnamectl set-hostname nehraclasses.local
nmcli con add type ethernet con-name ens224 ifname ens224
nmcli con modify ens224 ipv6.addresses 'dfdd:df34:abc1::c0ad:8/64' ipv6.method manual
nmcli con up ens224

```

> 💡 Golden Rule:
> 
> 
> **Always use correct nmcli parameters**—typos cause silent failures!
> 
> Verify every change with `ip a` and `nmcli con show NAME`.
>