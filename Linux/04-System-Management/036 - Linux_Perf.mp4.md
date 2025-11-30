# 036: Resource Monitoring, Management & Troubleshooting in Linux


## 1. CPU and Memory Monitoring

### `top` - Real-time Process Monitor

```bash
top                    # Interactive process viewer
top -p <PID>           # Monitor specific process

```

- **Key columns**: `%CPU`, `%MEM`, `PID`, `COMMAND`
- **Interactive commands**:
    - `k` → Kill process
    - `r` → Renice process
    - `q` → Quit

### `free` - Memory Usage

```bash
free                   # Show memory in KB
free -m                # Show in MB
free -g                # Show in GB
free -s 3              # Update every 3 seconds
watch -d -n 5 free     # Highlight changes every 5 seconds

```

- **Columns**: `total`, `used`, `free`, `shared`, `buff/cache`, `available`
- **Critical metric**: `available` (not `free`) shows usable memory

---

## 2. I/O and CPU Detailed Monitoring

### `iostat` - I/O Statistics

```bash
# Install sysstat package first
sudo yum install sysstat -y

iostat                 # CPU + disk I/O summary
iostat -d 2 8          # Disk stats every 2s, 8 times

```

- **Key metrics**:
    - `%util` → Device utilization (100% = bottleneck)
    - `await` → Average I/O wait time (ms)

### `mpstat` - Per-CPU Statistics

```bash
mpstat                 # Average across all CPUs
mpstat -P ALL          # Per-CPU breakdown

```

- **Identify**: CPU imbalances or single-core bottlenecks

### `sar` - System Activity Reporter

```bash
sar 2 3                # CPU stats every 2s, 3 times
sar -r 5 5             # Memory stats (-r = RAM)
sar -d                 # Disk I/O stats (-d = devices)

```

- **Historical data**: `/var/log/sa/` (requires `sysstat` service enabled)

---

## 3. Advanced Monitoring Tools

| Tool | Purpose | Install Command |
| --- | --- | --- |
| `htop` | Enhanced `top` (color, tree view) | `sudo yum install htop` |
| `btop` | Modern resource monitor (graphs) | `sudo dnf install btop` |
| `nmon` | AIX-style monitor (RHEL: `sudo yum install nmon`) | `nmon` |

> 💡 Note: ztop is not standard—likely a typo for htop/btop.
> 

---

## 4. File and Process Troubleshooting

### `lsof` - List Open Files

```bash
lsof                   # All open files (files, sockets, pipes)
lsof | head            # First 10 entries
lsof -p <PID>          # Files opened by specific process
lsof -u zybi           # Files opened by user "zybi"

```

### Network Port Monitoring

```bash
lsof -i :80            # Processes using port 80
lsof -i :22            # SSH connections
lsof -i TCP            # All TCP connections

```

### Filesystem Monitoring

```bash
lsof /test/            # Processes using /test directory
lsof /var/log/*        # Processes using log files

```

### `fuser` - Identify Processes Using Files

```bash
fuser /test/           # Show PIDs using /test
fuser -m -u /test/     # Show PIDs + usernames
fuser -v /test/        # Verbose output (like lsof)

```

### Kill Processes Using Filesystem

```bash
fuser -m -k -u /test/  # Kill all processes using /test
# -k = kill, -u = show usernames

```

> ⚠️ Warning: fuser -k terminates processes immediately—use cautiously!
> 

---

## 5. Practical Troubleshooting Scenarios

### Scenario 1: "Device busy" error when unmounting

```bash
# Find processes using the mount
lsof /mnt/data
fuser -v /mnt/data

# Kill processes (if safe)
fuser -k /mnt/data
umount /mnt/data

```

### Scenario 2: High I/O wait (`%wa` in `top`)

```bash
# Identify busy disk
iostat -x 1

# Find processes causing I/O
iotop                  # Install with: yum install iotop
# OR
lsof +D /var/log       # Check log directory

```

### Scenario 3: Memory leak suspicion

```bash
# Monitor memory over time
watch -n 5 free -m

# Check top memory consumers
ps aux --sort=-%mem | head

```

---

## 6. Key Commands Reference

| Task | Command |
| --- | --- |
| Real-time processes | `top`, `htop` |
| Memory usage | `free -m`, `watch -n 2 free` |
| Disk I/O stats | `iostat -d 2`, `iotop` |
| Per-CPU usage | `mpstat -P ALL` |
| Historical data | `sar -u 1 3` (CPU), `sar -r` (RAM) |
| Open files | `lsof /path`, `lsof -i :port` |
| Kill file users | `fuser -k /path` |
| Process by name | `ps -C bash`, `pidof nginx` |

---

## 7. Best Practices & Pro Tips

### ✅ Do

- **Use `available` memory** (not `free`) for capacity planning
- **Monitor `%util` in `iostat`**—sustained >80% indicates I/O bottleneck
- **Combine tools**: `lsof` + `fuser` for unmount issues
- **Enable `sysstat` service** for historical `sar` data:
    
    ```bash
    sudo systemctl enable --now sysstat
    
    ```
    

### ❌ Don't

- **Kill processes blindly** with `fuser -k`—verify first!
- **Ignore `buff/cache`** in `free`—Linux uses it for performance
- **Use `top` alone**—supplement with `iostat`/`mpstat` for full picture

### Pro Workflow for High Load

1. `top` → Identify high CPU/memory process
2. `iostat -x 1` → Check for I/O bottleneck
3. `lsof -p <PID>` → See what files/process is using
4. `strace -p <PID>` → Trace system calls (advanced)

---

## Summary Cheat Sheet

```bash
# CPU/Memory
top
free -m
mpstat -P ALL

# I/O
iostat -d 2 5
iotop

# Open Files
lsof /path
lsof -i :80
fuser -v /path

# Kill File Users
fuser -k /path

# Historical Data
sar -u 1 3    # CPU
sar -r        # Memory

```

> 💡 Golden Rule:
> 
> 
> **Correlate metrics**—high load could be CPU, memory, OR I/O!
> 
> Always:
> 
> 1. Check `top` → 2. Verify with `iostat`/`mpstat` → 3. Drill down with `lsof`/`fuser`