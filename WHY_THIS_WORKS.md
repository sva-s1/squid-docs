# Understanding Why This Squid Logging Architecture Works

**For:** People who want to understand the "why" behind the configuration
**Level:** Beginner to Intermediate

---

## Table of Contents
1. [The Problem We're Solving](#the-problem-were-solving)
2. [Why Not Use Squid's Native UDP/TCP Logging?](#why-not-use-squids-native-udptcp-logging)
3. [The Architecture](#the-architecture)
4. [Benefits of Using rsyslog as Intermediary](#benefits-of-using-rsyslog-as-intermediary)
5. [Understanding RFC 5424 Format](#understanding-rfc-5424-format)
6. [Understanding Syslog Facilities and Priorities](#understanding-syslog-facilities-and-priorities)
7. [Performance Impact](#performance-impact)
8. [Reliability and Data Loss Prevention](#reliability-and-data-loss-prevention)

---

## The Problem We're Solving

### What You Actually Want
Send Squid proxy access logs to a remote syslog server using:
- **Transport:** UDP
- **Port:** 10001 (custom port)
- **Format:** RFC 5424 (standardized syslog format)

### What Seems Like It Should Work (But Doesn't)
```
access_log udp://YOUR_SYSLOG_IP:10001 squid
```

**Why it doesn't work:**
1. Squid's UDP module sends logs in **Squid's native format**, not RFC 5424
2. The remote syslog server expects RFC 5424 format
3. Format mismatch = logs might be rejected, malformed, or unparseable

**Real example of what Squid UDP sends:**
```
1764725709.484     88 ::1 TCP_MISS/200 348 HEAD http://www.example.com/ - HIER_DIRECT/216.68.12.65 text/html
```

**What RFC 5424 format looks like:**
```
<174>1 2025-12-02T20:38:05.173191-05:00 squidproxy (squid-1) 13292 - - 1764725709.484     88 ::1 TCP_MISS/200 348 HEAD http://www.example.com/ - HIER_DIRECT/216.68.12.65 text/html
```

Notice the difference? RFC 5424 has:
- Priority in angle brackets `<174>`
- Version number `1`
- Structured timestamp `2025-12-02T20:38:05.173191-05:00`
- Hostname, app name, process ID before the actual message

---

## Why Not Use Squid's Native UDP/TCP Logging?

### Squid's Available Logging Modules

| Module | Syntax | Format | RFC 5424? |
|--------|--------|--------|-----------|
| stdio | `stdio:/path/to/file` | Squid native | ❌ No |
| daemon | `daemon:/path/to/file` | Squid native | ❌ No |
| **syslog** | `syslog:facility.priority` | **Converted by OS syslog** | ✅ **YES** |
| udp | `udp://host:port` | Squid native | ❌ No |
| tcp | `tcp://host:port` | Squid native | ❌ No |

**Key insight:** Only the `syslog` module can produce RFC 5424 format, because it delegates formatting to the operating system's syslog service (rsyslog).

### What Customer Tried (That Didn't Work Well)

Your customer said TCP "sucked" - here's why:

**Configuration attempted:**
```
access_log tcp://172.24.3.230:9537 squid
```

**Problems with this approach:**
1. **Format:** Still Squid's native format, not RFC 5424
2. **Blocking:** Squid has to wait for TCP acknowledgment, slows down request processing
3. **No buffering:** If connection breaks, logs are lost immediately
4. **No format control:** Can't modify or enrich logs before sending

---

## The Architecture

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR SERVER                              │
│                                                                 │
│  ┌──────────┐                                                   │
│  │  Client  │                                                   │
│  │ Requests │                                                   │
│  └────┬─────┘                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                           │
│  │  Squid Proxy     │  1. Proxy request                        │
│  │  Port 3128       │  2. Generate log entry                   │
│  └────┬─────────────┘                                           │
│       │                                                         │
│       │ syslog() API call                                       │
│       │ (syslog:local5.info)                                    │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                           │
│  │   rsyslog        │  3. Receive log via syslog()             │
│  │   daemon         │  4. Format as RFC 5424                   │
│  │                  │  5. Buffer in memory queue               │
│  └────┬─────────────┘                                           │
│       │                                                         │
│       │ UDP packet                                              │
│       │ (port 10001)                                            │
│       ▼                                                         │
└───────┼─────────────────────────────────────────────────────────┘
        │
        │ Network (UDP over IP)
        │
        ▼
┌───────────────────────┐
│   Remote Syslog       │
│   Server              │
│   (S1 Collector)      │
│                       │
│   Receives RFC 5424   │
│   formatted logs      │
└───────────────────────┘
```

### Step-by-Step Data Journey

**Step 1: Client Makes Request**
- User browses to `http://www.example.com` through the proxy
- Squid processes the request

**Step 2: Squid Generates Log Entry**
- After completing the request, Squid generates an access log entry
- Because we configured `access_log syslog:local5.info squid`, Squid calls the system's `syslog()` function

**Step 3: syslog() API Call**
- Squid makes a function call: `syslog(LOG_LOCAL5 | LOG_INFO, "log message")`
- This is a **local function call** (not network) - VERY fast
- The log is handed off to rsyslog daemon

**Step 4: rsyslog Receives and Processes**
- rsyslog daemon receives the log entry
- Checks its configuration files in `/etc/rsyslog.d/`
- Finds our rule: `local5.* @YOUR_SYSLOG_IP:10001;RFC5424Format`
- Applies the RFC5424Format template
- Adds structured fields (timestamp, hostname, PID, etc.)

**Step 5: rsyslog Buffers**
- Log is placed in rsyslog's memory queue
- Queue prevents blocking - Squid doesn't wait for network send
- If remote server is slow/down, logs stay in queue

**Step 6: Network Transmission**
- rsyslog sends UDP packet to your remote syslog server
- UDP = fire-and-forget (no waiting for acknowledgment)
- Packet contains RFC 5424 formatted message

**Step 7: Remote Server Receives**
- Remote syslog server receives properly formatted RFC 5424 log
- Can parse it correctly because format matches specification
- Stores/processes log as configured

---

## Benefits of Using rsyslog as Intermediary

### 1. Format Conversion (Primary Benefit)

**Without rsyslog:**
```
Raw Squid format → Remote server (can't parse it properly)
```

**With rsyslog:**
```
Raw Squid format → rsyslog converts → RFC 5424 → Remote server (parses perfectly)
```

rsyslog acts as a **translator** between Squid's language and RFC 5424's language.

---

### 2. Spooling and Buffering (Reliability)

**What is spooling?**
Think of a printer queue - you send documents to print, and they wait in line. Even if the printer is busy, your documents don't disappear.

**How rsyslog spools logs:**

```
rsyslog Memory Queue (default):
┌─────────────────────────────┐
│ [Log 1] [Log 2] [Log 3] ... │  ← Logs wait here
└─────────────────────────────┘
            ↓
    Sent when network available
```

**Scenario: Network Hiccup**

Without rsyslog (Squid UDP direct):
```
Squid → Network down → Log LOST forever ❌
```

With rsyslog:
```
Squid → rsyslog queue → Network down → Log STAYS in queue ✅
                                    → Network up → Log SENT ✅
```

**Real world example:**
- Network congestion for 5 seconds
- Without rsyslog: All logs during those 5 seconds = gone
- With rsyslog: All logs buffered, sent once network recovers

---

### 3. Performance - Non-Blocking I/O

**Blocking vs Non-Blocking explained simply:**

**Blocking (like Squid TCP direct):**
```
1. Squid: "Here's a log!"
2. Squid: *waits for network to send*
3. Squid: *waits for server to acknowledge*
4. Server: "Got it!"
5. Squid: "OK, now I can process the next request"
```
⏱️ **Total time:** 50ms (wasted waiting on network)

**Non-Blocking (with rsyslog):**
```
1. Squid: "Here's a log!"
2. rsyslog: "Got it, I'll handle it"
3. Squid: "Cool, next request!"
```
⏱️ **Total time:** 0.1ms (just a local function call)

**Impact on performance:**
- **With rsyslog:** Squid can handle 10,000 requests/second
- **With direct TCP:** Squid might only handle 5,000 requests/second (slowed by network waits)

---

### 4. Flexibility and Configuration

**Change remote server without restarting Squid:**
```bash
# Edit rsyslog config
vi /etc/rsyslog.d/30-squid-forward.conf

# Change IP
local5.* @10.0.0.NEW_IP:10001;RFC5424Format

# Restart only rsyslog (Squid keeps running)
systemctl restart rsyslog
```

**Send to multiple destinations:**
```bash
# Add to /etc/rsyslog.d/30-squid-forward.conf
local5.* @SYSLOG_SERVER_1:10001;RFC5424Format
local5.* @SYSLOG_SERVER_2:10001;RFC5424Format
local5.* @SYSLOG_SERVER_3:10001;RFC5424Format
```
Now same logs go to 3 servers - no changes to Squid needed!

**Filter certain logs:**
```bash
# Only send successful requests (HTTP 200-299)
:msg, contains, "TCP_MISS/2" @YOUR_SYSLOG_IP:10001;RFC5424Format
```

---

### 5. Disk-Based Queues (Advanced Reliability)

**You can configure rsyslog to use disk queues instead of just memory:**

```bash
# Add to /etc/rsyslog.d/30-squid-forward.conf
$ActionQueueType LinkedList      # Use linked list for queue
$ActionQueueFileName squidqueue  # Store in /var/spool/rsyslog/
$ActionQueueMaxDiskSpace 1g      # Use up to 1GB disk space
$ActionQueueSaveOnShutdown on    # Save queue when rsyslog stops
$ActionResumeRetryCount -1       # Retry forever

local5.* @YOUR_SYSLOG_IP:10001;RFC5424Format
```

**Now even if:**
- Remote server is down for 2 hours
- Your server reboots
- Network cable unplugged

**Your logs are SAFE on disk and will be sent when possible!**

This is like having a backup generator - enterprise-grade reliability.

---

### 6. Built-in Log Management

**Automatic log rotation with logrotate:**

rsyslog integrates seamlessly with the system's logrotate service (pre-installed on Rocky Linux).

**What logrotate does automatically:**
- Rotates `/var/log/messages` daily (or when it reaches size limit)
- Compresses old logs (`.gz` files)
- Keeps last 4 weeks of logs by default
- Deletes very old logs automatically

**Example of rotated logs:**
```bash
ls -lh /var/log/messages*
```

**Output:**
```
-rw-------. 1 root root  2.1M Dec  3 10:15 /var/log/messages
-rw-------. 1 root root  8.2M Dec  2 03:22 /var/log/messages-20251202.gz
-rw-------. 1 root root  7.9M Nov 25 03:15 /var/log/messages-20251125.gz
-rw-------. 1 root root  8.1M Nov 18 03:28 /var/log/messages-20251118.gz
```

**Benefits:**
- No manual log management needed
- Disk space doesn't fill up
- Old logs are compressed (saves 80-90% space)
- Standard system tool - no custom scripts

**If you were logging directly to files with Squid:**
- You'd need to write custom rotation scripts
- Risk of filling up disk if you forget
- More complexity to maintain

---

## Understanding RFC 5424 Format

### Anatomy of an RFC 5424 Log Message

```
<174>1 2025-12-02T20:38:05.173191-05:00 squidproxy (squid-1) 13292 - - 1764725709.484 88 ::1 TCP_MISS/200...
└┬─┘│ └──────────┬──────────────────┘ └───┬────┘ └──┬───┘ └─┬──┘   │ │ └─────────┬──────────────────────┘
 │  │            │                        │          │      │      │ │           │
 │  │            │                        │          │      │      │ │           └─ Actual log message
 │  │            │                        │          │      │      │ └─ Structured data (empty here)
 │  │            │                        │          │      │      └─ Message ID (empty here)
 │  │            │                        │          │      └─ Process ID
 │  │            │                        │          └─ Application name
 │  │            │                        └─ Hostname
 │  │            └─ Timestamp (RFC 3339 format with timezone)
 │  └─ Version (always "1" for RFC 5424)
 └─ Priority (<Facility*8 + Severity>)
```

### Breaking Down Each Field

**1. Priority: `<174>`**
- Format: `<Facility * 8 + Severity>`
- Facility 21 (local5) * 8 + Severity 6 (info) = 174
- Tells receiver how important the message is and what category it belongs to

**2. Version: `1`**
- RFC 5424 version
- Always `1` for this specification
- Older RFC 3164 doesn't have a version number

**3. Timestamp: `2025-12-02T20:38:05.173191-05:00`**
- RFC 3339 format (ISO 8601 compliant)
- Includes microseconds: `.173191`
- Includes timezone: `-05:00` (EST)
- Remote server can convert to its local timezone

**4. Hostname: `squidproxy`**
- Identifies which server sent the log
- Useful when receiving from multiple servers
- Automatically filled by rsyslog

**5. App Name: `(squid-1)`**
- Name of the application that generated the log
- Squid uses `(squid-1)` for the first worker process
- If you ran multiple Squid workers, you'd see `(squid-2)`, `(squid-3)`, etc.

**6. Process ID: `13292`**
- The Linux process ID
- Run `ps aux | grep squid` to see matching PID
- Useful for correlating logs with specific process crashes

**7. Message ID: `-`**
- Optional identifier for the type of message
- Squid doesn't use this, so rsyslog puts `-`
- Other apps might use values like `TCPRS` or `AUTH_FAIL`

**8. Structured Data: `-`**
- Optional key-value pairs in format `[id key1="value1" key2="value2"]`
- Squid doesn't use this, so rsyslog puts `-`
- Example: `[exampleSDID@32473 iut="3" eventSource="Application" eventID="1011"]`

**9. Message: `1764725709.484 88 ::1 TCP_MISS/200...`**
- The actual Squid log entry
- Everything from here on is Squid's native access log format

---

## Understanding Syslog Facilities and Priorities

### What Are Facilities?

Facilities are like "channels" or "categories" for logs. Think of them like TV channels.

| Facility | Number | Typical Use |
|----------|--------|-------------|
| kern | 0 | Kernel messages |
| user | 1 | User-level messages |
| mail | 2 | Mail system |
| daemon | 3 | System daemons |
| auth | 4 | Security/authorization |
| syslog | 5 | Syslog itself |
| ... | ... | ... |
| **local0** | **16** | **Local use 0** |
| **local1** | **17** | **Local use 1** |
| **local2** | **18** | **Local use 2** |
| **local3** | **19** | **Local use 3** |
| **local4** | **20** | **Local use 4** |
| **local5** | **21** | **Local use 5** ← We use this |
| **local6** | **22** | **Local use 6** |
| **local7** | **23** | **Local use 7** |

**Why we chose local5:**
- `local0` through `local7` are reserved for custom applications
- Most systems don't use `local5` by default
- Easy to filter and route separately from system logs

### What Are Priorities (Severities)?

Priorities indicate how important a message is.

| Priority | Number | Description | Example |
|----------|--------|-------------|---------|
| emerg | 0 | System unusable | Kernel panic |
| alert | 1 | Action must be taken immediately | Database corruption |
| crit | 2 | Critical conditions | Hard disk failure |
| err | 3 | Error conditions | Application crash |
| warning | 4 | Warning conditions | Disk 90% full |
| notice | 5 | Normal but significant | Configuration change |
| **info** | **6** | **Informational** | **Normal log entry** ← We use this |
| debug | 7 | Debug messages | Verbose debugging |

**Why we chose info:**
- Squid access logs are informational (normal operation)
- Not errors, not warnings - just recording what happened
- Standard priority for access logs

### How Priority Number is Calculated

```
Priority Number = (Facility * 8) + Severity

Our configuration:
Facility: local5 = 21
Severity: info = 6

Calculation: (21 * 8) + 6 = 168 + 6 = 174

Result: <174>
```

---

## Performance Impact

### CPU Usage

**Squid without logging:**
- 100% processing requests

**Squid with syslog logging:**
- ~99.9% processing requests
- ~0.1% writing to syslog

**Impact:** Negligible. The `syslog()` call is extremely fast (microseconds).

### Memory Usage

**rsyslog memory queue:**
- Default: ~10MB for queue
- Each log entry: ~500 bytes average
- Can buffer ~20,000 logs in memory

**Total memory impact:** < 15MB additional RAM usage

### Network Usage

**Bandwidth calculation:**
- Average log entry: ~500 bytes
- 1000 requests/second = 500 KB/second = 4 Mbps
- On a gigabit network: 0.4% bandwidth usage

**Impact:** Minimal unless you're doing millions of requests/second.

---

## Reliability and Data Loss Prevention

### When Can Logs Be Lost?

**Scenario 1: rsyslog crashes**
- **Memory queue:** Logs in queue at crash time = LOST
- **Disk queue:** Logs in queue = SAVED (will send on restart)
- **Solution:** Use disk-based queue for critical environments

**Scenario 2: Remote server is down**
- **No queue:** Logs = LOST
- **With queue:** Logs buffered until server returns
- **Queue full:** Oldest logs dropped to make room for new ones
- **Solution:** Increase queue size or use disk queue

**Scenario 3: Network partition (your server can't reach remote server)**
- **No queue:** Logs = LOST
- **With queue:** Logs buffered for days if needed (with disk queue)
- **Solution:** Monitor queue size with `rsyslogd -n` or external monitoring

**Scenario 4: Squid crashes**
- Current request's log might not be written
- All previously written logs = SAFE
- **Impact:** Minimal (only last few milliseconds of logs)

### Improving Reliability

**Add disk-based queue:**
```bash
# In /etc/rsyslog.d/30-squid-forward.conf
# Add BEFORE the forwarding rule:

$ActionQueueType LinkedList
$ActionQueueFileName squidq
$ActionQueueMaxDiskSpace 2g
$ActionQueueSaveOnShutdown on
$ActionResumeRetryCount -1
$ActionQueueHighWatermark 80000
$ActionQueueLowWatermark 60000
```

**What this does:**
- `LinkedList`: Type of queue structure
- `squidq`: Filename prefix for queue files
- `2g`: Use up to 2 GB of disk for buffering
- `SaveOnShutdown`: Persist queue when rsyslog stops
- `ResumeRetryCount -1`: Retry forever
- `HighWatermark/LowWatermark`: When to start/stop using disk

**Now you can survive:**
- Days of network outage (2GB = ~4 million log entries)
- Remote server being down for maintenance
- Network reconfigurations
- Server reboots

---

## Comparison Table: Different Approaches

| Approach | Format | Reliability | Performance | Flexibility | Complexity |
|----------|--------|-------------|-------------|-------------|------------|
| `access_log stdio:/var/log/squid/access.log` | Squid native | ⭐⭐⭐⭐⭐ (local file) | ⭐⭐⭐⭐ (disk I/O) | ⭐ (local only) | ⭐⭐⭐⭐⭐ (simple) |
| `access_log udp://host:port` | Squid native | ⭐ (no buffer) | ⭐⭐⭐⭐⭐ (fire & forget) | ⭐⭐ (single dest) | ⭐⭐⭐⭐ (simple) |
| `access_log tcp://host:port` | Squid native | ⭐⭐ (no buffer, blocking) | ⭐⭐ (blocking) | ⭐⭐ (single dest) | ⭐⭐⭐⭐ (simple) |
| **`access_log syslog:local5.info` + rsyslog** | **RFC 5424** | **⭐⭐⭐⭐** (buffered) | **⭐⭐⭐⭐⭐** (non-blocking) | **⭐⭐⭐⭐⭐** (multi-dest, filtering) | **⭐⭐⭐** (moderate) |
| Syslog + disk queue | RFC 5424 | ⭐⭐⭐⭐⭐ (disk backed) | ⭐⭐⭐⭐ (disk writes) | ⭐⭐⭐⭐⭐ (multi-dest, filtering) | ⭐⭐ (complex) |

**Recommended:** The approach we implemented (syslog + rsyslog) provides the best balance of RFC 5424 format compliance, reliability, performance, and flexibility.

---

## Summary

**The rsyslog intermediary approach wins because:**

1. ✅ **Correct Format:** Produces RFC 5424 format required by remote server
2. ✅ **Reliable:** Buffers logs during network issues
3. ✅ **Fast:** Non-blocking, doesn't slow down Squid
4. ✅ **Flexible:** Easy to change destinations, add filters, modify formats
5. ✅ **Enterprise-Ready:** Can add disk queues for maximum reliability
6. ✅ **Built-in Log Management:** Automatic rotation and compression via logrotate
7. ✅ **Standard:** Uses operating system facilities, not custom solutions
8. ✅ **Maintainable:** Standard rsyslog configuration, lots of documentation

**This is why rsyslog is the recommended approach for production environments!**
