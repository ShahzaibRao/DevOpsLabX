# 050: Apache HTTP/HTTPS Server Configuration | Hosting Multiple Websites with SSL
---

## 1. Basic Apache Installation and Configuration

### Install and Configure Basic Web Server

```bash
# Install Apache
sudo dnf install httpd -y

# Create basic index page
sudo rm -rf /var/www/html/*
echo "Shuru ALLAH pak k pak naam se" | sudo tee /var/www/html/index.html

# Enable and start service
sudo systemctl enable --now httpd
sudo systemctl status httpd

```

### Configure Firewall for HTTP

```bash
# Allow HTTP traffic
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

```

### Test Basic Web Server

```bash
curl localhost
curl http://<server-ip>

```

> 💡 Note: By default, Apache serves content on port 80 (HTTP).
> 

---

## 2. Hosting Multiple Websites with SSL

### Set Static Hostname

```bash
sudo hostnamectl set-hostname www.shahzaib.com
# Logout and login to apply

```

### Install Required Packages

```bash
sudo dnf install -y httpd openssl mod_ssl

```

---

## 3. Generate Self-Signed SSL Certificate

### Create Certificate

```bash
# Generate self-signed certificate (valid 365 days)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \\
  -keyout /etc/pki/tls/private/server.key \\
  -out /etc/pki/tls/certs/server.crt

```

> ⚠️ During certificate creation:
> 
> - **Common Name (CN)**: Enter your domain (e.g., `www.shahzaib.com`)
> - Other fields can be left blank or filled as needed

### Verify Certificate Files

```bash
ls -l /etc/pki/tls/certs/server.crt
ls -l /etc/pki/tls/private/server.key

```

---

## 4. Configure Virtual Hosts with SSL

### Create Virtual Host Configuration

```bash
sudo vim /etc/httpd/conf.d/vhosts.conf

```

```
<VirtualHost *:443>
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/server.crt
    SSLCertificateKeyFile /etc/pki/tls/private/server.key
    ServerName www.shahzaib.com
    DocumentRoot /var/www/html/website1
</VirtualHost>

<VirtualHost *:443>
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/server.crt
    SSLCertificateKeyFile /etc/pki/tls/private/server.key
    ServerName www.shahzaib.net
    DocumentRoot /var/www/html/website2
</VirtualHost>

```

> 🔍 Critical Notes:
> 
> - **File location**: Use `/etc/httpd/conf.d/` (not `httpd.conf`)
> - **Same certificate**: Both sites use the same self-signed cert (for lab only)
> - **DocumentRoot**: Must exist before starting Apache

### Validate Configuration

```bash
sudo httpd -t
# OR
sudo apachectl configtest

```

---

## 5. Create Website Directories

### Set Up Document Roots

```bash
sudo mkdir -p /var/www/html/website1 /var/www/html/website2

# Set proper SELinux contexts
sudo restorecon -R /var/www/html/

# Verify contexts
ls -ldZ /var/www/html/website1
ls -ldZ /var/www/html/website2

```

### Add Website Content

```bash
# Website 1
echo "<h1>Welcome to Shahzaib.com</h1>" | sudo tee /var/www/html/website1/index.html

# Website 2
echo "<h1>Welcome to Shahzaib.net</h1>" | sudo tee /var/www/html/website2/index.html

```

---

## 6. Configure Firewall for HTTPS

```bash
# Allow HTTPS traffic
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

```

---

## 7. Configure Local DNS Resolution

### Edit Hosts File (Client-Side)

```bash
# On client machine (or server for testing)
sudo vim /etc/hosts

```

```
192.168.100.5 www.shahzaib.com
192.168.100.5 www.shahzaib.net

```

> 💡 Note: Replace 192.168.100.5 with your server's actual IP address.
> 

---

## 8. Start and Test Apache

### Restart Apache

```bash
sudo systemctl restart httpd
sudo systemctl status httpd

```

### Test Websites

1. Open browser and navigate to:
    - `https://www.shahzaib.com`
    - `https://www.shahzaib.net`
2. **Accept security warning** (self-signed certificate)
3. Verify correct content loads for each site

---

## 9. Key Commands Reference

| Task | Command |
| --- | --- |
| Install Apache | `dnf install httpd mod_ssl -y` |
| Generate SSL cert | `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout key -out crt` |
| Validate config | `httpd -t` |
| Create vhost file | `/etc/httpd/conf.d/vhosts.conf` |
| Set SELinux context | `restorecon -R /var/www/html/` |
| Firewall HTTPS | `firewall-cmd --permanent --add-service=https` |
| Test config | `curl -k <https://domain`> |

---

## 10. Best Practices & Troubleshooting

### ✅ Do

- **Use separate certificates** for production (not self-signed)
- **Set proper SELinux contexts** on web directories
- **Validate config** before restarting Apache (`httpd -t`)
- **Use FQDNs** in SSL certificates

### ❌ Don't

- **Edit `/etc/hosts` on server** for production (use real DNS)
- **Use same certificate** for multiple domains in production
- **Ignore SELinux contexts** (causes "403 Forbidden" errors)

### Common Issues

- **"403 Forbidden"**:
Fix SELinux contexts: `sudo restorecon -R /var/www/html/`
- **"SSL_ERROR_BAD_CERT_DOMAIN"**:
Certificate CN doesn't match domain (expected with self-signed)
- **Apache won't start**:
Check config syntax: `sudo httpd -t`
- **Connection refused**:
Verify firewall: `sudo firewall-cmd --list-services | grep https`

### Production Recommendations

- **Use Let's Encrypt** for free trusted certificates:
    
    ```bash
    sudo dnf install certbot python3-certbot-apache
    sudo certbot --apache -d www.shahzaib.com
    
    ```
    
- **Separate document roots** with proper permissions:
    
    ```bash
    sudo chown -R apache:apache /var/www/html/website1
    sudo chmod -R 755 /var/www/html/website1
    
    ```
    

---

## Summary Workflow

```bash
# 1. Install
dnf install httpd mod_ssl -y

# 2. Generate SSL cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/pki/tls/private/server.key -out /etc/pki/tls/certs/server.crt

# 3. Create vhosts
vim /etc/httpd/conf.d/vhosts.conf

# 4. Create websites
mkdir -p /var/www/html/website{1,2}
echo "Content" > /var/www/html/website1/index.html

# 5. Configure firewall
firewall-cmd --permanent --add-service=https

# 6. Test
systemctl restart httpd
# Browser: <https://www.shahzaib.com>

```

> 💡 Golden Rule:
> 
> 
> **Always validate and test**! Use:
> 
> - `httpd -t` for config syntax
> - `restorecon -R` for SELinux contexts
> - Separate certificates for production domains