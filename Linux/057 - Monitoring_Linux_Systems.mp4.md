# 057: Prometheus & Grafana Configuration | Linux Monitoring


## 1. Understanding the Monitoring Stack

### Core Components

| Component | Purpose |
| --- | --- |
| **Prometheus** | Time-series database + metrics collector |
| **Grafana** | Visualization and dashboarding |
| **Node Exporter** | Exports system metrics to Prometheus |
| **Alertmanager** | Handles alerts from Prometheus |

### Why Prometheus over InfluxDB?

- **Native Kubernetes support**
- **Powerful query language (PromQL)**
- **Built-in alerting**
- **Pull-based architecture** (more secure)

> 💡 Key Difference:
> 
> - **InfluxDB**: Push-based (Telegraf → InfluxDB)
> - **Prometheus**: Pull-based (Prometheus scrapes exporters)

---

## 2. Installing and Configuring Prometheus

### Create Service User

```bash
sudo useradd --no-create-home --shell /bin/false prometheus

```

### Download and Install Prometheus

```bash
# Download latest version
cd /tmp
wget <https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz>
tar xvf prometheus-2.47.1.linux-amd64.tar.gz

# Install binaries
sudo cp prometheus-2.47.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.47.1.linux-amd64/promtool /usr/local/bin/

# Create config directories
sudo mkdir /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

```

### Configure Prometheus

```bash
sudo vim /etc/prometheus/prometheus.yml

```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

```

### Create Systemd Service

```bash
sudo vim /etc/systemd/system/prometheus.service

```

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \\
    --config.file /etc/prometheus/prometheus.yml \\
    --storage.tsdb.path /var/lib/prometheus/ \\
    --web.console.templates=/etc/prometheus/consoles \\
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target

```

### Start Prometheus

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus

```

---

## 3. Installing Node Exporter

### Download and Install

```bash
cd /tmp
wget <https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz>
tar xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/

```

### Create Systemd Service

```bash
sudo vim /etc/systemd/system/node_exporter.service

```

```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

```

### Start Node Exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter

```

> 🔍 Verify:
> 
> 
> ```bash
> curl <http://localhost:9100/metrics> | head
> 
> ```
> 

---

## 4. Installing and Configuring Grafana

### Install Grafana

```bash
# Add repository
sudo tee /etc/yum.repos.d/grafana.repo <<EOF
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

# Install
sudo dnf install grafana -y

```

### Start Grafana

```bash
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server

```

### Initial Setup

1. **Access Grafana**: `http://<server-ip>:3000`
2. **Login**: `admin` / `admin`
3. **Set new password**

---

## 5. Configuring Grafana Data Source

### Add Prometheus Data Source

1. **Configuration** → **Data Sources** → **Add data source**
2. **Select**: **Prometheus**
3. **Configure**:
    - **URL**: `http://localhost:9090`
    - **Access**: `Server (default)`
4. **Click**: **Save & Test**

> 💡 Note:
> 
> 
> Use `localhost:9090` if Grafana and Prometheus are on same server
> 

---

## 6. Creating Dashboards

### Import Pre-built Dashboard

1. **Create** → **Import**
2. **Enter ID**: `1860` (Node Exporter Full)
3. **Select Prometheus data source**
4. **Import**

### Build Custom Dashboard

1. **Create** → **Dashboard** → **Add new panel**
2. **Query**:
    
    ```
    # CPU Usage
    100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    
    # Memory Usage
    (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
    
    # Disk Usage
    100 - ((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes)
    
    ```
    
3. **Visualization**: Time series, gauge, or stat
4. **Save dashboard**

---

## 7. Setting Up Alerts

### Create Alert Rule

1. **Alerting** → **Alert rules** → **New alert rule**
2. **Configure**:
    - **Query**: `node_load1 > 2`
    - **Condition**: `IS ABOVE 2`
    - **Evaluation group**: `1m`
    - **For**: `5m`
3. **Add notification**: Email, Slack, etc.

### Configure Alertmanager (Optional)

```bash
# Install Alertmanager
wget <https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz>
# Configure /etc/alertmanager/alertmanager.yml
# Set in prometheus.yml: alerting -> alertmanagers

```

---

## 8. Key Commands Reference

| Task | Command |
| --- | --- |
| Install Prometheus | Manual download + systemd |
| Install Node Exporter | Manual download + systemd |
| Install Grafana | `dnf install grafana` |
| Start services | `systemctl enable --now [service]` |
| Test Node Exporter | `curl <http://localhost:9100/metrics`> |
| Access Grafana | `http://IP:3000` |
| Prometheus queries | Use PromQL in Grafana |

---

## 9. Best Practices & Troubleshooting

### ✅ Do

- **Use separate users** for Prometheus services
- **Monitor Prometheus itself** (scrape Prometheus job)
- **Set retention policies**: `-storage.tsdb.retention.time=30d`
- **Use HTTPS** for production Grafana

### ❌ Don't

- **Run as root** (security risk)
- **Forget firewall rules**: `firewall-cmd --add-port={9090,9100,3000}/tcp`
- **Skip backup** of Prometheus data (`/var/lib/prometheus`)

### Common Issues

- **"No data" in Grafana**:
Check Prometheus targets: `http://localhost:9090/targets`
- **Node Exporter not scraping**:
Verify service status: `systemctl status node_exporter`
- **Grafana can't connect to Prometheus**:
Check network/firewall: `telnet localhost 9090`

### Security Hardening

```bash
# Firewall rules
sudo firewall-cmd --permanent --add-port=9090/tcp  # Prometheus
sudo firewall-cmd --permanent --add-port=9100/tcp  # Node Exporter
sudo firewall-cmd --permanent --add-port=3000/tcp  # Grafana
sudo firewall-cmd --reload

# Grafana HTTPS (in /etc/grafana/grafana.ini)
[server]
protocol = https
cert_file = /path/to/cert.pem
cert_key = /path/to/key.pem

```

---

## Summary Workflow

```bash
# 1. Install Prometheus
# Download → Configure → Systemd service

# 2. Install Node Exporter
# Download → Systemd service

# 3. Install Grafana
dnf install grafana -y
systemctl enable --now grafana-server

# 4. Configure Grafana
# Browser: <http://IP:3000> → Add Prometheus data source

# 5. Create dashboards
# Import ID 1860 or build custom with PromQL

```

> 💡 Golden Rule:
> 
> 
> **Prometheus pulls, doesn't push**! Always:
> 
> - Verify **targets are up** in Prometheus UI
> - Use **PromQL** for powerful queries
> - **Monitor your monitor** (scrape Prometheus itself)