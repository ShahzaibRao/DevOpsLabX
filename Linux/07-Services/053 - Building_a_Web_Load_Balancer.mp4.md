# 053: HAProxy Load Balancer Configuration on RHEL ---



## 1. Environment Setup

### Deploy 3 Machines

| Role | Hostname | IP Address |
| --- | --- | --- |
| **Load Balancer** | `haproxy-centos8` | `192.168.100.5` |
| **Web Server 1** | `nginx-node1` | `192.168.100.6` |
| **Web Server 2** | `nginx-node2` | `192.168.100.7` |

### Configure Hostnames

```bash
# On each machine
sudo hostnamectl set-hostname <hostname>
# Logout and login to apply

```

### Configure Hosts Files (All Machines)

```bash
sudo vim /etc/hosts

```

```
192.168.100.5 haproxy-centos8
192.168.100.6 nginx-node1
192.168.100.7 nginx-node2

```

> 💡 Verify Connectivity:
> 
> 
> ```bash
> ping haproxy-centos8
> ping nginx-node1
> 
> ```
> 

---

## 2. HAProxy Installation and Configuration

### Install HAProxy

```bash
# On HAProxy server
sudo dnf update -y
sudo dnf install -y haproxy

# Backup original config
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.backup

```

### Configure HAProxy

```bash
sudo vim /etc/haproxy/haproxy.cfg

```

### Replace with Minimal Configuration

```
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# Frontend Configuration
frontend http_balancer
    bind 192.168.100.5:80
    option http-server-close
    option forwardfor
    stats uri /haproxy?stats
    stats refresh 30s
    stats show-node
    stats show-desc HAProxy Load Balancer
    default_backend nginx-webservers

# Backend Configuration
backend nginx-webservers
    mode http
    balance roundrobin
    option httpchk HEAD / HTTP/1.1\\r\\nHost:\\ localhost
    server nginx-node1 192.168.100.6:80 check
    server nginx-node2 192.168.100.7:80 check

```

---

## 3. Configure Logging

### Enable UDP Logging in rsyslog

```bash
sudo vim /etc/rsyslog.conf

```

**Uncomment these lines**:

```
module(load="imudp")
input(type="imudp" port="514")

```

### Create HAProxy Log Configuration

```bash
sudo vim /etc/rsyslog.d/haproxy.conf

```

```
# HAProxy logging
local2.*    /var/log/haproxy.log

```

> 🔍 Note:
> 
> 
> Your original config had typos (`inof` → `info`, wrong paths). Using single log file is simpler.
> 

### Restart rsyslog

```bash
sudo systemctl restart rsyslog
sudo systemctl enable --now rsyslog

```

---

## 4. SELinux Configuration

### Enable HAProxy Network Access

```bash
# Check current boolean state
getsebool haproxy_connect_any

# Enable permanently
sudo setsebool -P haproxy_connect_any 1

# Verify
getsebool haproxy_connect_any

```

> 💡 Why this matters:
> 
> 
> SELinux blocks HAProxy from connecting to backend servers by default.
> 

---

## 5. Firewall Configuration

```bash
# Allow HTTP traffic
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

```

---

## 6. Configure Backend Nginx Servers

### On Both Nginx Nodes

```bash
# Install Nginx
sudo dnf install nginx -y

# Enable and start
sudo systemctl enable --now nginx
sudo systemctl status nginx

# Create unique index pages
# Node 1:
echo "Nginx Node1 - Welcome to first nginx-webserver" | sudo tee /usr/share/nginx/html/index.html

# Node 2:
echo "Nginx Node2 - Welcome to second nginx-webserver" | sudo tee /usr/share/nginx/html/index.html

# Configure firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload

```

---

## 7. Start HAProxy and Test

### Enable and Start HAProxy

```bash
sudo systemctl enable --now haproxy
sudo systemctl status haproxy

```

### Test Load Balancing

```bash
# Test from HAProxy server or client
curl <http://192.168.100.5>
curl <http://192.168.100.5>

# Should alternate between Node1 and Node2 responses

```

### Access HAProxy Stats

- **URL**: `http://192.168.100.5/haproxy?stats`
- **No authentication** (add `stats auth user:password` for production)

### Monitor Logs

```bash
# View access logs
sudo tail -f /var/log/haproxy.log

# Check for errors
sudo grep -i error /var/log/haproxy.log

```

---

## 8. Key Commands Reference

| Task | Command |
| --- | --- |
| Install HAProxy | `dnf install haproxy` |
| Validate config | `haproxy -c -f /etc/haproxy/haproxy.cfg` |
| Restart HAProxy | `systemctl restart haproxy` |
| View stats | `http://IP/haproxy?stats` |
| Monitor logs | `tail -f /var/log/haproxy.log` |
| Test backend | `curl -H "Host: localhost" <http://192.168.100.6`> |

---

## 9. Best Practices & Troubleshooting

### ✅ Do

- **Validate config** before restarting: `haproxy -c -f /etc/haproxy/haproxy.cfg`
- **Use unique content** on backend servers for testing
- **Monitor stats page** for server health
- **Enable SELinux booleans** instead of disabling SELinux

### ❌ Don't

- **Use identical content** on backend servers (can't verify load balancing)
- **Forget firewall rules** on backend servers
- **Skip SELinux configuration** (causes connection failures)

### Common Issues

- **"503 Service Unavailable"**:
Check backend server status in stats page → verify Nginx is running
- **Connection refused**:
Verify firewall on backend servers: `firewall-cmd --list-services | grep http`
- **SELinux denials**:
Check audit logs: `sudo ausearch -m avc -ts recent`
- **HAProxy won't start**:
Validate config syntax: `sudo haproxy -c -f /etc/haproxy/haproxy.cfg`

### Production Enhancements

```
# Add to frontend for HTTPS
bind 192.168.100.5:443 ssl crt /etc/ssl/certs/haproxy.pem

# Add authentication to stats
stats auth admin:SecurePassword123

# Health check improvement
option httpchk GET /health HTTP/1.1\\r\\nHost:\\ localhost

```

---

## Summary Workflow

```bash
# 1. Setup hosts files on all machines

# 2. Install and configure HAProxy
dnf install haproxy
# Edit /etc/haproxy/haproxy.cfg
setsebool -P haproxy_connect_any 1
firewall-cmd --permanent --add-port=80/tcp

# 3. Configure logging
# Edit /etc/rsyslog.conf + /etc/rsyslog.d/haproxy.conf

# 4. Install Nginx on backend servers
dnf install nginx
# Create unique index.html files

# 5. Test
curl <http://192.168.100.5>
# Browser: <http://192.168.100.5/haproxy?stats>

```

> 💡 Golden Rule:
> 
> 
> **Test incrementally**! Always:
> 
> - Verify **backend servers work individually** first
> - Check **HAProxy stats page** for server status
> - **Monitor logs** during testing
