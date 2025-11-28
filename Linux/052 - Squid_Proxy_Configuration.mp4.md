# 052: Squid Proxy Server Configuration | Restrict Websites, Hours, Files & Downloads
---

## 1. Understanding Squid Proxy

### Core Features

- **Web caching**: Improves browsing speed by storing frequently accessed content
- **Access control**: Block websites, file types, and set time restrictions
- **Bandwidth management**: Control download speeds for users/networks
- **Security**: Filter malicious content and enforce internet policies

### Key Concepts

- **ACL (Access Control List)**: Defines rules for traffic filtering
- **http_access**: Applies allow/deny rules based on ACLs
- **Order matters**: Rules processed top-to-bottom (first match wins)

---

## 2. Basic Squid Installation and Setup

### Install and Configure

```bash
# Install Squid
sudo dnf install -y squid

# Enable and start service
sudo systemctl enable --now squid
sudo systemctl status squid

```

### Configure Firewall

```bash
# Allow Squid port (3128)
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload

```

### Get Server IP

```bash
ip a show eth0    # Note your server IP (e.g., 192.168.1.10)

```

---

## 3. Basic Network Access Configuration

### Edit Main Config

```bash
sudo vim /etc/squid/squid.conf

```

### Allow Local Network

```
# Define local network ACL
acl localnet src 192.168.1.0/24

# Allow local network access
http_access allow localnet

# Deny all other requests (must be last)
http_access deny all

```

> ⚠️ Critical Rule Order:
> 
> - **Allow rules BEFORE deny rules**
> - **`http_access deny all` must be LAST**

---

## 4. Blocking Websites

### Create Blocked Sites List

```bash
sudo vim /etc/squid/blocksites

```

```
\\.facebook\\.com
\\.twitter\\.com
\\.flipkart\\.com
\\.youtube\\.com

```

> 💡 Regex Tips:
> 
> - Use `\\.` to match literal dots
> - `$` matches end of URL
> - Case-insensitive by default

### Configure Squid to Block Sites

```
# Add to /etc/squid/squid.conf
acl blocksites url_regex -i "/etc/squid/blocksites"
http_access deny blocksites

```

> 🔍 Flag Explanation:
> 
> - `i` = Case-insensitive matching

---

## 5. Blocking File Types

### Create File Block List

```bash
sudo vim /etc/squid/blockfiles.acl

```

```
\\.torrent$
\\.mp3$
\\.3gp$
\\.mp4$
\\.exe$

```

### Configure File Blocking

```
# Add to /etc/squid/squid.conf
acl blockfiles urlpath_regex -i "/etc/squid/blockfiles.acl"
http_access deny blockfiles

```

> ⚠️ Note:
> 
> - `urlpath_regex` matches **file paths** (not full URLs)
> - `$` ensures matches end of filename

---

## 6. Time-Based Access Control

### Restrict Work Hours

```
# Block access outside business hours (10 AM - 7 PM)
acl work_hours time MTWHF 10:00-19:00
http_access deny !work_hours

```

> 💡 Time Format:
> 
> - Days: `SMTWHFA` (Sun, Mon, Tue, Wed, Thu, Fri, Sat, All)
> - `!work_hours` = NOT during work hours

### Alternative: Allow Only During Hours

```
# Allow only during work hours
acl work_hours time MTWHF 10:00-19:00
http_access allow work_hours
http_access deny all

```

---

## 7. Bandwidth Throttling

### Configure Speed Control

```
# Define network for speed control
acl speedcontrol src 192.168.1.109/32

# Enable delay pools
delay_pools 1
delay_class 1 2
# Format: delay_parameters POOL restore_rate/max_quota
delay_parameters 1 524288/524288 52428/52428
delay_access 1 allow speedcontrol

```

> 🔍 Bandwidth Values:
> 
> - **524288** = 512 KB/s (restore rate)
> - **52428** = 51.2 KB/s (burst rate)
> - Values in **bytes/second**

---

## 8. Complete Configuration Example

### `/etc/squid/squid.conf`

```
# Basic network
acl localnet src 192.168.1.0/24
http_access allow localhost
http_access allow localnet

# Blocked sites
acl blocksites url_regex -i "/etc/squid/blocksites"
http_access deny blocksites

# Blocked files
acl blockfiles urlpath_regex -i "/etc/squid/blockfiles.acl"
http_access deny blockfiles

# Work hours (allow only 10AM-7PM Mon-Fri)
acl work_hours time MTWHF 10:00-19:00
http_access allow work_hours

# Bandwidth control
acl speedcontrol src 192.168.1.109/32
delay_pools 1
delay_class 1 2
delay_parameters 1 524288/524288 52428/52428
delay_access 1 allow speedcontrol

# Final deny
http_access deny all

```

---

## 9. Apply Configuration and Test

### Restart Squid

```bash
# Validate config syntax
sudo squid -k parse

# Restart service
sudo systemctl restart squid

```

### Configure Client Browser

1. **Firefox Settings** → **Network Settings**
2. Select **Manual proxy configuration**
3. **HTTP Proxy**: `[your-server-ip]`
4. **Port**: `3128`
5. Check **"Use this proxy server for all protocols"**

### Test Restrictions

```bash
# Test blocked site
curl -x <http://192.168.1.10:3128> <http://facebook.com>

# Test allowed site
curl -x <http://192.168.1.10:3128> <http://google.com>

# Test file download
curl -x <http://192.168.1.10:3128> -O <http://example.com/test.mp4>

```

---

## 10. Key Commands Reference

| Task | Command |
| --- | --- |
| Install Squid | `dnf install squid` |
| Start service | `systemctl enable --now squid` |
| Validate config | `squid -k parse` |
| Restart | `systemctl restart squid` |
| View logs | `tail -f /var/log/squid/access.log` |
| Test proxy | `curl -x <http://IP:3128> <http://site.com`> |

---

## 11. Best Practices & Troubleshooting

### ✅ Do

- **Test rules incrementally** (add one ACL at a time)
- **Monitor logs**: `/var/log/squid/access.log`
- **Use regex carefully** (test with `grep -E`)
- **Backup config** before major changes

### ❌ Don't

- **Forget `http_access deny all`** at end
- **Mix allow/deny order** (causes unexpected behavior)
- **Block too aggressively** (breaks legitimate sites)

### Common Issues

- **"Access Denied" for all sites**:
Check rule order—`deny all` must be last
- **Blocked sites still accessible**:
Verify regex syntax and file paths
- **Squid won't start**:
Validate config: `sudo squid -k parse`
- **Slow performance**:
Check cache size settings in `/etc/squid/squid.conf`

### Log Analysis

```bash
# View blocked requests
grep TCP_DENIED /var/log/squid/access.log

# View bandwidth usage
tail -f /var/log/squid/access.log | awk '{print $3, $2}'

```

---

## Summary Workflow

```bash
# 1. Install
dnf install squid -y
systemctl enable --now squid

# 2. Configure firewall
firewall-cmd --permanent --add-port=3128/tcp

# 3. Create block lists
echo "\\.facebook\\.com" > /etc/squid/blocksites
echo "\\.mp4$" > /etc/squid/blockfiles.acl

# 4. Edit /etc/squid/squid.conf (add ACLs + http_access rules)

# 5. Validate and restart
squid -k parse
systemctl restart squid

# 6. Configure client browser proxy settings

```

> 💡 Golden Rule:
> 
> 
> **ACL order is everything**! Always:
> 
> - Place **specific rules first**
> - End with **`http_access deny all`**
> - **Test incrementally**—don't add all rules at once!