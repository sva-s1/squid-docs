# Squid Syslog Forwarding: Quick Reference Guide

**For:** Quick command lookups and common tasks

---

## Table of Contents
1. [Configuration Files](#configuration-files)
1. [Service Management](#service-management)
1. [Testing Commands](#testing-commands)
1. [Monitoring Commands](#monitoring-commands)
1. [Common Modifications](#common-modifications)
1. [Troubleshooting Commands](#troubleshooting-commands)

---

## Configuration Files

### Important File Locations

| File Path | Purpose |
|-----------|---------|
| `/etc/squid/squid.conf` | Main Squid configuration |
| `/etc/rsyslog.d/30-squid-forward.conf` | Our custom syslog forwarding rules |
| `/var/log/messages` | Local system log (where Squid logs appear) |
| `/var/log/squid/cache.log` | Squid's internal diagnostic log |
| `/var/spool/squid/` | Squid cache directory |

---

### View Configurations

```bash
# View Squid's access_log configuration
grep access_log /etc/squid/squid.conf

# View rsyslog forwarding configuration
cat /etc/rsyslog.d/30-squid-forward.conf

# View full Squid config (commented lines too)
cat /etc/squid/squid.conf

# View only active Squid config (no comments)
grep -v "^#" /etc/squid/squid.conf | grep -v "^$"
```

---

### Edit Configurations

```bash
# Edit Squid config (with vi)
vi /etc/squid/squid.conf

# Edit Squid config (with nano - easier)
nano /etc/squid/squid.conf

# Edit rsyslog forwarding config
nano /etc/rsyslog.d/30-squid-forward.conf

# Backup config before editing
cp /etc/squid/squid.conf /etc/squid/squid.conf.backup.$(date +%Y%m%d)
```

---

## Service Management

### Start/Stop/Restart Services

```bash
# Restart Squid
systemctl restart squid

# Start Squid
systemctl start squid

# Stop Squid
systemctl stop squid

# Restart rsyslog
systemctl restart rsyslog

# Restart both (in correct order)
systemctl restart rsyslog && systemctl restart squid
```

---

### Check Service Status

```bash
# Check if Squid is running
systemctl status squid

# Check if rsyslog is running
systemctl status rsyslog

# Check both at once
systemctl status squid rsyslog

# Check if services are enabled to start at boot
systemctl is-enabled squid
systemctl is-enabled rsyslog
```

---

### Enable/Disable Auto-Start

```bash
# Enable Squid to start at boot
systemctl enable squid

# Disable Squid from starting at boot
systemctl disable squid

# Enable and start Squid in one command
systemctl enable --now squid
```

---

### Reload Configs Without Restarting

```bash
# Reload Squid config without full restart (minimal downtime)
squid -k reconfigure

# Or via systemctl
systemctl reload squid

# rsyslog requires restart (no reload option)
systemctl restart rsyslog
```

---

## Testing Commands

### Test Configuration Syntax

```bash
# Test Squid config for errors
squid -k parse

# Test rsyslog config for errors
rsyslogd -N1

# Test both
squid -k parse && rsyslogd -N1 && echo "Both configs OK!"
```

---

### Generate Test Traffic

```bash
# Single request through proxy
curl -x http://localhost:3128 http://www.example.com/ -I

# Multiple requests (10 times)
for i in {1..10}; do
  curl -x http://localhost:3128 http://www.example.com/ -I -s > /dev/null
  echo "Request $i sent"
done

# Request with output
curl -x http://localhost:3128 http://www.google.com/

# Request to HTTPS site
curl -x http://localhost:3128 https://www.google.com/ -I
```

---

### Test Network Connectivity

```bash
# Ping remote syslog server (replace with your IP)
ping -c 5 YOUR_SYSLOG_IP

# Test UDP port reachability
nc -zvu YOUR_SYSLOG_IP 10001

# Trace route to remote server
traceroute YOUR_SYSLOG_IP

# Check if packets are being sent (run for 15 seconds)
timeout 15 tcpdump -i any -nn 'udp and dst port 10001' -c 5

# Check packets with content visible
timeout 15 tcpdump -i any -nn -A 'udp and dst port 10001' -c 2
```

---

### Test Syslog Directly (Bypass Squid)

```bash
# Send test message via logger
logger -p local5.info "TEST MESSAGE FROM LOGGER"

# Send test and verify it appears in local log
logger -p local5.info "TEST $(date)" && sleep 2 && tail -1 /var/log/messages

# Send test and check if sent to remote (with tcpdump running)
logger -p local5.info "NETWORK TEST $(date)"
```

---

## Monitoring Commands

### View Logs

```bash
# View last 20 Squid logs
grep squid /var/log/messages | tail -20

# View last 5 Squid logs (most recent)
grep squid /var/log/messages | tail -5

# Watch logs in real-time (live)
tail -f /var/log/messages | grep --line-buffered squid

# View Squid's cache.log (internal diagnostics)
tail -20 /var/log/squid/cache.log

# View last 50 rsyslog journal entries
journalctl -u rsyslog -n 50

# View last 50 Squid journal entries
journalctl -u squid -n 50

# Follow journal logs in real-time
journalctl -u squid -f
```

---

### Search Logs

```bash
# Search for specific domain
grep squid /var/log/messages | grep "example.com"

# Search for HTTP errors (4xx, 5xx)
grep squid /var/log/messages | grep -E "TCP_.*/(4[0-9]{2}|5[0-9]{2})"

# Search for specific client IP
grep squid /var/log/messages | grep "192.168.1.100"

# Count total Squid log entries
grep -c squid /var/log/messages

# Show logs from last hour
grep "$(date -d '1 hour ago' '+%b %e %H'):" /var/log/messages | grep squid
```

---

### Monitor System Resources

```bash
# Check Squid CPU and memory usage
ps aux | grep squid

# Check rsyslog CPU and memory usage
ps aux | grep rsyslog

# Detailed process info
top -p $(pgrep squid)

# Check network connections (Squid)
ss -tnp | grep squid

# Check listening ports
ss -tlnp | grep -E '(squid|rsyslog|3128)'

# Check disk space (for cache and logs)
df -h /var/spool/squid /var/log

# Check Squid cache size
du -sh /var/spool/squid
```

---

### Statistics

```bash
# Count logs per hour (last 24 hours)
for hour in {0..23}; do
  echo -n "$(date -d "$hour hours ago" '+%b %e %H'):00 - "
  grep "$(date -d "$hour hours ago" '+%b %e %H'):" /var/log/messages | grep -c squid
done

# Top 10 most accessed domains
grep squid /var/log/messages | awk '{print $NF}' | sort | uniq -c | sort -rn | head -10

# Top 10 client IPs
grep squid /var/log/messages | awk '{print $9}' | sort | uniq -c | sort -rn | head -10

# Count HTTP status codes
grep squid /var/log/messages | grep -oE "TCP_[A-Z]+/[0-9]{3}" | cut -d'/' -f2 | sort | uniq -c
```

---

## Common Modifications

### Change Remote Syslog Server IP/Port

```bash
# Edit rsyslog config
nano /etc/rsyslog.d/30-squid-forward.conf

# Find this line:
# local5.* @OLD_IP:OLD_PORT;RFC5424Format

# Change to new IP/port:
# local5.* @NEW_IP:NEW_PORT;RFC5424Format

# Save and restart rsyslog
systemctl restart rsyslog
```

---

### Add Second Destination (Send to Multiple Servers)

```bash
# Edit rsyslog config
nano /etc/rsyslog.d/30-squid-forward.conf

# Add additional line:
local5.* @SYSLOG_SERVER_1:10001;RFC5424Format
local5.* @SYSLOG_SERVER_2:10001;RFC5424Format

# Save and restart rsyslog
systemctl restart rsyslog
```

---

### Change to TCP Instead of UDP

```bash
# Edit rsyslog config
nano /etc/rsyslog.d/30-squid-forward.conf

# Change @ to @@ (double @ = TCP):
# local5.* @@YOUR_SYSLOG_IP:10001;RFC5424Format

# Save and restart rsyslog
systemctl restart rsyslog
```

---

### Disable Remote Logging Temporarily

```bash
# Rename config to disable it
mv /etc/rsyslog.d/30-squid-forward.conf /etc/rsyslog.d/30-squid-forward.conf.disabled

# Restart rsyslog
systemctl restart rsyslog

# To re-enable:
mv /etc/rsyslog.d/30-squid-forward.conf.disabled /etc/rsyslog.d/30-squid-forward.conf
systemctl restart rsyslog
```

---

### Change Squid's Listening Port

```bash
# Edit Squid config
nano /etc/squid/squid.conf

# Find this line:
# http_port 3128

# Change to new port:
# http_port 8080

# Save and restart Squid
systemctl restart squid

# Update your curl commands to use new port:
curl -x http://localhost:8080 http://www.example.com/ -I
```

---

### Add Disk Queue for Reliability

```bash
# Edit rsyslog config
nano /etc/rsyslog.d/30-squid-forward.conf

# Add these lines BEFORE the forwarding rule:
$ActionQueueType LinkedList
$ActionQueueFileName squidqueue
$ActionQueueMaxDiskSpace 1g
$ActionQueueSaveOnShutdown on
$ActionResumeRetryCount -1

# Keep existing forwarding rule:
local5.* @YOUR_SYSLOG_IP:10001;RFC5424Format

# Create queue directory
mkdir -p /var/spool/rsyslog
chmod 700 /var/spool/rsyslog

# Restart rsyslog
systemctl restart rsyslog
```

---

## Troubleshooting Commands

### Quick Health Check

```bash
# Run all basic checks
echo "=== Squid Status ===" && systemctl status squid --no-pager | head -5
echo ""
echo "=== rsyslog Status ===" && systemctl status rsyslog --no-pager | head -5
echo ""
echo "=== Recent Squid Logs ===" && grep squid /var/log/messages | tail -3
echo ""
echo "=== Network Test ===" && ping -c 2 YOUR_SYSLOG_IP
```

---

### Check for Errors

```bash
# Check Squid errors in journal
journalctl -u squid -p err -n 20

# Check rsyslog errors in journal
journalctl -u rsyslog -p err -n 20

# Check Squid cache.log for errors
grep -i error /var/log/squid/cache.log | tail -20

# Check for FATAL errors
grep -i fatal /var/log/squid/cache.log

# Check system messages for problems
grep -iE "error|warning|failed" /var/log/messages | tail -20
```

---

### Debug Packet Flow

```bash
# Step 1: Start tcpdump in one terminal
timeout 30 tcpdump -i any -nn -A 'udp and dst port 10001' -c 3

# Step 2: In another terminal, generate traffic
curl -x http://localhost:3128 http://www.example.com/ -I

# Step 3: Check if packets captured
# Should see RFC 5424 formatted messages
```

---

### Reset Everything

```bash
# Complete reset (WARNING: stops services)

# Stop services
systemctl stop squid rsyslog

# Clear Squid cache
rm -rf /var/spool/squid/*
squid -z

# Start services
systemctl start rsyslog
systemctl start squid

# Verify
systemctl status squid rsyslog
```

---

### Restore Backup Configuration

```bash
# Restore Squid config from backup
cp /etc/squid/squid.conf.backup /etc/squid/squid.conf

# Test restored config
squid -k parse

# If OK, restart Squid
systemctl restart squid
```

---

## One-Liners

### Quick Status

```bash
# Everything in one command
systemctl status squid rsyslog --no-pager | grep -E "Active:|Loaded:"
```

---

### Quick Test

```bash
# Generate traffic and show log in one command
curl -x http://localhost:3128 http://www.example.com/ -I -s > /dev/null && sleep 2 && grep squid /var/log/messages | tail -1
```

---

### Quick Restart

```bash
# Restart both services
systemctl restart rsyslog squid && echo "Services restarted" || echo "ERROR: Restart failed"
```

---

### Quick Log View

```bash
# Last 10 logs with timestamps
grep squid /var/log/messages | tail -10 | cut -d' ' -f1-6,9-
```

---

## Useful Aliases

Add these to your `~/.bashrc` for shortcuts:

```bash
# Edit ~/.bashrc
nano ~/.bashrc

# Add these lines at the end:
alias squid-status='systemctl status squid'
alias squid-restart='systemctl restart squid'
alias squid-logs='grep squid /var/log/messages | tail -20'
alias squid-watch='tail -f /var/log/messages | grep --line-buffered squid'
alias squid-test='curl -x http://localhost:3128 http://www.example.com/ -I'
alias squid-config='nano /etc/squid/squid.conf'
alias rsyslog-config='nano /etc/rsyslog.d/30-squid-forward.conf'
alias squid-check='squid -k parse && rsyslogd -N1'

# Save and reload
source ~/.bashrc
```

**Now you can use:**
- `squid-status` instead of `systemctl status squid`
- `squid-logs` instead of `grep squid /var/log/messages | tail -20`
- `squid-test` instead of `curl -x http://localhost:3128...`
- etc.

---

## Emergency Commands

### Squid Won't Start

```bash
# Check what's wrong
squid -k parse
journalctl -u squid -n 50

# Common fixes
chown -R squid:squid /var/spool/squid
squid -z
setenforce 0  # Temporary SELinux disable
systemctl restart squid
```

---

### No Logs Appearing

```bash
# Check config
grep access_log /etc/squid/squid.conf

# Generate test traffic
curl -x http://localhost:3128 http://www.example.com/ -I

# Check logs
tail -f /var/log/messages | grep squid
```

---

### No Logs Reaching Remote Server

```bash
# Test network
ping -c 3 YOUR_SYSLOG_IP

# Check config
cat /etc/rsyslog.d/30-squid-forward.conf

# Watch packets
timeout 10 tcpdump -i any -nn 'udp and dst port 10001' -c 2

# Test with logger
logger -p local5.info "TEST"
```

---

## File Locations Quick Reference

```
Configuration Files:
├── /etc/squid/squid.conf                      (Squid main config)
├── /etc/rsyslog.d/30-squid-forward.conf       (Our forwarding config)
└── /etc/rsyslog.conf                          (rsyslog main config)

Log Files:
├── /var/log/messages                          (System log - Squid logs here)
├── /var/log/squid/cache.log                   (Squid diagnostics)
└── /var/log/squid/access.log                  (Only if stdio logging enabled)

Data Directories:
├── /var/spool/squid/                          (Squid cache)
└── /var/spool/rsyslog/                        (rsyslog queue - if disk queue enabled)

Binaries:
├── /usr/sbin/squid                            (Squid binary)
├── /usr/sbin/rsyslogd                         (rsyslog daemon)
└── /usr/lib64/squid/                          (Squid helpers)

Systemd Units:
├── /usr/lib/systemd/system/squid.service      (Squid service)
└── /usr/lib/systemd/system/rsyslog.service    (rsyslog service)
```

---

## Port Reference

| Port | Protocol | Purpose |
|------|----------|---------|
| 3128 | TCP | Squid proxy (default) |
| 514 | UDP | Standard syslog port (not used in this setup) |
| 10001 | UDP | Custom remote syslog port (what we use) |

---

## Related Documentation Files

- **Full Setup Guide:** `/root/SQUID_LOGGING_SETUP.md`
- **Architecture Explanation:** `/root/WHY_THIS_WORKS.md`
- **Troubleshooting:** `/root/TROUBLESHOOTING.md`
- **This File:** `/root/QUICK_REFERENCE.md`

---

**Pro Tip:** Keep this file open in another terminal for quick command lookups!


