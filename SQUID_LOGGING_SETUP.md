# Squid Proxy: Complete Guide to Shipping Logs to Remote Syslog Server

**Goal:** Send Squid proxy logs to a remote syslog server (10.0.0.241:10001) using UDP in RFC 5424 format

**Difficulty Level:** Beginner-friendly
**Time Required:** 15-20 minutes
**System:** Rocky Linux 9.7 with Squid 5.5 already installed

---

## Table of Contents
1. [Prerequisites and Tools](#prerequisites-and-tools)
2. [Understanding the Setup](#understanding-the-setup)
3. [Step-by-Step Configuration](#step-by-step-configuration)
4. [Testing Your Configuration](#testing-your-configuration)
5. [Verifying Logs Are Being Sent](#verifying-logs-are-being-sent)

---

## Prerequisites and Tools

Before starting, you'll need some troubleshooting tools installed. Think of these as your diagnostic equipment.

### Install tcpdump (Network Packet Sniffer)
**What it does:** Lets you see network traffic going in and out of your server. Like having X-ray vision for network packets.

```bash
dnf install -y tcpdump
```

**Expected output:**
```
Installed:
  tcpdump-14:4.99.0-9.el9.x86_64
Complete!
```

### Install netcat (Network Testing Tool)
**What it does:** Allows you to test network connections and send/receive data over TCP/UDP. Useful for testing if your remote syslog server is reachable.

```bash
dnf install -y nmap-ncat
```

**Expected output:**
```
Installed:
  nmap-ncat-7.92-1.el9.x86_64
Complete!
```

**Note:** On Rocky Linux 9, netcat is provided by the `nmap-ncat` package. The command is `nc`.

### Verify Squid is Installed
**What it does:** Confirms Squid proxy server is installed on your system.

```bash
squid -v
```

**Expected output:**
```
Squid Cache: Version 5.5
Service Name: squid
...
```

If you see this, you're good to go! If you get "command not found", Squid is not installed.

### Verify rsyslog is Installed
**What it does:** Checks that rsyslog (the log forwarding service) is installed. It comes pre-installed on Rocky Linux 9.7.

```bash
rpm -q rsyslog
```

**Expected output:**
```
rsyslog-8.2506.0-2.el9.x86_64
```

---

## Understanding the Setup

### Why Can't Squid Send Logs Directly in RFC 5424 Format?

**Short Answer:** Squid doesn't speak RFC 5424 natively when using UDP/TCP modules.

**Longer Explanation:**
- Squid has a `udp://` module, but it sends logs in Squid's own format (not RFC 5424 syslog format)
- Your remote syslog server expects RFC 5424 format
- If you send Squid's native format, the remote server won't understand it properly

### The Solution: Use rsyslog as a Translator

```
┌────────┐              ┌─────────┐              ┌──────────────────────┐
│ Squid  │─────────────>│ rsyslog │─────────────>│  Remote Syslog       │
│ Proxy  │    syslog    │ (local) │     UDP      │  (S1 Collector)      │
└────────┘              └─────────┘   RFC5424    │  10.0.0.241          │
                                                 └──────────────────────┘
```

**How it works:**
1. Squid sends logs to the LOCAL syslog (using the syslog module)
2. Local rsyslog receives logs and reformats them to RFC 5424
3. rsyslog forwards the formatted logs to your remote server via UDP

### Why Use Local rsyslog? (Benefits)

**1. Reliability - Spooling/Buffering**
- rsyslog can store logs in a queue if the network is down
- Squid's native UDP just drops logs if the network hiccups
- Think of it like: Squid UDP = throwing letters at a mailbox vs rsyslog = having a mail carrier with a bag

**2. Performance**
- rsyslog handles the network sending in a separate process
- Squid doesn't have to wait for logs to be sent
- Squid can focus on proxying traffic, not managing log delivery

**3. Flexibility**
- Need to change the remote server IP? Just edit rsyslog config, no need to restart Squid
- Want to send logs to multiple destinations? rsyslog can do that easily
- Need to filter certain logs? rsyslog has powerful filtering

**4. Format Conversion**
- rsyslog can convert to RFC 5424 format (which Squid cannot do natively)
- Can add extra fields, modify messages, etc.

**5. Guaranteed Delivery (with disk queue)**
- Can configure rsyslog to use disk-based queues
- Even if server is down for hours, logs won't be lost

---

## Step-by-Step Configuration

### Step 1: Back Up the Original Squid Configuration

**Why:** Always keep a backup so you can restore if something goes wrong.

```bash
cp /etc/squid/squid.conf /etc/squid/squid.conf.backup
```

**What this does:** Makes a copy of the original config file with `.backup` extension

**Verify the backup:**
```bash
ls -lh /etc/squid/squid.conf*
```

**Expected output:**
```
-rw-r--r--. 1 root squid 2.0K Dec  2 20:28 /etc/squid/squid.conf
-rw-r--r--. 1 root squid 2.0K Dec  2 20:45 /etc/squid/squid.conf.backup
```

---

### Step 2: Configure Squid to Send Logs to Local Syslog

**What we're doing:** Telling Squid to send its access logs to the local syslog service.

**Edit the Squid configuration file:**
```bash
vi /etc/squid/squid.conf
```

**OR if you prefer nano (easier for beginners):**
```bash
nano /etc/squid/squid.conf
```

**Find this section** (around line 67):
```
# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid
```

**Add these lines RIGHT AFTER the `coredump_dir` line:**
```
# Configure access logging to local syslog (local5 facility, info priority)
# This will be forwarded to remote syslog server via rsyslog in RFC 5424 format
access_log syslog:local5.info squid
```

**What each part means:**
- `access_log` = the directive that controls where logs go
- `syslog:` = tells Squid to use the syslog module (not file, not UDP, not TCP)
- `local5` = the syslog "facility" (like a channel or category) - we use local5 because it's typically unused
- `info` = the log priority level
- `squid` = the log format to use (Squid's default access log format)

**Save and exit:**
- In `vi`: Press `ESC`, type `:wq`, press `ENTER`
- In `nano`: Press `CTRL+X`, then `Y`, then `ENTER`

**Verify your edit:**
```bash
grep "access_log syslog" /etc/squid/squid.conf
```

**Expected output:**
```
access_log syslog:local5.info squid
```

---

### Step 3: Test Squid Configuration for Errors

**Why:** Check that we didn't make any typos before restarting Squid.

```bash
squid -k parse
```

**What this does:** Squid reads the config file and checks for errors WITHOUT actually applying changes.

**Expected output:** You'll see processing messages. If there are NO error messages, you're good!

```
2025/12/02 20:28:33| Processing Configuration File: /etc/squid/squid.conf (depth 0)
...
```

**If you see errors:** Go back to Step 2 and fix the typo.

---

### Step 4: Restart Squid to Apply Configuration

**What we're doing:** Restarting Squid so it starts using the new log configuration.

```bash
systemctl restart squid
```

**Wait a few seconds, then check if Squid started successfully:**
```bash
systemctl status squid
```

**Expected output:**
```
● squid.service - Squid caching proxy
   Active: active (running) since...
```

Look for **"active (running)"** in green. That means it's working!

**If you see "failed" in red:**
```bash
journalctl -u squid -n 50
```
This shows you the last 50 log lines to see what went wrong.

---

### Step 5: Create rsyslog Configuration for Remote Forwarding

**What we're doing:** Creating a NEW configuration file that tells rsyslog to forward Squid logs to your remote server in RFC 5424 format.

**Why a separate file?**
- Keeps configs organized
- Easy to enable/disable by renaming the file
- Won't interfere with other rsyslog settings

**Create the configuration file:**
```bash
nano /etc/rsyslog.d/30-squid-forward.conf
```

**Paste this entire configuration** (use right-click to paste in most terminals):

```
# Forward Squid proxy logs to remote syslog server in RFC 5424 format
# This configuration sends local5.* logs to 10.0.0.241:10001 via UDP

# Template for RFC 5424 format
# See: https://www.rfc-editor.org/rfc/rfc5424
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n"
)

# Forward local5 facility logs to remote syslog server using RFC 5424 format
local5.* @10.0.0.241:10001;RFC5424Format
```

**Understanding each part:**

1. **Template Definition** (lines 5-7):
   - Creates a template named "RFC5424Format"
   - Defines how the log message should be formatted
   - `%PRI%` = priority/facility
   - `%TIMESTAMP:::date-rfc3339%` = timestamp in RFC 3339 format
   - `%HOSTNAME%` = your server's hostname
   - `%APP-NAME%` = application name (squid-1)
   - `%PROCID%` = process ID
   - `%msg%` = the actual log message from Squid

2. **Forwarding Rule** (line 10):
   - `local5.*` = all logs from the local5 facility (any priority level)
   - `@` = send via UDP (one @ symbol)
   - `10.0.0.241:10001` = destination IP and port
   - `;RFC5424Format` = use the template we defined above

**Important:** If you need to change the destination IP or port, just edit line 10!

**Save and exit:** Press `CTRL+X`, then `Y`, then `ENTER`

**Verify the file was created:**
```bash
cat /etc/rsyslog.d/30-squid-forward.conf
```

You should see the configuration you just created.

---

### Step 6: Test rsyslog Configuration for Errors

**Why:** Make sure we didn't make syntax errors in the rsyslog config.

```bash
rsyslogd -N1
```

**What this does:** Tests the rsyslog configuration without actually starting it.

**Expected output:**
```
rsyslogd: version 8.2506.0-2.el9, config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
```

**If you see errors:** Go back to Step 5 and check for typos.

---

### Step 7: Restart rsyslog to Apply Configuration

**What we're doing:** Restarting rsyslog so it starts forwarding logs.

```bash
systemctl restart rsyslog
```

**Check if rsyslog started successfully:**
```bash
systemctl status rsyslog
```

**Expected output:**
```
● rsyslog.service - System Logging Service
   Active: active (running) since...
```

Look for **"active (running)"** in green.

---

## Testing Your Configuration

Now let's test if everything works!

### Test 1: Generate Proxy Traffic

**What we're doing:** Making a request through the Squid proxy to generate a log entry.

```bash
curl -x http://localhost:3128 http://www.example.com/ -I
```

**What this command does:**
- `curl` = command line tool to make web requests
- `-x http://localhost:3128` = use Squid proxy on port 3128
- `http://www.example.com/` = the website to request
- `-I` = only get headers (faster, smaller response)

**Expected output:**
```
HTTP/1.1 200 OK
Content-Type: text/html
...
X-Cache: MISS from squidproxy
Via: 1.1 squidproxy (squid/5.5)
```

If you see this, Squid is working!

---

### Test 2: Verify Logs Appear in Local Syslog

**What we're doing:** Checking that Squid is sending logs to local syslog.

```bash
grep -i squid /var/log/messages | tail -5
```

**What this command does:**
- `grep -i squid` = search for lines containing "squid" (case-insensitive)
- `/var/log/messages` = the main system log file
- `tail -5` = show only the last 5 lines

**Expected output:**
```
Dec  2 20:35:09 squidproxy (squid-1)[13292]: 1764725709.484     88 ::1 TCP_MISS/200 348 HEAD http://www.example.com/ - HIER_DIRECT/216.68.12.65 text/html
```

**Understanding the log:**
- `Dec 2 20:35:09` = timestamp
- `squidproxy` = your hostname
- `(squid-1)[13292]` = program name and process ID
- Rest = Squid's access log format showing the web request

**If you DON'T see any logs:**
- Wait 30 seconds and try again (logs might be buffered)
- Check if Squid is running: `systemctl status squid`
- See TROUBLESHOOTING.md

---

## Verifying Logs Are Being Sent

### Test 3: Capture Network Traffic to Remote Server

**What we're doing:** Using tcpdump to literally watch the UDP packets being sent to 10.0.0.241:10001.

**Start the packet capture:**
```bash
timeout 15 tcpdump -i any -nn -A 'udp and dst host 10.0.0.241 and dst port 10001' -c 3
```

**What this command does:**
- `timeout 15` = automatically stop after 15 seconds
- `tcpdump` = packet capture tool
- `-i any` = listen on all network interfaces
- `-nn` = don't resolve hostnames or port names (faster)
- `-A` = show packet contents in ASCII (so we can read the logs)
- `udp` = only show UDP packets
- `dst host 10.0.0.241` = only packets going TO this IP
- `dst port 10001` = only packets going TO this port
- `-c 3` = capture 3 packets then stop

**This will wait for packets...** Keep this running and open a NEW terminal window (or SSH session) to the server.

---

### Test 4: Generate Traffic in Another Terminal

**In the NEW terminal window, run:**
```bash
curl -x http://localhost:3128 http://www.google.com/ -I
```

**Then check your tcpdump window** - you should see output like this:

```
20:38:05.263860 ens18 Out IP 10.0.0.188.58897 > 10.0.0.241.10001: UDP, length 181
E...0.@.@...
...
.....'....{<174>1 2025-12-02T20:38:05.173191-05:00 squidproxy (squid-1) 13292 - - 1764725885.173    104 ::1 TCP_MISS/200 1175 HEAD http://www.google.com/ - HIER_DIRECT/74.125.21.103 text/html
```

**This is EXACTLY what we want to see!**

**Breaking down the RFC 5424 format:**
- `<174>` = Priority (facility: local5=21, severity: info=6, calculation: 21*8+6=174)
- `1` = RFC 5424 version number
- `2025-12-02T20:38:05.173191-05:00` = Timestamp in RFC 3339 format
- `squidproxy` = Hostname
- `(squid-1)` = Application name
- `13292` = Process ID
- `-` = Message ID (empty)
- `-` = Structured data (empty)
- Rest = The actual Squid log message

**If you DON'T see packets:**
- Make sure the destination IP (10.0.0.241) is reachable from your server
- Check if there's a firewall blocking UDP port 10001
- See TROUBLESHOOTING.md

---

## Success Indicators

You've successfully configured everything if you see:

✅ **Step 1:** Squid config has `access_log syslog:local5.info squid`
✅ **Step 2:** rsyslog config exists at `/etc/rsyslog.d/30-squid-forward.conf`
✅ **Step 3:** Squid is running (`systemctl status squid` shows active)
✅ **Step 4:** rsyslog is running (`systemctl status rsyslog` shows active)
✅ **Step 5:** Squid logs appear in `/var/log/messages`
✅ **Step 6:** tcpdump shows UDP packets going to 10.0.0.241:10001
✅ **Step 7:** Packets contain RFC 5424 formatted messages (version `1` after the priority)

---

## What Happens on Reboot?

**Good news:** Both Squid and rsyslog are configured to start automatically on boot!

**To verify:**
```bash
systemctl is-enabled squid
systemctl is-enabled rsyslog
```

**Expected output:**
```
enabled
enabled
```

If either says "disabled", enable them:
```bash
systemctl enable squid
systemctl enable rsyslog
```

---

## Quick Daily Checks

**Check if Squid is running:**
```bash
systemctl status squid
```

**Check if logs are being sent (look at recent messages):**
```bash
grep -i squid /var/log/messages | tail -10
```

**Check rsyslog is running:**
```bash
systemctl status rsyslog
```

---

## Configuration File Locations Reference

| File | Purpose | Can I Edit? |
|------|---------|-------------|
| `/etc/squid/squid.conf` | Main Squid configuration | YES (carefully!) |
| `/etc/squid/squid.conf.backup` | Backup of original config | Restore if needed |
| `/etc/rsyslog.d/30-squid-forward.conf` | Our custom log forwarding rules | YES |
| `/var/log/messages` | Local system log (where Squid logs first appear) | NO (read-only) |

---

## Next Steps

- **Need to change the remote server IP/port?** Edit `/etc/rsyslog.d/30-squid-forward.conf` line 10, then `systemctl restart rsyslog`
- **Want to send to multiple servers?** Add more lines like line 10 with different IPs
- **Having problems?** See `TROUBLESHOOTING.md`
- **Want to understand the architecture better?** See `WHY_THIS_WORKS.md`
- **Need quick commands?** See `QUICK_REFERENCE.md`

---

**You're all set! Your Squid proxy is now forwarding logs in RFC 5424 format to your remote syslog server.**
