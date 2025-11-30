# 051: MySQL Server Configuration 

---

## 1. Prerequisites and Installation

### Add User to sudo Group

```bash
# Ensure your user can run sudo commands
sudo usermod -aG wheel zybi
# OR for RHEL:
sudo usermod -aG sudo zybi

```

> 💡 Note: The user running MySQL commands needs sudo privileges for service management.
> 

### Install MySQL

```bash
# Enable repositories
sudo dnf repolist all

# Install MySQL server
sudo dnf install -y @mysql

# Enable and start service
sudo systemctl enable --now mysqld.service
sudo systemctl status mysqld.service

```

> ⚠️ Service Name:
> 
> 
> In RHEL 8, the service is **`mysqld`** (not `mysql`)
> 

---

## 2. Secure MySQL Installation

### Run Security Script

```bash
sudo mysql_secure_installation

```

**Interactive Prompts**:

```
Enter current password for root (enter for none): [Press Enter]
Set root password? [Y/n] y
New password: ********
Re-enter new password: ********
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables? [Y/n] y

```

> 🔍 Security Settings Explained:
> 
> - **Root password**: Required for all MySQL operations
> - **Anonymous users**: Removed for security
> - **Remote root login**: Disabled (only localhost allowed)
> - **Test database**: Removed (potential attack vector)

---

## 3. Create Database and User

### Login as Root

```bash
mysql -u root -p

```

### Create Database and User

```sql
-- Create database
CREATE DATABASE shahzaib;

-- Create user (FIXED SYNTAX)
CREATE USER 'shahzaib'@'localhost' IDENTIFIED BY 'P@$$w0rd';

-- Grant privileges
GRANT ALL PRIVILEGES ON shahzaib.* TO 'shahzaib'@'localhost';

-- Reload privileges
FLUSH PRIVILEGES;

-- Exit
EXIT;

```

---

## 4. Test New User Login

### Connect with New User

```bash
mysql -u shahzaib -p

```

### Verify Access

```sql
-- Show available databases
SHOW DATABASES;

-- Get help
\\h

-- Cancel current command
\\c

-- Exit
EXIT;

```

> 💡 User Permissions:
> 
> 
> The user `shahzaib` can only access the `shahzaib` database (not `mysql`, `information_schema`, etc.)
> 

---

## 5. Key Commands Reference

| Task | Command |
| --- | --- |
| Install MySQL | `sudo dnf install -y @mysql` |
| Start service | `sudo systemctl enable --now mysqld` |
| Secure install | `sudo mysql_secure_installation` |
| Login as root | `mysql -u root -p` |
| Create database | `CREATE DATABASE dbname;` |
| Create user | `CREATE USER 'user'@'host' IDENTIFIED BY 'password';` |
| Grant privileges | `GRANT ALL PRIVILEGES ON db.* TO 'user'@'host';` |
| Reload privileges | `FLUSH PRIVILEGES;` |

---

## 6. Web Management with phpMyAdmin (Optional)

### Install phpMyAdmin

```bash
# Enable EPEL repository first
sudo dnf install -y <https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm>

# Install phpMyAdmin
sudo dnf install -y phpMyAdmin

# Configure Apache (if using Apache)
sudo vim /etc/httpd/conf.d/phpMyAdmin.conf
# Uncomment lines for 127.0.0.1 access

# Restart Apache
sudo systemctl restart httpd

```

### Access phpMyAdmin

- URL: `http://<server-ip>/phpmyadmin`
- Login with MySQL credentials (`shahzaib`/`P@$$w0rd`)

> ⚠️ Security Warning:
> 
> 
> **Never expose phpMyAdmin to public internet** without additional security (HTTPS, IP restrictions, 2FA).
> 

---

## 7. Best Practices & Troubleshooting

### ✅ Do

- **Use strong passwords** for MySQL users
- **Limit user privileges** to only what's needed
- **Regularly backup databases**:
    
    ```bash
    mysqldump -u root -p shahzaib > shahzaib_backup.sql
    
    ```
    
- **Keep MySQL updated**: `sudo dnf update mysql-server`

### ❌ Don't

- **Use root for applications** (create dedicated users)
- **Store passwords in scripts** (use `.my.cnf` with proper permissions)
- **Skip `mysql_secure_installation`**

### Common Issues

- **"Access denied for user"**:
Verify user syntax: `'user'@'host'` (quotes and @ placement matter)
- **"Can't connect to local MySQL server"**:
Check if service is running: `sudo systemctl status mysqld`
- **"Unknown database"**:
Verify database name: `SHOW DATABASES;`

### MySQL Configuration File

```bash
# Main config file
/etc/my.cnf

# Additional configs
/etc/my.cnf.d/

```

---

## Summary Workflow

```bash
# 1. Install
sudo dnf install -y @mysql
sudo systemctl enable --now mysqld

# 2. Secure
sudo mysql_secure_installation

# 3. Create user
mysql -u root -p
CREATE DATABASE shahzaib;
CREATE USER 'shahzaib'@'localhost' IDENTIFIED BY 'P@$$w0rd';
GRANT ALL PRIVILEGES ON shahzaib.* TO 'shahzaib'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# 4. Test
mysql -u shahzaib -p
SHOW DATABASES;

```

> 💡 Golden Rule:
> 
> 
> **Never use root for applications**! Always:
> 
> - Create **dedicated database users**
> - Grant **minimum required privileges**
> - Run `mysql_secure_installation` immediately after install