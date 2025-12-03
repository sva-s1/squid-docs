# Squid Proxy: Remote Syslog Logging Documentation

![Rocky Linux](https://img.shields.io/badge/Rocky%20Linux-9.7-10B981?style=flat-square&logo=rockylinux&logoColor=white)
![Squid](https://img.shields.io/badge/Squid-5.5-0088CC?style=flat-square&logo=squid&logoColor=white)
![rsyslog](https://img.shields.io/badge/rsyslog-8.x-FF6B6B?style=flat-square&logo=linux&logoColor=white)
![RFC 5424](https://img.shields.io/badge/RFC%205424-Compliant-4CAF50?style=flat-square&logo=rfc&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Beginner-green?style=flat-square)
![Time](https://img.shields.io/badge/Setup%20Time-15--20%20min-orange?style=flat-square&logo=clockify&logoColor=white)

**Complete guide for shipping Squid proxy logs to remote syslog server in RFC 5424 format**

![KRAGLE](src/images/thekragle.jpg)

---

## What's This About?

This documentation set explains how to configure Squid proxy server (version 5.5) to send access logs to a remote syslog server using:
- **Transport:** UDP
- **Port:** 10001 (customizable)
- **Format:** RFC 5424 (industry standard syslog format)

**The solution:** Use rsyslog as an intermediary to convert Squid's native log format to RFC 5424 and forward to your remote destination.

---

## Documentation Files

This guide is split into multiple focused documents. Start with the one that matches your needs:

### 1. 📘 [SQUID_LOGGING_SETUP.md](SQUID_LOGGING_SETUP.md)
**START HERE** - Step-by-step setup instructions

**For:** Setting up the configuration from scratch
**Level:** Beginner-friendly (5th grade level)
**Time:** 15-20 minutes

**You'll learn:**
- How to configure Squid to log to local syslog
- How to configure rsyslog to forward logs in RFC 5424 format
- How to test your configuration
- How to verify logs are being sent

---

### 2. 🧠 [WHY_THIS_WORKS.md](WHY_THIS_WORKS.md)
**Understanding the architecture**

**For:** People who want to understand the "why" behind the setup
**Level:** Beginner to Intermediate

**You'll learn:**
- Why Squid can't send RFC 5424 format directly
- Why we use rsyslog as an intermediary
- Benefits of this architecture (performance, reliability, flexibility, log management)
- Detailed explanation of RFC 5424 format
- How syslog facilities and priorities work

---

### 3. 🔧 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
**Problem solving guide**

**For:** When something isn't working correctly
**Level:** All levels

**You'll learn:**
- How to diagnose common problems
- Solutions for Squid not starting
- Solutions for logs not appearing
- Solutions for logs not reaching remote server
- Network troubleshooting techniques
- Performance issue fixes

---

### 4. ⚡ [QUICK_REFERENCE.md](QUICK_REFERENCE.md)
**Command cheat sheet**

**For:** Quick lookup of common commands
**Level:** All levels

**You'll find:**
- Service management commands
- Testing commands
- Monitoring commands
- Common configuration changes
- One-liner commands
- File locations reference

---

### 5. 📋 THIS FILE (README.md)
**Navigation and overview**

You're reading it right now!

---

## Quick Start (30 Seconds)

**Already have Squid installed? Just want to get it working?**

```bash
# 1. Configure Squid
echo "access_log syslog:local5.info squid" >> /etc/squid/squid.conf

# 2. Create rsyslog config
cat > /etc/rsyslog.d/30-squid-forward.conf <<'EOF'
template(name="RFC5424Format" type="string"
  string="<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n"
)
local5.* @YOUR_SYSLOG_IP:10001;RFC5424Format
EOF

# 3. Restart services
systemctl restart rsyslog squid

# 4. Test
curl -x http://localhost:3128 http://www.example.com/ -I

# 5. Verify
timeout 10 tcpdump -i any -nn -A 'udp and dst port 10001' -c 1
```

**For detailed explanation of what each step does, see SQUID_LOGGING_SETUP.md**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     YOUR SERVER (Rocky 9.7)                     │
│                                                                 │
│  ┌────────────┐                                                 │
│  │   Squid    │  Processes proxy requests                       │
│  │  Port 3128 │  Generates access logs                          │
│  └─────┬──────┘                                                 │
│        │ syslog:local5.info                                     │
│        ▼                                                        │
│  ┌────────────┐                                                 │
│  │  rsyslog   │  Receives logs from Squid                       │
│  │   daemon   │  Converts to RFC 5424 format                    │
│  └─────┬──────┘  Buffers and forwards                           │
│        │                                                        │
└────────┼────────────────────────────────────────────────────────┘
         │ UDP Port 10001
         │ RFC 5424 Format
         ▼
   ┌──────────────────┐
   │  Remote Syslog   │
   │  (S1 Collector)  │
   │  Port 10001      │
   └──────────────────┘
```

**Key insight:** Squid → rsyslog → Remote server
- Squid sends to LOCAL syslog (fast, non-blocking)
- rsyslog converts to RFC 5424 and forwards (reliable, buffered)

---

## What You Need

### Already Installed
- ✅ Rocky Linux 9.7
- ✅ Squid 5.5
- ✅ rsyslog (comes with Rocky Linux)

### Need to Install (for troubleshooting)
```bash
dnf install -y tcpdump nc
```

---

## Configuration Files Created

After following the setup guide, you'll have:

| File | Purpose |
|------|---------|
| `/etc/squid/squid.conf` | Modified: Added `access_log syslog:local5.info squid` |
| `/etc/rsyslog.d/30-squid-forward.conf` | Created: RFC 5424 template and forwarding rule |

**That's it!** Only 2 files modified/created.

---

## How to Use This Documentation

### Scenario 1: First Time Setup
**Path:** README.md (you are here) → SQUID_LOGGING_SETUP.md → Test it → Done!

Optional: Read WHY_THIS_WORKS.md to understand the architecture

---

### Scenario 2: Something's Not Working
**Path:** README.md → TROUBLESHOOTING.md → Find your symptom → Apply solution

Use QUICK_REFERENCE.md for command lookups

---

### Scenario 3: Need to Change Configuration
**Path:** QUICK_REFERENCE.md → "Common Modifications" section

Example: Changing remote server IP, adding second destination, etc.

---

### Scenario 4: Want to Understand How It Works
**Path:** WHY_THIS_WORKS.md → Detailed architecture explanation

Learn about RFC 5424, syslog facilities, performance implications, etc.

---

### Scenario 5: Daily Operations
**Path:** QUICK_REFERENCE.md → Keep it open for command lookups

Common tasks: Checking logs, restarting services, monitoring, etc.

---

## Recommended Reading Order

**For beginners:**
1. 📘 SQUID_LOGGING_SETUP.md (follow step-by-step)
2. ⚡ QUICK_REFERENCE.md (bookmark for daily use)
3. 🧠 WHY_THIS_WORKS.md (when you want to understand deeper)
4. 🔧 TROUBLESHOOTING.md (when needed)

**For experienced admins:**
1. 🧠 WHY_THIS_WORKS.md (understand architecture first)
2. 📘 SQUID_LOGGING_SETUP.md (skim for specifics)
3. ⚡ QUICK_REFERENCE.md (bookmark for daily use)
4. 🔧 TROUBLESHOOTING.md (reference as needed)

---

## Key Concepts Explained

### What is RFC 5424?
Standardized syslog message format that includes structured fields like timestamp, hostname, application name, process ID, etc.

**Example:**
```
<174>1 2025-12-02T20:38:05.173191-05:00 squidproxy (squid-1) 13292 - - [log message]
```

See WHY_THIS_WORKS.md for detailed breakdown.

---

### What is a Syslog Facility?
Think of it like a TV channel. Different facilities (channels) are for different types of logs.
- `local5` = Custom application channel #5 (we use this for Squid)

See WHY_THIS_WORKS.md for complete facility table.

---

### Why Not Use Squid's UDP Module?
```bash
# This doesn't work for RFC 5424:
access_log udp://10.0.0.241:10001 squid
```

**Reason:** Squid's UDP module sends logs in Squid's native format, NOT RFC 5424 format.

See WHY_THIS_WORKS.md for detailed explanation.

---

## Testing Your Setup

### Quick Test (30 seconds)
```bash
# Generate traffic
curl -x http://localhost:3128 http://www.example.com/ -I

# Check local logs
grep squid /var/log/messages | tail -1

# Check network packets
timeout 5 tcpdump -i any -nn 'udp and dst port 10001' -c 1
```

### Comprehensive Test
See "Testing Your Configuration" section in SQUID_LOGGING_SETUP.md

---

## Common Questions

**Q: Will this work with Squid 6.x or 7.x?**
A: Yes, the syslog module is standard across all Squid versions.

**Q: Can I send to multiple remote servers?**
A: Yes! Add multiple lines to `/etc/rsyslog.d/30-squid-forward.conf`. See QUICK_REFERENCE.md "Common Modifications" section.

**Q: What if the remote server is down?**
A: Logs are buffered in rsyslog's queue. Add disk-based queue for maximum reliability. See [WHY_THIS_WORKS.md](WHY_THIS_WORKS.md) "Reliability" section.

**Q: Does this impact Squid performance?**
A: Minimal impact (<0.1% CPU). The syslog() call is extremely fast. See [WHY_THIS_WORKS.md](WHY_THIS_WORKS.md) "Performance Impact" section.

**Q: Can I use TCP instead of UDP?**
A: Yes, change `@` to `@@` in the rsyslog config. See [QUICK_REFERENCE.md](QUICK_REFERENCE.md) "Change to TCP" section.

**Q: How do I change the destination IP/port?**
A: Edit `/etc/rsyslog.d/30-squid-forward.conf`, change the IP/port, restart rsyslog. See [QUICK_REFERENCE.md](QUICK_REFERENCE.md) "Common Modifications" section.

---

## Getting Help

1. ✅ Check TROUBLESHOOTING.md for your specific problem
2. ✅ Run the diagnostic report generator (in TROUBLESHOOTING.md)
3. ✅ Review WHY_THIS_WORKS.md to understand the architecture
4. ✅ Check official docs:
   - Squid: https://wiki.squid-cache.org
   - rsyslog: https://www.rsyslog.com/doc/

---

## Important Notes

### Default Behavior After Reboot
Both Squid and rsyslog are configured to start automatically on boot. Your logging setup will survive reboots.

**Verify:**
```bash
systemctl is-enabled squid rsyslog
```

Both should show `enabled`.

---

### Log Rotation
The local `/var/log/messages` file is automatically rotated by `logrotate`. Squid logs won't fill up your disk.

---

### Security Considerations
- UDP is unencrypted - logs are sent in plain text over the network
- If you need encryption, use TLS-enabled syslog (not covered in this guide)
- Firewall should allow outbound UDP to your remote syslog server IP and port

---

### Performance at Scale
- Default configuration handles up to 10,000 requests/second easily
- For higher volume, consider disk-based queues (see WHY_THIS_WORKS.md)
- For millions of requests/second, consider log sampling or dedicated log collectors

---

## File Checksums (For Verification)

After setup, you can verify your config files:

```bash
# Check if your rsyslog config matches expected format
wc -l /etc/rsyslog.d/30-squid-forward.conf
# Should show: 10 lines (including comments)

# Verify Squid has syslog logging enabled
grep -c "access_log syslog" /etc/squid/squid.conf
# Should show: 1 (or more if you have multiple access_log lines)
```

---

## Version Information

**Created for:**
- Rocky Linux 9.7
- Squid 5.5
- rsyslog 8.2506.0

**Compatible with:**
- RHEL 9.x
- CentOS Stream 9
- AlmaLinux 9.x
- Oracle Linux 9.x
- Squid 4.x, 5.x, 6.x
- rsyslog 8.x+

---

## What's Next?

**After successful setup:**

1. **Customize** - Modify remote server IP, add filtering, etc. (see QUICK_REFERENCE.md)
2. **Monitor** - Set up monitoring for Squid and rsyslog services
3. **Optimize** - Add disk queues if needed (see WHY_THIS_WORKS.md)
4. **Secure** - Consider TLS if transmitting over untrusted networks
5. **Document** - Note your customizations for future reference

---

## License and Support

This documentation is provided as-is for educational and operational purposes.

**Support:** These are community-maintained guides. For production support, consult your organization's support channels.

---

## Summary

**What we've accomplished:**
✅ Squid sends logs to local syslog using syslog module
✅ rsyslog converts logs to RFC 5424 format
✅ rsyslog forwards logs to remote server via UDP on port 10001
✅ Reliable, performant, and flexible logging architecture

**Total config changes:** 2 files
**Total downtime:** < 30 seconds (service restarts)
**Complexity:** Low to Moderate
**Reliability:** High (with buffering)

---

**Ready to get started?**

👉 Start with [SQUID_LOGGING_SETUP.md](SQUID_LOGGING_SETUP.md) for step-by-step instructions

👉 Or jump to [QUICK_REFERENCE.md](QUICK_REFERENCE.md) for quick commands

Good luck! 🚀
