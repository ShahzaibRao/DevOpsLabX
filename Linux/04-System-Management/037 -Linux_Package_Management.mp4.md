# 037: Package Management in Linux


## 1. Overview of Package Management Systems

| Distribution | Package Format | Low-Level Tool | High-Level Tool |
| --- | --- | --- | --- |
| **Debian/Ubuntu** | `.deb` | `dpkg` | `apt`, `aptitude` |
| **RHEL/Fedora/CentOS** | `.rpm` | `rpm` | `yum` (RHEL 7), `dnf` (RHEL 8+) |

> 💡 Key Concept:
> 
> - **Low-level tools** (`dpkg`, `rpm`) handle single packages (no dependencies)
> - **High-level tools** (`apt`, `dnf`) resolve dependencies automatically

---

## 2. Debian/Ubuntu Package Management

### Low-Level: `dpkg`

```bash
# List installed packages
dpkg -l
dpkg -l | wc -l                 # Count installed packages
dpkg -l apache2                 # Search specific package

# Find which package owns a file
dpkg -S /usr/share/doc/rsync   # File path
dpkg -S rsync                   # Filename

# List files in a package
dpkg -L apache2

# Install/remove packages (no dependencies!)
sudo dpkg -i package.deb        # Install
sudo dpkg -r apache2            # Remove (keep config)
sudo dpkg -P apache2            # Purge (remove config too)

```

### High-Level: `apt`

```bash
# Update package lists
sudo apt update

# Install/remove packages
sudo apt install apache2
sudo apt remove apache2         # Keep config
sudo apt purge apache2          # Remove config

# Search packages
apt-cache search apache2
apt list --installed | grep apache

# Clean package cache
sudo apt clean                  # Remove all .deb files
ls /var/cache/apt/archives/     # Package cache location

```

### Advanced: `aptitude`

```bash
sudo aptitude update
sudo aptitude safe-upgrade      # Safe system upgrade
sudo aptitude install apache2
sudo aptitude search apache2
sudo aptitude purge apache2

# View repositories
cat /etc/apt/sources.list

```

### Adding Repositories

```bash
# Add repository
sudo add-apt-repository ppa:repository/name

# Add GPG key
wget -qO - <https://repo.example.com/key.gpg> | sudo apt-key add -

```

---

## 3. RHEL/Fedora Package Management

### Low-Level: `rpm`

```bash
# Query packages
rpm -qa                         # List all installed
rpm -qa | wc -l                 # Count packages
rpm -qa | grep samba            # Search packages
rpm -qi httpd                   # Package info

# Install/remove (no dependencies!)
sudo rpm -ivh package.rpm       # Install (-v=verbose, -h=progress)
sudo rpm -e httpd               # Remove
sudo rpm -Uvh package.rpm       # Upgrade

# Package database location
ls /var/lib/rpm                 # RPM database

# Extract RPM contents
rpm2cpio package.rpm | cpio -t   # List files in RPM
rpm2cpio package.rpm > package.cpio  # Convert to cpio archive

```

### High-Level: `dnf` (RHEL 8+) / `yum` (RHEL 7)

```bash
# Mount installation media (for offline installs)
sudo mount /dev/sr0 /mnt
cd /mnt/BaseOS/Packages

# Install with dependency resolution
sudo dnf install ypbind         # RHEL 8+
# sudo yum install ypbind       # RHEL 7

# List packages
dnf list                        # Available packages
dnf list installed              # Installed packages
dnf list httpd                  # Specific package

# Search packages
dnf search httpd                # By name/description
dnf provides /usr/share/man/man5/passwd.5.gz  # Find package owning file

# Group management
dnf grouplist
dnf groupinstall "Web Server"

# Repository management
ls /etc/yum.repos.d/            # Repository files
cat /etc/yum.conf               # Main config

```

### EPEL Repository (Extra Packages)

```bash
# Install EPEL (RHEL 8)
sudo dnf install <https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm>

# Verify
dnf repolist enabled | grep epel

```

---

## 4. Cross-Platform Package Conversion

### Using `alien`

```bash
# On Ubuntu: Convert RPM to DEB
sudo apt install alien
alien --to-deb package.rpm

# On CentOS: Convert DEB to RPM
sudo yum install alien
alien --to-rpm package.deb

```

> ⚠️ Warning:
> 
> 
> Converted packages may have **dependency issues**—use only as last resort!
> 

---

## 5. Compiling from Source Code

### Step-by-Step

```bash
# Install build tools
sudo dnf groupinstall "Development Tools"    # RHEL
# sudo apt install build-essential           # Debian

# Download and extract
tar -xvf package.tar.gz
cd package/

# Build and install
./configure    # Check dependencies, generate Makefile
make           # Compile code
sudo make install  # Install to /usr/local/

```

### Key Notes

- **`./configure`**: Creates `Makefile` (run `./configure --help` for options)
- **`make`**: Compiles source code
- **`make install`**: Copies binaries to system directories
- **Uninstall**: Usually no clean way—keep source directory for `make uninstall` (if supported)

---

## 6. Key Commands Reference

### Debian/Ubuntu

| Task | Command |
| --- | --- |
| Install package | `sudo apt install pkg` |
| Remove package | `sudo apt remove pkg` |
| Purge config | `sudo apt purge pkg` |
| Update lists | `sudo apt update` |
| Search packages | `apt search keyword` |
| Find file owner | `dpkg -S /path/to/file` |

### RHEL/Fedora

| Task | Command |
| --- | --- |
| Install package | `sudo dnf install pkg` |
| Remove package | `sudo dnf remove pkg` |
| List installed | `dnf list installed` |
| Search packages | `dnf search keyword` |
| Find file owner | `dnf provides /path/to/file` |
| Install group | `dnf groupinstall "Group Name"` |

---

## 7. Best Practices & Troubleshooting

### ✅ Do

- **Prefer high-level tools** (`apt`, `dnf`) over low-level (`dpkg`, `rpm`)
- **Update package lists** before installing: `apt update` / `dnf check-update`
- **Use groups** for related packages: `dnf groupinstall "Development Tools"`
- **Keep source directories** when compiling from source (for uninstall)

### ❌ Don't

- **Mix package managers** (e.g., `dnf` + manual `rpm` installs)
- **Ignore GPG key warnings**—verify repository authenticity
- **Compile from source unnecessarily**—use packages when available

### Common Issues

- **"Dependency not satisfied" with rpm**:
Use `dnf install package.rpm` instead of `rpm -ivh`
- **Broken packages**:
    
    ```bash
    # Debian
    sudo apt --fix-broken install
    
    # RHEL
    sudo dnf distro-sync
    
    ```
    
- **Full package cache**:
    
    ```bash
    sudo apt clean          # Debian
    sudo dnf clean all      # RHEL
    
    ```
    

---

## Summary Workflow

### Install Package

```bash
# Debian
apt update && apt install apache2

# RHEL
dnf install httpd

```

### Compile from Source

```bash
dnf groupinstall "Development Tools"
tar -xvf app.tar.gz
cd app/
./configure --prefix=/usr
make
sudo make install

```

> 💡 Golden Rule:
> 
> 
> **Packages > Source**. Always:
> 
> - Use **distribution packages** when available
> - Reserve **source compilation** for:
>     - Latest versions not in repos
>     - Custom compile options
>     - No package available