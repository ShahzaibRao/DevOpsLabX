# 049: DNS Configuration in Linux with BIND

## 1. Understanding DNS and BIND

### Core Concepts

- **DNS (Domain Name System)**: Translates human-readable names (e.g., `www.example.com`) to IP addresses
- **BIND (Berkeley Internet Name Domain)**: Most widely used DNS server software on Linux
- **Zone Types**:
    - **Forward Zone**: Name → IP (A records)
    - **Reverse Zone**: IP → Name (PTR records)

### Key Components

| Component | Purpose |
| --- | --- |
| **SOA Record** | Start of Authority (zone metadata) |
| **NS Record** | Name Server for the domain |
| **A Record** | Hostname to IPv4 address |
| **PTR Record** | IPv4 address to hostname |
| **CNAME** | Alias for another hostname |

---

## 2. Installing and Configuring BIND

### Initial Setup

```bash
# Set hostname
sudo hostnamectl set-hostname ns1.raoshahzaib.local

# Install BIND
sudo dnf install bind bind-utils -y

# Enable and start service
sudo systemctl enable --now named
sudo systemctl status named

```

---

## 3. Main BIND Configuration (`/etc/named.conf`)

### Critical Changes

```bash
sudo vim /etc/named.conf

```

```
// Comment out these lines to allow external queries
// listen-on port 53 { 127.0.0.1; };
// listen-on-v6 port 53 { ::1; };

// Allow queries from your network
options {
    allow-query { localhost; 192.168.122.0/24; };
    // Other options...
};

// Forward Zone
zone "raoshahzaib.local" IN {
    type master;
    file "raoshahzaib.local.db";
    allow-update { none; };
};

// Reverse Zone
zone "122.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.122.db";
    allow-update { none; };
};

```

> ⚠️ Critical Notes:
> 
> - **Comment out `listen-on`** to bind to all interfaces
> - **Network range** must match your environment (`192.168.122.0/24`)
> - **Reverse zone name**: IP octets in reverse order (`122.168.192` for `192.168.122.x`)

---

## 4. Creating DNS Zone Files

### Forward Zone (`/var/named/raoshahzaib.local.db`)

```bash
sudo vim /var/named/raoshahzaib.local.db

```

```
$TTL 86400
@   IN  SOA     ns1.raoshahzaib.local. root.raoshahzaib.local. (
        3       ; Serial (increment on changes)
        3600    ; Refresh
        1800    ; Retry
        604800  ; Expire
        86400   ; Minimum TTL
)

; Name Server
@       IN  NS      ns1.raoshahzaib.local.

; A Records
ns1     IN  A       192.168.122.14
www     IN  A       192.168.122.101

; CNAME Record
ftp     IN  CNAME   www.raoshahzaib.local.

```

### Reverse Zone (`/var/named/192.168.122.db`)

```bash
sudo vim /var/named/192.168.122.db

```

```
$TTL 86400
@   IN  SOA     ns1.raoshahzaib.local. root.raoshahzaib.local. (
        3       ; Serial
        3600    ; Refresh
        1800    ; Retry
        604800  ; Expire
        86400   ; Minimum TTL
)

; Name Server
@       IN  NS      ns1.raoshahzaib.local.

; PTR Records
14      IN  PTR     ns1.raoshahzaib.local.
101     IN  PTR     www.raoshahzaib.local.

```

> 💡 Key Syntax Rules:
> 
> - **SOA email**: Use `root.domain.` (dot at end = absolute name)
> - **Serial number**: Increment on every change (use date format: `2023101501`)
> - **PTR records**: Only last octet of IP (`14` for `192.168.122.14`)

---

## 5. Validation and Testing

### Check Configuration Syntax

```bash
# Validate main config
sudo named-checkconf /etc/named.conf

# Validate zones
sudo named-checkzone raoshahzaib.local /var/named/raoshahzaib.local.db
sudo named-checkzone 122.168.192.in-addr.arpa /var/named/192.168.122.db

```

### Apply Configuration

```bash
# Restart BIND
sudo systemctl restart named

# Configure firewall
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload

```

> 🔍 Firewall Note:
> 
> 
> Use `--add-service=dns` (not `--add-port=53/udp`)—includes both UDP/TCP
> 

---

## 6. Client Configuration and Testing

### Configure DNS Client

```bash
# Temporary client config
sudo vim /etc/resolv.conf

```

```
nameserver 192.168.122.14  # Your DNS server IP

```

> ⚠️ Permanent Client Config:
> 
> 
> Use NetworkManager or systemd-resolved to avoid `/etc/resolv.conf` being overwritten
> 

### Test DNS Resolution

```bash
# Forward lookup
dig www.raoshahzaib.local
dig ns1.raoshahzaib.local

# Reverse lookup
dig -x 192.168.122.101

# Test all records
nslookup www.raoshahzaib.local
host www.raoshahzaib.local

```

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| Install BIND | `dnf install bind bind-utils` |
| Validate config | `named-checkconf /etc/named.conf` |
| Validate zone | `named-checkzone domain /path/to/zone.db` |
| Test forward DNS | `dig hostname.domain` |
| Test reverse DNS | `dig -x IP_ADDRESS` |
| Firewall rule | `firewall-cmd --permanent --add-service=dns` |

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Increment SOA serial** on every zone change
- **Use FQDNs** in zone files (end with dot: `ns1.domain.`)
- **Test with `dig`** before client deployment
- **Backup zone files** before editing

### ❌ Don't

- **Edit `/etc/resolv.conf` directly** on clients (use proper network config)
- **Use localhost IPs** in public zones
- **Forget firewall rules** (DNS uses UDP 53 primarily, TCP 53 for large responses)

### Common Issues

- **"connection timed out"**:
Check firewall: `sudo firewall-cmd --list-services | grep dns`
- **"NXDOMAIN" (non-existent domain)**:
Verify zone name in `named.conf` matches zone file
- **Permission denied on zone files**:
Fix ownership: `sudo chown root:named /var/named/*.db`
- **Serial number not updated**:
Clients won't refresh zone data

### SELinux Considerations

```bash
# If SELinux blocks BIND
sudo setsebool -P named_write_master_zones 1
sudo restorecon -R /var/named

```

---

## Summary Workflow

```bash
# 1. Install and configure
hostnamectl set-hostname ns1.domain.local
dnf install bind bind-utils -y

# 2. Edit /etc/named.conf (allow queries, add zones)

# 3. Create zone files in /var/named/

# 4. Validate and restart
named-checkconf
named-checkzone domain /var/named/zone.db
systemctl restart named

# 5. Configure firewall and test
firewall-cmd --permanent --add-service=dns
dig hostname.domain

```

> 💡 Golden Rule:
> 
> 
> **Always validate before restarting**! Use:
> 
> - `named-checkconf` for config syntax
> - `named-checkzone` for zone files
> - Increment **SOA serial** on every change