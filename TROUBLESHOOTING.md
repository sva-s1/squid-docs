# Squid Syslog Forwarding: Troubleshooting Guide

**For:** Diagnosing and fixing common problems with Squid → rsyslog → remote syslog setup

---

## Table of Contents
1. [Quick Diagnostic Checklist](#quick-diagnostic-checklist)
2. [Problem: Squid Won't Start](#problem-squid-wont-start)
3. [Problem: rsyslog Won't Start](#problem-rsyslog-wont-start)
4. [Problem: No Logs in /var/log/messages](#problem-no-logs-in-varlogmessages)
5. [Problem: No Logs Reaching Remote Server](#problem-no-logs-reaching-remote-server)
6. [Problem: Logs Are Malformed on Remote Server](#problem-logs-are-malformed-on-remote-server)
7. [Problem: Performance Issues](#problem-performance-issues)
8. [Network Troubleshooting](#network-troubleshooting)
9. [Log Analysis Tools](#log-analysis-tools)
10. [Getting Help](#getting-help)

---

## Quick Diagnostic Checklist

Run these commands to quickly identify where the problem is:

```bash
# 1. Is Squid running?
systemctl status squid

# 2. Is rsyslog running?
systemctl status rsyslog

# 3. Can Squid config parse without errors?
squid -k parse

# 4. Can rsyslog config parse without errors?
rsyslogd -N1

# 5. Are Squid logs reaching local syslog?
grep -i squid /var/log/messages | tail -5

# 6. Is network reachable?
ping -c 3 10.0.0.241

# 7. Is UDP port 10001 reachable?
nc -zvu 10.0.0.241 10001

# 8. Are packets being sent?
timeout 5 tcpdump -i any -nn 'udp and dst port 10001' -c 1
```

Based on which command fails, jump to the relevant section below.

---

## Problem: Squid Won't Start

### Symptom
```bash
systemctl status squid
```
Shows: **`Active: failed`** or **`Active: inactive (dead)`**

---

### Solution 1: Check Configuration Syntax

**Step 1: Test configuration**
```bash
squid -k parse
```

**If you see errors like:**
```
ERROR: Directive 'access_log' is not recognized
```

**Fix:** You have a typo in `/etc/squid/squid.conf`
```bash
# Check what you actually wrote
grep access_log /etc/squid/squid.conf

# Should be exactly:
access_log syslog:local5.info squid
```

**Step 2: Check for extra spaces or tabs**
```bash
cat -A /etc/squid/squid.conf | grep access_log
```

Look for:
- `$` at end of line = good
- Extra `$` or `^I` (tabs) = bad (remove them)

---

### Solution 2: Check Squid Error Logs

```bash
journalctl -u squid -n 50 --no-pager
```

**Common errors and fixes:**

**Error:** `FATAL: Failed to make swap directory /var/spool/squid: (13) Permission denied`
```bash
# Fix permissions
chown -R squid:squid /var/spool/squid
chmod 750 /var/spool/squid

# Reinitialize cache directories
squid -z

# Try starting again
systemctl restart squid
```

**Error:** `FATAL: Bungled squid.conf line X: access_log syslog:local5.info squid`
```bash
# Check for typos, make sure it's EXACTLY:
access_log syslog:local5.info squid

# No extra spaces, correct facility (local5), correct priority (info)
```

**Error:** `FATAL: Cannot open HTTP Port`
```bash
# Port 3128 already in use
ss -tlnp | grep 3128

# Kill other process or change Squid's port in squid.conf:
http_port 3129
```

---

### Solution 3: SELinux Issues (Rocky/RHEL/CentOS)

**Check if SELinux is blocking:**
```bash
getenforce
```

If it says **`Enforcing`**, check for denials:
```bash
ausearch -m avc -ts recent | grep squid
```

**Temporary fix (for testing only):**
```bash
setenforce 0
systemctl restart squid
```

**If Squid starts now, SELinux was the problem.**

**Permanent fix:**
```bash
# Re-enable SELinux
setenforce 1

# Allow Squid to write to syslog
setsebool -P squid_use_syslog on

# Restart Squid
systemctl restart squid
```

---

### Solution 4: Restore Backup Configuration

If you've tried everything:
```bash
# Restore original config
cp /etc/squid/squid.conf.backup /etc/squid/squid.conf

# Start Squid with original config
systemctl restart squid

# Verify it works
systemctl status squid

# Now re-apply changes carefully, one line at a time
```

---

## Problem: rsyslog Won't Start

### Symptom
```bash
systemctl status rsyslog
```
Shows: **`Active: failed`** or **`Active: inactive (dead)`**

---

### Solution 1: Check Configuration Syntax

```bash
rsyslogd -N1
```

**If you see errors:**
```
error during parsing file /etc/rsyslog.d/30-squid-forward.conf, on or before line X
```

**Common mistakes:**

**Missing closing parenthesis:**
```bash
# WRONG:
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME%"

# RIGHT (note closing parenthesis):
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME%"
)
```

**Wrong quote type:**
```bash
# WRONG (fancy quotes):
template(name="RFC5424Format" type="string"

# RIGHT (straight quotes):
template(name="RFC5424Format" type="string"
```

**Fix:** Edit `/etc/rsyslog.d/30-squid-forward.conf` and fix syntax error.

---

### Solution 2: Check for File Syntax

```bash
cat -A /etc/rsyslog.d/30-squid-forward.conf
```

Look for weird characters that shouldn't be there:
- `^M` = Windows line endings (bad)
- Non-ASCII characters = bad

**Fix Windows line endings:**
```bash
dos2unix /etc/rsyslog.d/30-squid-forward.conf
```

Or manually:
```bash
sed -i 's/\r$//' /etc/rsyslog.d/30-squid-forward.conf
```

---

### Solution 3: Check rsyslog Service Logs

```bash
journalctl -u rsyslog -n 50 --no-pager
```

**Look for:** Any ERROR or FATAL messages

---

### Solution 4: Disable Custom Config and Test

```bash
# Temporarily rename your config
mv /etc/rsyslog.d/30-squid-forward.conf /etc/rsyslog.d/30-squid-forward.conf.disabled

# Try starting rsyslog
systemctl restart rsyslog
systemctl status rsyslog
```

**If rsyslog starts now:**
- The problem is in your custom config file
- Fix syntax in `/etc/rsyslog.d/30-squid-forward.conf.disabled`
- Rename back and test again

---

## Problem: No Logs in /var/log/messages

### Symptom
```bash
grep -i squid /var/log/messages
```
Returns nothing or very old logs.

---

### Solution 1: Verify Squid is Actually Logging

**Step 1: Check Squid's access_log configuration**
```bash
grep access_log /etc/squid/squid.conf
```

**Should see:**
```
access_log syslog:local5.info squid
```

**If you see something else like:**
```
# access_log syslog:local5.info squid
```
The `#` means it's commented out! Remove the `#`:
```bash
vi /etc/squid/squid.conf
# Remove the # before access_log
# Save and exit
systemctl restart squid
```

---

### Solution 2: Generate Test Traffic

Maybe Squid just hasn't received any requests yet!

```bash
# Generate a test request
curl -x http://localhost:3128 http://www.example.com/ -I

# Wait 5 seconds for log to flush
sleep 5

# Check again
grep -i squid /var/log/messages | tail -5
```

---

### Solution 3: Check rsyslog is Processing local5

**Verify rsyslog received the log:**
```bash
# Check rsyslog's debug output
rsyslogd -N1 | grep local5
```

**Check if local5 logs are being filtered out:**
```bash
grep -r "local5" /etc/rsyslog.d/
```

**Make sure there's no rule like:**
```bash
local5.* ~
```
This would DISCARD local5 logs (the `~` means discard).

---

### Solution 4: Check File Permissions

```bash
ls -lh /var/log/messages
```

**Should see:**
```
-rw-r--r--. 1 root root 123K Dec  2 20:45 /var/log/messages
```

**If permissions are wrong:**
```bash
chmod 644 /var/log/messages
chown root:root /var/log/messages
systemctl restart rsyslog
```

---

### Solution 5: Restart Both Services

Sometimes logs get stuck:
```bash
systemctl restart squid
systemctl restart rsyslog

# Generate test traffic
curl -x http://localhost:3128 http://www.google.com/ -I

# Check logs
tail -f /var/log/messages
```

---

## Problem: No Logs Reaching Remote Server

### Symptom
- Logs appear in `/var/log/messages` ✅
- But remote server never receives them ❌

---

### Solution 1: Test Network Connectivity

**Step 1: Can you ping the remote server?**
```bash
ping -c 5 10.0.0.241
```

**Expected output:**
```
5 packets transmitted, 5 received, 0% packet loss
```

**If you see `100% packet loss`:**
- Remote server is down, or
- Network path is broken, or
- Firewall is blocking ICMP

**Fix:** Verify remote server is up and network is working.

---

**Step 2: Can you reach UDP port 10001?**
```bash
nc -zvu 10.0.0.241 10001
```

**Expected output:**
```
Connection to 10.0.0.241 10001 port [udp/*] succeeded!
```

**If you see `failed`:**
- Port is blocked by firewall, or
- Remote syslog server not listening on that port

**Install nc if needed:**
```bash
dnf install -y nc
```

---

### Solution 2: Check Firewall (Local)

**Check if firewall is blocking outbound UDP:**
```bash
firewall-cmd --list-all
```

**Allow outbound UDP (usually allowed by default):**
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" destination address="10.0.0.241/32" port port="10001" protocol="udp" accept'
firewall-cmd --reload
```

---

### Solution 3: Verify rsyslog Configuration

**Check forwarding rule exists:**
```bash
grep "@10.0.0.241" /etc/rsyslog.d/30-squid-forward.conf
```

**Should see:**
```
local5.* @10.0.0.241:10001;RFC5424Format
```

**Common mistakes:**

**Missing @ symbol:**
```bash
# WRONG:
local5.* 10.0.0.241:10001;RFC5424Format

# RIGHT (note the @ before IP):
local5.* @10.0.0.241:10001;RFC5424Format
```

**Single @ = UDP, Double @@ = TCP:**
```bash
@10.0.0.241:10001   # UDP (correct for your setup)
@@10.0.0.241:10001  # TCP (wrong)
```

---

### Solution 4: Capture Packets with tcpdump

**Verify packets are being sent:**
```bash
# Start packet capture
timeout 15 tcpdump -i any -nn -c 5 'udp and dst host 10.0.0.241 and dst port 10001'
```

**In another terminal, generate traffic:**
```bash
curl -x http://localhost:3128 http://www.example.com/ -I
```

**What you should see in tcpdump:**
```
IP 10.0.0.188.XXXXX > 10.0.0.241.10001: UDP, length 200
```

**If you see packets:**
- ✅ rsyslog IS sending
- Problem is network or remote server

**If you DON'T see packets:**
- ❌ rsyslog is NOT sending
- Check rsyslog config and restart rsyslog

---

### Solution 5: Check rsyslog Queue

**Maybe queue is stuck or full:**
```bash
# Check rsyslog debug info
kill -USR1 $(cat /var/run/rsyslogd.pid)

# Check syslog for debug output
tail -50 /var/log/messages | grep rsyslog
```

**Restart rsyslog to clear queue:**
```bash
systemctl restart rsyslog
```

---

### Solution 6: Test with logger Command

**Bypass Squid and test rsyslog directly:**
```bash
logger -p local5.info "TEST MESSAGE FROM LOGGER"
```

**Then check tcpdump:**
```bash
timeout 5 tcpdump -i any -nn -A 'udp and dst port 10001' -c 1
```

**Should see:** Your test message being sent

**If you see test message but NOT Squid logs:**
- Problem is with Squid → rsyslog communication
- Go back to "No Logs in /var/log/messages" section

---

## Problem: Logs Are Malformed on Remote Server

### Symptom
Remote server receives logs but they're not in proper RFC 5424 format.

---

### Solution 1: Verify Template is Applied

**Check your rsyslog config:**
```bash
cat /etc/rsyslog.d/30-squid-forward.conf
```

**Make sure the template is defined AND referenced:**
```bash
# Template must be defined
template(name="RFC5424Format" type="string" ...

# AND it must be referenced in the forwarding rule (after semicolon)
local5.* @10.0.0.241:10001;RFC5424Format
```

**Common mistake - missing semicolon and template name:**
```bash
# WRONG:
local5.* @10.0.0.241:10001

# RIGHT:
local5.* @10.0.0.241:10001;RFC5424Format
```

---

### Solution 2: Check Template Syntax

**Template must be on multiple lines with closing parenthesis:**
```bash
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n"
)
```

**Note the `)` on the last line!**

---

### Solution 3: Capture and Inspect Actual Packet

```bash
timeout 10 tcpdump -i any -nn -A 'udp and dst port 10001' -c 1
```

**Look for the RFC 5424 format:**
```
<174>1 2025-12-02T20:38:05.173191-05:00 squidproxy (squid-1) 13292 - - message...
```

**Check for:**
- Priority in angle brackets: `<174>`
- Version number: `1`
- RFC 3339 timestamp: `2025-12-02T20:38:05.173191-05:00`

**If you see this format:** rsyslog IS sending correctly - problem is on remote server's parsing

**If you DON'T see this format:** Template is not being applied correctly

---

## Problem: Performance Issues

### Symptom
Squid is slow, high CPU usage, or high memory usage.

---

### Solution 1: Check if Logging is Blocking

**Monitor rsyslog's queue:**
```bash
# Check rsyslog stats
kill -USR1 $(cat /var/run/rsyslogd.pid)
tail -20 /var/log/messages | grep -i queue
```

**Look for:**
- `queue full` = Queue is overflowing
- High enqueued count = Logs backing up

---

### Solution 2: Increase rsyslog Queue Size

**Edit `/etc/rsyslog.d/30-squid-forward.conf`:**

**Add BEFORE the forwarding rule:**
```bash
# Increase queue size
$ActionQueueSize 100000      # Default is 10000
$ActionQueueDiscardMark 80000
$ActionQueueHighWaterMark 60000

local5.* @10.0.0.241:10001;RFC5424Format
```

**Restart rsyslog:**
```bash
systemctl restart rsyslog
```

---

### Solution 3: Enable Disk Queue (for high volume)

**Add to `/etc/rsyslog.d/30-squid-forward.conf` BEFORE forwarding rule:**
```bash
# Use disk-assisted queue
$ActionQueueType LinkedList
$ActionQueueFileName squidqueue
$ActionQueueMaxDiskSpace 1g
$ActionQueueSaveOnShutdown on
$ActionResumeRetryCount -1

local5.* @10.0.0.241:10001;RFC5424Format
```

**Create queue directory:**
```bash
mkdir -p /var/spool/rsyslog
chown root:root /var/spool/rsyslog
chmod 700 /var/spool/rsyslog
```

**Restart rsyslog:**
```bash
systemctl restart rsyslog
```

---

### Solution 4: Check CPU and Memory

```bash
# Check Squid CPU/memory
ps aux | grep squid

# Check rsyslog CPU/memory
ps aux | grep rsyslog
```

**If rsyslog is using > 5% CPU consistently:**
- You might be sending LOTS of logs
- Consider filtering to reduce volume
- Or increase server resources

---

## Network Troubleshooting

### Test UDP Connectivity

**Method 1: Using nc (netcat)**
```bash
# Install nc if needed
dnf install -y nc

# Test UDP connection
nc -zvu 10.0.0.241 10001
```

---

**Method 2: Using socat (advanced)**
```bash
# Install socat if needed
dnf install -y socat

# Send a test UDP packet
echo "TEST" | socat - UDP4:10.0.0.241:10001

# On remote server, listen for it:
socat UDP4-LISTEN:10001,fork STDOUT
```

---

### Check Routing

```bash
# How do packets reach 10.0.0.241?
ip route get 10.0.0.241
```

**Expected output:**
```
10.0.0.241 via 10.0.0.1 dev eth0 src 10.0.0.188
```

**Check:**
- Correct interface (eth0, ens18, etc.)
- Correct gateway
- Correct source IP

---

### Check MTU (Maximum Transmission Unit)

**Large log messages might be fragmented:**
```bash
# Check interface MTU
ip link show
```

**Should see:**
```
mtu 1500
```

**If logs are > 1500 bytes, they'll be fragmented. Test:**
```bash
ping -M do -s 1472 10.0.0.241
```

**If ping fails:** MTU mismatch, network doesn't support large packets

**Solution:** Reduce log verbosity or fix network MTU

---

## Log Analysis Tools

### View Logs in Real-Time

```bash
# Watch /var/log/messages live
tail -f /var/log/messages | grep --line-buffered squid
```

Press `CTRL+C` to stop.

---

### Count Logs Per Minute

```bash
# How many Squid logs in last hour?
grep -c "squid" /var/log/messages

# How many per minute (last 60 minutes)?
for i in {1..60}; do
  echo -n "Minute $i: "
  grep "$(date -d "$i minutes ago" '+%b %e %H:%M')" /var/log/messages | grep -c squid
done
```

---

### Search for Specific URLs

```bash
# Find logs for example.com
grep squid /var/log/messages | grep "example.com"

# Find logs with HTTP errors (4xx, 5xx)
grep squid /var/log/messages | grep -E "TCP_.*/(4[0-9]{2}|5[0-9]{2})"
```

---

### Extract Client IPs

```bash
# See which clients are using the proxy
grep squid /var/log/messages | awk '{print $9}' | sort | uniq -c | sort -rn
```

---

## Getting Help

### Collect Diagnostic Information

**Before asking for help, collect this info:**

```bash
# Create a diagnostic report
cat > /root/squid-diagnostic.txt <<'EOF'
====== SQUID VERSION ======
EOF
squid -v >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== SQUID STATUS ======
EOF
systemctl status squid >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== RSYSLOG STATUS ======
EOF
systemctl status rsyslog >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== SQUID CONFIG (access_log lines) ======
EOF
grep -E "^access_log|^#.*access_log" /etc/squid/squid.conf >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== RSYSLOG CONFIG ======
EOF
cat /etc/rsyslog.d/30-squid-forward.conf >> /root/squid-diagnostic.txt 2>&1

cat >> /root/squid-diagnostic.txt <<'EOF'

====== RECENT SQUID LOGS ======
EOF
grep squid /var/log/messages | tail -10 >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== SQUID JOURNAL ERRORS ======
EOF
journalctl -u squid -n 20 --no-pager >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== RSYSLOG JOURNAL ERRORS ======
EOF
journalctl -u rsyslog -n 20 --no-pager >> /root/squid-diagnostic.txt

cat >> /root/squid-diagnostic.txt <<'EOF'

====== NETWORK CONNECTIVITY ======
EOF
ping -c 3 10.0.0.241 >> /root/squid-diagnostic.txt 2>&1

echo "Diagnostic report created: /root/squid-diagnostic.txt"
```

**Share this file when asking for help!**

---

### Enable Debug Logging (Advanced)

**Squid debug logging:**
```bash
# Edit /etc/squid/squid.conf
# Add this line:
debug_options ALL,1 33,2 28,3

# Restart Squid
systemctl restart squid

# Check debug logs
journalctl -u squid -f
```

**rsyslog debug logging:**
```bash
# Edit /etc/rsyslog.conf
# Add this line at the top:
$DebugLevel 2

# Restart rsyslog
systemctl restart rsyslog

# Check debug logs
tail -f /var/log/messages | grep rsyslog
```

**IMPORTANT:** Disable debug logging after troubleshooting (generates lots of logs!)

---

## Common Error Messages and Solutions

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `FATAL: Failed to make swap directory` | Permission issue | `chown -R squid:squid /var/spool/squid && squid -z` |
| `ERROR: No running copy` | Squid not running | `systemctl start squid` |
| `WARNING: Cannot open HTTP Port` | Port already in use | Check with `ss -tlnp \| grep 3128`, kill other process |
| `error during parsing file` | rsyslog syntax error | Run `rsyslogd -N1` to see exact error |
| `Connection refused` | Remote server not listening | Verify remote server is running and accepting UDP on port 10001 |
| `Network unreachable` | Routing problem | Check `ip route` and network configuration |

---

## Still Having Problems?

1. ✅ **Re-read the setup guide:** `/root/SQUID_LOGGING_SETUP.md`
2. ✅ **Check architecture explanation:** `/root/WHY_THIS_WORKS.md`
3. ✅ **Generate diagnostic report:** See "Collect Diagnostic Information" above
4. ✅ **Check Squid documentation:** https://wiki.squid-cache.org
5. ✅ **Check rsyslog documentation:** https://www.rsyslog.com/doc/

**Need more help? Share:**
- The diagnostic report from `/root/squid-diagnostic.txt`
- Exact error messages
- What you've already tried

Good luck!
