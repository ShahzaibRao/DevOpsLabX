# 041: Firewall configuration


## 1. What is a Firewall?

### Core Purpose

- **Traffic Filter**: Controls incoming/outgoing network traffic based on security rules
- **Security Barrier**: Protects private networks from unauthorized internet access
- **Critical for**:
    - Organizations with sensitive data
    - Servers exposed to public networks
    - Preventing lateral movement during breaches

> 💡 Key Insight:
> 
> 
> Firewalls implement **defense in depth**—even if one device is compromised, others stay protected.
> 

---

## 2. Firewalld vs. iptables

| Feature | **Firewalld** (RHEL 8 Default) | **iptables** (Legacy) |
| --- | --- | --- |
| **Management** | Dynamic (runtime changes) | Static (requires restart) |
| **Configuration** | Zones, Services, Rich Rules | Raw packet filtering rules |
| **Interface** | `firewall-cmd`, GUI, XML files | Command-line only |
| **Persistence** | Automatic (`--permanent`) | Manual save/restore |
| **Use Case** | Modern servers, dynamic environments | Legacy systems, custom rules |

> ⚠️ RHEL 8 Note:
> 
> 
> **Firewalld is default**—iptables rules are managed through firewalld's backend.
> 

---

## 3. Firewalld Installation & Basic Setup

### Install and Enable

```bash
# Remove if conflicting (rarely needed)
sudo dnf remove firewalld -y

# Install and enable
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld
sudo systemctl status firewalld

# Verify running
firewall-cmd --state    # Should return "running"

```

---

## 4. Core Concepts: Zones and Services

### Zones (Predefined Security Levels)

| Zone | Policy | Use Case |
| --- | --- | --- |
| `public` | Default for untrusted networks | Internet-facing servers |
| `dmz` | Limited access to LAN | Public servers needing internal access |
| `internal` | Trusted internal network | Corporate LAN |
| `home` | Trustworthy home network | Home PCs |
| `drop` | **All incoming dropped** | Highly secure systems |
| `block` | All incoming rejected | Similar to drop but sends rejection |

### View Zones

```bash
# List all zones
firewall-cmd --get-zones

# View active zones
firewall-cmd --get-active-zones

# Check default zone
firewall-cmd --get-default-zone

# Inspect zone rules
cat /usr/lib/firewalld/zones/public.xml

```

### Services (Predefined Port Rules)

- **Service**: Named group of ports/protocols (e.g., `http` = TCP 80)
- **Location**:
    - Default services: `/usr/lib/firewalld/services/`
    - Custom services: `/etc/firewalld/services/`

```bash
# List all available services
firewall-cmd --get-services

# View service definition
cat /usr/lib/firewalld/services/http.xml

```

---

## 5. Managing Firewalld Rules

### Basic Service Management

```bash
# Add HTTP service (runtime only)
firewall-cmd --add-service=http

# Add permanently
firewall-cmd --permanent --add-service=http

# Reload to apply permanent rules
firewall-cmd --reload

# Verify
firewall-cmd --list-services

```

### Port Management

```bash
# Add port directly (not recommended—use services when possible)
firewall-cmd --permanent --add-port=80/tcp

# List ports
firewall-cmd --list-ports

```

### Zone Management

```bash
# Assign interface to zone
firewall-cmd --zone=home --change-interface=ens33

# Add service to specific zone
firewall-cmd --permanent --zone=public --add-service=dns

# List zone details
firewall-cmd --list-all --zone=public

```

---

## 6. Advanced Firewalld Features

### Port Forwarding

```bash
# Forward port 80 → 8080 on same server
firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080

# Forward to different server
firewall-cmd --permanent --add-masquerade
firewall-cmd --permanent --add-forward-port=port=443:proto=tcp:toport=443:toaddr=192.168.100.44

# List forward rules
firewall-cmd --list-forward-ports

```

### Custom Services

```bash
# Create custom service
sudo cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/myapp.xml
sudo vim /etc/firewalld/services/myapp.xml

```

Example `myapp.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MyApp</short>
  <description>Custom application on port 8080</description>
  <port protocol="tcp" port="8080"/>
</service>

```

```bash
# Reload and use
firewall-cmd --reload
firewall-cmd --permanent --add-service=myapp

```

---

## 7. Practical Example: Web Server Setup

### Step-by-Step

```bash
# 1. Install web server
sudo dnf install httpd -y
sudo systemctl enable --now httpd

# 2. Create test page
echo "Subhan ALLAH, Beshk ALLAH pak sb se bara hai" | sudo tee /var/www/html/index.html

# 3. Test locally
curl localhost

# 4. Allow HTTP through firewall
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

# 5. Verify
firewall-cmd --list-services | grep http

```

> ✅ Result: Web server accessible from external networks.
> 

---

## 8. Key Commands Reference

| Task | Command |
| --- | --- |
| Check status | `firewall-cmd --state` |
| List active zones | `firewall-cmd --get-active-zones` |
| List services | `firewall-cmd --list-services` |
| Add service (permanent) | `firewall-cmd --permanent --add-service=http` |
| Add port | `firewall-cmd --permanent --add-port=8080/tcp` |
| Port forwarding | `firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080` |
| Masquerading | `firewall-cmd --permanent --add-masquerade` |
| Reload config | `firewall-cmd --reload` |
| Remove service | `firewall-cmd --permanent --remove-service=http` |

---

## 9. Configuration File Locations

| Purpose | Path |
| --- | --- |
| Default zones | `/usr/lib/firewalld/zones/` |
| Default services | `/usr/lib/firewalld/services/` |
| **Custom zones** | `/etc/firewalld/zones/` |
| **Custom services** | `/etc/firewalld/services/` |
| Main config | `/etc/firewalld/firewalld.conf` |

> 💡 Golden Rule:
> 
> 
> **Always edit custom configs in `/etc/firewalld/`**—files in `/usr/lib/` get overwritten on updates.
> 

---

## 10. Best Practices & Troubleshooting

### ✅ Do

- **Use services instead of ports** (more maintainable)
- **Test rules before making permanent**
- **Use `-permanent` + `-reload`** for persistent changes
- **Document custom services** with descriptions

### ❌ Don't

- **Edit `/usr/lib/firewalld/` files directly**
- **Mix runtime and permanent rules** without understanding
- **Disable firewall entirely** for troubleshooting—use `-add-service` instead

### Common Issues

- **"Command failed"**:
Check zone name: `firewall-cmd --get-zones`
- **Rules not applying**:
Remember: `-permanent` requires `firewall-cmd --reload`
- **Port forwarding not working**:
Ensure `masquerade` is enabled for external forwarding

### GUI Management

```bash
# Install firewall-config GUI
sudo dnf install firewall-config -y
# Launch: firewall-config

```

---

## Summary Workflow

### Allow Web Server

```bash
dnf install httpd -y
systemctl enable --now httpd
firewall-cmd --permanent --add-service=http
firewall-cmd --reload

```

### Create Custom Service

```bash
cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/myapp.xml
# Edit myapp.xml → set port 8080
firewall-cmd --reload
firewall-cmd --permanent --add-service=myapp

```

> 💡 Golden Rule:
> 
> 
> **Firewalld = Zones + Services**. Always:
> 
> - Use **predefined services** when possible
> - Store **custom configs in `/etc/firewalld/`**
> - **Reload** after permanent changes!