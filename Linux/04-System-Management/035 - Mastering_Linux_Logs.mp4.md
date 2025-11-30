# 035: System Logging & Log Management in Linux (RHEL)

## 1. Understanding Linux Logging

### Why Logs Matter

- **Troubleshooting**: Identify errors, warnings, and system events
- **Security Auditing**: Track unauthorized access attempts
- **Performance Monitoring**: Analyze resource usage patterns
- **Compliance**: Meet regulatory requirements (HIPAA, PCI-DSS)

### Default Log Locations

```bash
ls -l /var/log

```

Key files:

- `/var/log/messages` - General system messages
- `/var/log/secure` - Authentication logs (SSH, sudo)
- `/var/log/maillog` - Mail server logs
- `/var/log/cron` - Cron job logs
- `/var/log/dmesg` - Kernel ring buffer messages

### User Session Logs

```bash
who                 # Currently logged-in users
last                # Login history
last reboot         # System reboot history
lastlog             # Last login for all users

```

---

## 2. The rsyslog Daemon

### Core Functionality

- **Listens** for log messages from applications/kernel
- **Filters** messages based on facility/priority rules
- **Stores** messages in specified files or forwards to remote servers

### Configuration Files

- **Main config**: `/etc/rsyslog.conf`
- **Modular configs**: `/etc/rsyslog.d/*.conf`

---

## 3. rsyslog Configuration Syntax

### Rule Structure

```
facility.priority    action

```

### Facilities (Message Sources)

| Facility | Purpose |
| --- | --- |
| `auth`/`authpriv` | Security/authorization messages |
| `kern` | Kernel messages |
| `mail` | Mail subsystem |
| `cron` | Cron daemon |
| `daemon` | System daemons |
| `user` | User-level applications |
| `local0-local7` | Custom/local use |

### Priorities (Severity Levels)

| Priority | Description |
| --- | --- |
| `debug` | Debug information |
| `info` | General information |
| `notice` | Normal but significant |
| `warning` | Warning conditions |
| `err` | Error conditions |
| `crit` | Critical conditions |
| `alert` | Immediate action required |
| `emerg` | System unusable |

### Priority Modifiers

- = All priorities
- `.` = This priority and higher
- `.=`  = Only this priority
- `.!` = Exclude this priority

### Examples

```
# Log all cron messages
cron.*                          /var/log/cron

# Log mail warnings and higher
mail.warning                    /var/log/mail.warn

# Log ONLY mail info messages
mail.=info                      /var/log/mail.info

```

---

## 4. Creating Custom Log Rules

### Step-by-Step Setup

```bash
# Edit rsyslog config
sudo vi /etc/rsyslog.conf

# Add custom rules at the end
local4.crit                     /var/log/local4crit.log
local4.=info                    /var/log/local4info.log

# Restart rsyslog
sudo systemctl restart rsyslog

```

### Test Custom Logging

```bash
# Generate test messages
logger -p local4.info "This is an info message from local4"
logger -p local4.crit "This is a critical message from local4"

# Verify logs
cat /var/log/local4info.log
cat /var/log/local4crit.log

```

> 💡 Note: logger is a command-line tool to send messages to syslog.
> 

---

## 5. Log Rotation with logrotate

### Why Rotate Logs?

- Prevent disk space exhaustion
- Archive old logs for historical analysis
- Compress logs to save space

### Configuration Files

- **Main config**: `/etc/logrotate.conf`
- **Service configs**: `/etc/logrotate.d/`

### Default logrotate.conf Structure

```
# Global settings
weekly
rotate 4
create
compress
include /etc/logrotate.d

# Service-specific rules
/var/log/messages {
    weekly
    rotate 4
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /usr/bin/systemctl kill -s HUP rsyslog.service >/dev/null 2>&1 || true
    endscript
}

```

### Key Directives

| Directive | Purpose |
| --- | --- |
| `weekly`/`daily`/`monthly` | Rotation frequency |
| `rotate N` | Keep N archived logs |
| `compress` | Gzip compress rotated logs |
| `delaycompress` | Compress previous log (not current) |
| `missingok` | Don't error if log missing |
| `notifempty` | Don't rotate empty logs |
| `postrotate`/`endscript` | Commands after rotation |

---

## 6. Advanced Logging Scenarios

### Remote Log Forwarding

```
# In /etc/rsyslog.conf
# Forward all logs to remote server
*.* @@192.168.1.100:514    # TCP
# *.* @192.168.1.100:514   # UDP

```

### Log Filtering by Content

```
# In /etc/rsyslog.d/10-filter.conf
if $msg contains 'ERROR' then /var/log/errors.log
& stop    # Don't process further

```

### Structured Logging (JSON)

```
# Enable JSON template
template(name="JsonFormat" type="list") {
    constant(value="{")
    constant(value="\\"timestamp\\":\\"")     property(name="timereported" dateFormat="rfc3339")
    constant(value="\\",\\"message\\":\\"")    property(name="msg" format="json")
    constant(value="\\"}")
}

# Apply to specific facility
local5.* action(type="omfile" file="/var/log/app.json" template="JsonFormat")

```

---

## 7. Key Commands Reference

| Task | Command |
| --- | --- |
| View system logs | `tail -f /var/log/messages` |
| Check rsyslog status | `systemctl status rsyslog` |
| Test log generation | `logger -p facility.priority "message"` |
| View log rotation config | `cat /etc/logrotate.conf` |
| Manually rotate logs | `logrotate -f /etc/logrotate.conf` |
| Check disk usage | `du -sh /var/log` |

---

## 8. Best Practices & Troubleshooting

### ✅ Do

- **Monitor log sizes**: `du -sh /var/log/*`
- **Set up log rotation**: Prevent disk exhaustion
- **Use remote logging**: Centralize logs for security
- **Restrict log permissions**: `chmod 600 /var/log/secure`

### ❌ Don't

- **Edit logs manually**: Breaks integrity for auditing
- **Disable logging**: Critical for security
- **Ignore rotation**: Leads to disk full errors

### Common Issues

- **"No space left on device"**:
    
    ```bash
    # Find large logs
    find /var/log -type f -size +100M -exec ls -lh {} \\;
    
    ```
    
- **rsyslog not starting**:
    
    ```bash
    # Check config syntax
    rsyslogd -N1
    
    ```
    
- **Missing logs**:
Verify application is configured to use syslog (not writing directly to files)

---

## Summary Workflow

### Custom Logging Setup

```bash
# 1. Add rule to /etc/rsyslog.conf
echo "local4.* /var/log/myapp.log" | sudo tee -a /etc/rsyslog.conf

# 2. Restart service
sudo systemctl restart rsyslog

# 3. Test
logger -p local4.info "Test message"

# 4. Verify
tail /var/log/myapp.log

```

### Log Rotation Setup

```bash
# Create custom rotation
sudo vi /etc/logrotate.d/myapp

```

```
/var/log/myapp.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}

```

> 💡 Golden Rule:
> 
> 
> **Logs are your forensic evidence**. Always:
> 
> - Keep logs **secure and unmodified**
> - **Rotate** to prevent disk issues
> - **Centralize** for security monitoring!