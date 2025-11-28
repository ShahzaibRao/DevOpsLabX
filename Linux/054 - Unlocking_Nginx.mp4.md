# 054: Nginx Web Server Configuration in Linux | HTTP/HTTPS
---

## 1. Understanding Nginx Architecture

### Core Components

| Section | Purpose |
| --- | --- |
| **Main** | Global settings (worker processes, user, PID) |
| **Events** | Connection handling (worker_connections) |
| **HTTP** | Web server configuration (server blocks, upstream) |
| **Stream** | TCP/UDP load balancing (non-HTTP traffic) |

### Key Directories (Debian/Ubuntu)

- **Main config**: `/etc/nginx/nginx.conf`
- **Site configs**: `/etc/nginx/sites-available/` → symlinked to `/etc/nginx/sites-enabled/`
- **Custom configs**: `/etc/nginx/conf.d/*.conf`
- **Web root**: `/var/www/html/`

> 💡 RHEL Note: RHEL uses /etc/nginx/conf.d/ directly (no sites-enabled)
> 

---

## 2. Basic Installation and Setup

### Install Nginx

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx -y

# RHEL/CentOS
sudo dnf install nginx -y

# Start and enable
sudo systemctl enable --now nginx
sudo systemctl status nginx

```

### Test Basic Installation

- Visit `http://<server-ip>` in browser
- Default page: `/var/www/html/index.html`

---

## 3. Hosting Multiple Websites (Virtual Hosts)

### Create Website Directories

```bash
# Cafe website
sudo mkdir -p /var/www/cafe
sudo git clone <https://github.com/codersgyan/Responsive-restaurant-website> /var/www/cafe

# Portfolio website
sudo mkdir -p /var/www/portfolio
sudo git clone <https://github.com/codersgyan/portfolio> /var/www/portfolio

# Dev website
sudo mkdir -p /var/www/dev
echo "Hello user" | sudo tee /var/www/dev/index.html

```

### Configure Virtual Hosts

### Cafe Website (`/etc/nginx/conf.d/cafe.conf`)

```
server {
    listen 80;
    root /var/www/cafe;
    server_name cafe.coderservergyan.com;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}

```

### Portfolio Website (`/etc/nginx/conf.d/portfolio.conf`)

```
server {
    listen 80;
    root /var/www/portfolio;
    server_name portfolio.coderservergyan.com;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}

```

### Dev Website with Authentication (`/etc/nginx/conf.d/dev.conf`)

```
server {
    listen 80;
    root /var/www/dev;
    server_name dev.coderservergyan.com;
    index index.html index.htm;

    location / {
        auth_basic "Development Site";
        auth_basic_user_file /etc/nginx/.htpasswd;
        try_files $uri $uri/ =404;
    }
}

```

---

## 4. Password Authentication Setup

### Create Password File

```bash
# Install apache2-utils for htpasswd
sudo apt install apache2-utils -y  # Ubuntu
# sudo dnf install httpd-tools -y  # RHEL

# Create user 'codersgyan'
sudo htpasswd -c /etc/nginx/.htpasswd codersgyan
# Enter password when prompted

# Verify
cat /etc/nginx/.htpasswd

```

> 🔍 Security Note:
> 
> - File permissions: `sudo chmod 640 /etc/nginx/.htpasswd`
> - Owner: `sudo chown root:nginx /etc/nginx/.htpasswd`

---

## 5. Nginx as Reverse Proxy

### Basic Reverse Proxy Configuration

```
server {
    listen 80;
    server_name app.coderservergyan.com;

    location / {
        proxy_pass <http://192.168.1.100:3000>;  # Backend app
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```

### Advanced Reverse Proxy (Load Balancing)

```
# Upstream block (in http section)
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

server {
    listen 80;
    server_name api.coderservergyan.com;

    location / {
        proxy_pass <http://backend>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

```

---

## 6. Essential Nginx Commands

| Task | Command |
| --- | --- |
| Test config syntax | `sudo nginx -t` |
| Reload config | `sudo systemctl reload nginx` |
| Show full config | `sudo nginx -T` |
| View logs | `sudo tail -f /var/log/nginx/access.log` |
| Create password file | `sudo htpasswd -c /etc/nginx/.htpasswd username` |

---

## 7. Client DNS Configuration

### For Testing (Local Machine)

```bash
# Edit hosts file
sudo vim /etc/hosts  # Linux/Mac
# C:\\Windows\\System32\\drivers\\etc\\hosts  # Windows

```

```
192.168.1.50 cafe.coderservergyan.com
192.168.1.50 portfolio.coderservergyan.com
192.168.1.50 dev.coderservergyan.com

```

> 💡 Production: Use real DNS records instead of hosts file
> 

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Test config** before reloading: `nginx -t`
- **Use separate config files** per site (`/etc/nginx/conf.d/site.conf`)
- **Set proper permissions**: `sudo chown -R www-data:www-data /var/www/`
- **Enable logging**: Check `/var/log/nginx/error.log` for issues

### ❌ Don't

- **Use `default_server` on multiple sites**
- **Forget `proxy_set_header`** in reverse proxy configs
- **Store passwords in world-readable files**

### Common Issues

- **"403 Forbidden"**:
Fix permissions: `sudo chmod -R 755 /var/www/site`
- **"404 Not Found"**:
Verify `root` path and `index` files exist
- **"502 Bad Gateway"**:
Check backend server is running and accessible
- **Config not applying**:
Always reload: `sudo systemctl reload nginx`

### Security Hardening

```
# Add to server block
server_tokens off;           # Hide Nginx version
client_max_body_size 10M;    # Limit upload size
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";

```

---

## Summary Workflow

```bash
# 1. Install
apt install nginx -y

# 2. Create websites
mkdir -p /var/www/{cafe,portfolio,dev}
git clone [URL] /var/www/cafe

# 3. Create configs in /etc/nginx/conf.d/
# cafe.conf, portfolio.conf, dev.conf

# 4. Set up authentication (if needed)
htpasswd -c /etc/nginx/.htpasswd username

# 5. Test and reload
nginx -t
systemctl reload nginx

# 6. Configure DNS/hosts file
# Browser: <http://cafe.coderservergyan.com>

```

> 💡 Golden Rule:
> 
> 
> **Always test with `nginx -t`** before reloading!
> 
> Use **separate config files** for each site, and **never edit the main nginx.conf** for site-specific settings.
>