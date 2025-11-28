# 040: NIC Teaming / Link Aggregation 


## 1. Understanding NIC Teaming

### Key Concepts

- **NIC Teaming** (Link Aggregation): Combines multiple physical NICs into one logical interface
- **Benefits**:
    - **Increased bandwidth**: Aggregate throughput of all NICs
    - **Redundancy**: Failover if one NIC fails
    - **Load balancing**: Distribute traffic across NICs
- **Common Use Cases**:
    - Data centers
    - High-availability servers
    - Network-intensive applications

### Teaming Runners (Modes)

| Runner | Purpose | Use Case |
| --- | --- | --- |
| `activebackup` | Only one NIC active (failover) | Critical redundancy |
| `roundrobin` | Packets rotated across NICs | Max bandwidth |
| `loadbalance` | Hash-based distribution | Balanced performance |
| `broadcast` | All packets sent on all NICs | Specialized networks |

> 💡 RHEL 8 Note:
> 
> 
> Uses **`teamd` daemon** (not legacy bonding) for better integration with NetworkManager.
> 

---

## 2. Installing and Preparing

### Install Teamd Package

```bash
# Check if installed
rpm -qa | grep teamd

# Install if missing
sudo dnf install teamd -y

```

### Verify Network Interfaces

```bash
# Check available interfaces
nmcli dev status

# Attach 2+ network interfaces in VM (e.g., ens224, ens256)

```

---

## 3. Creating NIC Team

### Step 1: Create Team Master

```bash
# Create team with active-backup runner
nmcli con add type team con-name team0 ifname team0 \\
  config '{"runner":{"name":"activebackup"}}'

# Configure IP address
nmcli con modify team0 \\
  ipv4.addresses 192.168.227.140/24 \\
  ipv4.gateway 192.168.227.1 \\
  ipv4.dns 8.8.8.8 \\
  ipv4.method manual \\
  connection.autoconnect yes

```

> ⚠️ Fix from Your Notes:
> 
> - Correct parameter: `config` (not `confif`)
> - Add **gateway** for proper routing
> - Use **`connection.autoconnect`** (not `connection.atutoconect`)

### Step 2: Add Team Slaves

```bash
# Add first slave interface
nmcli con add type team-slave con-name team0-port1 ifname ens224 master team0

# Add second slave interface
nmcli con add type team-slave con-name team0-port2 ifname ens256 master team0

```

> 🔍 Critical:
> 
> - Each slave must use a **different physical interface**
> - Slaves **inherit IP from master**—no IP config needed on slaves

---

## 4. Activating and Verifying Team

### Activate Connections

```bash
# Reload configurations
sudo nmcli con reload

# Bring up team and slaves
nmcli con up team0
nmcli con up team0-port1
nmcli con up team0-port2

```

### Verify Configuration

```bash
# Check IP assignment
ip a show team0

# View team status
teamdctl team0 state

# List all connections
nmcli con show

```

> 💡 Expected Output:
> 
> 
> ```bash
> # teamdctl team0 state
> setup:
>   runner: activebackup
> ports:
>   ens224
>     link watches:
>       link summary: up
>       instance[link_watch_0]:
>         name: ethtool
>         link: up
>   ens256
>     link watches:
>       link summary: up
>       instance[link_watch_0]:
>         name: ethtool
>         link: up
> runner:
>   active port: ens224
> 
> ```
> 

---

## 5. Testing Failover

### Simulate NIC Failure

```bash
# Disable first slave
nmcli con down team0-port1

# Verify failover
teamdctl team0 state
# Should show: active port: ens256

# Test connectivity
ping -c 3 8.8.8.8

# Restore first slave
nmcli con up team0-port1

```

> ✅ Active-Backup Behavior:
> 
> - Only **one port active** at a time
> - **Automatic failover** to backup when active fails
> - **No packet loss** during failover (with proper switch config)

---

## 6. Configuration Files

### Location and Structure

```bash
# Team master config
cat /etc/sysconfig/network-scripts/ifcfg-team0

# Team slave configs
cat /etc/sysconfig/network-scripts/ifcfg-team0-port1
cat /etc/sysconfig/network-scripts/ifcfg-team0-port2

```

> 🔍 Master Config Example:
> 
> 
> ```
> DEVICE=team0
> TEAM_CONFIG="{\\"runner\\":{\\"name\\":\\"activebackup\\"}}"
> DEVICETYPE=Team
> BOOTPROTO=none
> IPADDR=192.168.227.140
> PREFIX=24
> GATEWAY=192.168.227.1
> DNS1=8.8.8.8
> ONBOOT=yes
> 
> ```
> 

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Install teamd | `dnf install teamd` |
| Create team master | `nmcli con add type team con-name NAME ifname IFACE config '{"runner":{"name":"activebackup"}}'` |
| Add team slave | `nmcli con add type team-slave con-name SLAVE_NAME ifname PHYS_IFACE master TEAM_NAME` |
| Set IP on team | `nmcli con modify TEAM_NAME ipv4.addresses IP/GW ipv4.gateway GW ipv4.dns DNS ipv4.method manual` |
| View team status | `teamdctl TEAM_NAME state` |
| Reload configs | `nmcli con reload` |
| Activate team | `nmcli con up TEAM_NAME` |

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Use `activebackup`** for critical servers (simplest failover)
- **Test failover** regularly in maintenance windows
- **Match switch configuration** (LACP for `802.3ad` runner)
- **Monitor with `teamdctl`** during operations

### ❌ Don't

- **Mix different runner types** in same environment
- **Use teaming on desktops** (overkill for typical use)
- **Forget gateway configuration** on team master

### Common Issues

- **"Connection activation failed"**:
Verify physical interfaces exist: `ip link show`
- **No failover during test**:
Check: `teamdctl team0 state` → ensure both ports show "link: up"
- **Team not auto-starting**:
Verify: `connection.autoconnect yes` in master config

### Advanced Runners

```bash
# Round-robin (max bandwidth)
nmcli con add type team con-name team0 ifname team0 \\
  config '{"runner":{"name":"roundrobin"}}'

# Load balancing (802.3ad/LACP - requires switch support)
nmcli con add type team con-name team0 ifname team0 \\
  config '{"runner":{"name":"802.3ad","tx_hash":["eth","ipv4"]}}'

```

---

## Summary Workflow

```bash
# 1. Install teamd
dnf install teamd -y

# 2. Create team master
nmcli con add type team con-name team0 ifname team0 config '{"runner":{"name":"activebackup"}}'
nmcli con modify team0 ipv4.addresses 192.168.227.140/24 ipv4.gateway 192.168.227.1 ipv4.dns 8.8.8.8 ipv4.method manual

# 3. Add slaves
nmcli con add type team-slave con-name team0-port1 ifname ens224 master team0
nmcli con add type team-slave con-name team0-port2 ifname ens256 master team0

# 4. Activate
nmcli con reload
nmcli con up team0 team0-port1 team0-port2

# 5. Verify
teamdctl team0 state

```

> 💡 Golden Rule:
> 
> 
> **Test failover before production!**
> 
> Always:
> 
> - Use **`activebackup`** for simplicity
> - Verify with **`teamdctl`**
> - Ensure **gateway is configured** on team master