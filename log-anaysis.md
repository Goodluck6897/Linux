# Linux Log Analysis — RHEL 8/9 L3 Interview Notes

> **L3 troubleshooting mindset:**
> **Problem → Identify log → Filter → Understand error → Correlate → Fix → Verify**

---

## 1. Important Log Locations

On **RHEL 8/9**, know these locations:

| Log                        | Purpose                        |
| -------------------------- | ------------------------------ |
| `/var/log/messages`        | General system messages        |
| `/var/log/secure`          | SSH, authentication, sudo, PAM |
| `/var/log/maillog`         | Mail-related logs              |
| `/var/log/cron`            | Cron-related logs              |
| `/var/log/boot.log`        | Boot messages                  |
| `/var/log/dnf.log`         | DNF/package operations         |
| `/var/log/audit/audit.log` | Security/SELinux audit logs    |
| `/var/log/`                | General log directory          |

### Important

RHEL 8/9 uses:

```text
systemd
   |
   +-- journald
   |
   +-- journalctl
```

Therefore, **`journalctl` is one of the most important commands for modern RHEL troubleshooting.**

---

# 2. Traditional Log Analysis

## View a log

```bash
cat /var/log/messages
```

Better for large files:

```bash
less /var/log/messages
```

## Last 50 lines

```bash
tail -50 /var/log/messages
```

## Follow log in real time

```bash
tail -f /var/log/messages
```

Very commonly used during live troubleshooting.

---

# 3. Search Logs with grep

## Search for errors

```bash
grep -i "error" /var/log/messages
```

## Search multiple keywords

```bash
grep -Ei "error|failed|failure|critical" /var/log/messages
```

## Search SSH failures

```bash
grep -i "failed" /var/log/secure
```

Example:

```text
Failed password for invalid user admin from 10.10.10.20
```

You can identify:

* Username
* Source IP
* Authentication failure
* Possible brute-force activity

---

# 4. Understanding Log Timestamps

Example:

```text
Sep 10 13:40:21 server1 sshd[12345]: Failed password for user1
```

Breakdown:

```text
Sep 10       → Date
13:40:21     → Time
server1      → Hostname
sshd         → Process/service
[12345]      → PID
Failed ...   → Log message
```

## Why timestamps matter

Suppose you see:

```text
13:40:20  Disk I/O error
13:40:21  Filesystem error
13:40:22  Application stopped
```

The timeline suggests the application failure may have been caused by the underlying disk/filesystem problem.

### L3 approach

Always correlate:

```text
Application logs
      +
Service logs
      +
System logs
      +
Kernel logs
      +
Security logs
      +
Resource metrics
```

---

# 5. journalctl

`journalctl` is the primary command for querying the **systemd journal**.

## View all journal logs

```bash
journalctl
```

## Show latest logs

```bash
journalctl -n
```

or:

```bash
journalctl -n 50
```

## Follow logs in real time

```bash
journalctl -f
```

Equivalent conceptually to:

```bash
tail -f
```

---

# 6. Logs for a Specific Service

Suppose Apache/httpd has a problem.

## View httpd logs

```bash
journalctl -u httpd
```

## Last 100 lines

```bash
journalctl -u httpd -n 100
```

## Follow in real time

```bash
journalctl -u httpd -f
```

## Today's logs

```bash
journalctl -u httpd --since today
```

## Specific time range

```bash
journalctl -u httpd \
  --since "13:00" \
  --until "14:00"
```

### ⭐ Interview command

```bash
journalctl -u <service> --since "30 minutes ago"
```

This is extremely useful for investigating a recent service failure.

---

# 7. Boot Troubleshooting

## Current boot

```bash
journalctl -b
```

## Previous boot

```bash
journalctl -b -1
```

## Errors from current boot

```bash
journalctl -b -p err
```

## Warnings and more severe messages

```bash
journalctl -p warning
```

### Journal priority levels

| Priority | Meaning       |
| -------: | ------------- |
|        0 | Emergency     |
|        1 | Alert         |
|        2 | Critical      |
|        3 | Error         |
|        4 | Warning       |
|        5 | Notice        |
|        6 | Informational |
|        7 | Debug         |

For example:

```bash
journalctl -p err
```

shows **error and more severe messages**.

---

# 8. Kernel Log Analysis

Use:

```bash
dmesg
```

Human-readable timestamps:

```bash
dmesg -T
```

## Search for errors

```bash
dmesg -T | grep -i error
```

## Disk/storage problems

```bash
dmesg -T | grep -Ei "disk|nvme|scsi|I/O"
```

## Memory/OOM problems

```bash
dmesg -T | grep -Ei "oom|out of memory"
```

Example:

```text
Out of memory: Killed process 1234 (java)
```

This indicates that the **kernel OOM killer terminated the Java process**.

---

# 9. SSH Troubleshooting

Suppose a user cannot SSH to a server.

## Step 1 — Check sshd

```bash
systemctl status sshd
```

## Step 2 — Check service logs

```bash
journalctl -u sshd
```

## Step 3 — Check authentication logs

```bash
tail -f /var/log/secure
```

## Step 4 — Search authentication failures

```bash
grep -i "failed" /var/log/secure
```

## Step 5 — Search successful logins

```bash
grep -i "accepted" /var/log/secure
```

### Troubleshooting flow

```text
             SSH connection failed
                     |
                     v
             systemctl status sshd
                     |
                     v
              journalctl -u sshd
                     |
                     v
              /var/log/secure
                     |
                     v
       Check authentication / PAM
                     |
                     v
              Check permissions
                     |
                     v
              Check firewall
                     |
                     v
          Check network connectivity
```

Useful network check:

```bash
ss -lntp | grep :22
```

---

# 10. Application Log Analysis

Suppose a Java application writes logs to:

```text
/opt/myapp/logs/application.log
```

## Monitor the log

```bash
tail -f /opt/myapp/logs/application.log
```

## Search exceptions

```bash
grep -i "exception" /opt/myapp/logs/application.log
```

## Search common failure keywords

```bash
grep -Ei "error|exception|failed|timeout" \
/opt/myapp/logs/application.log
```

---

# 11. Java Application Managed by systemd

Suppose the application has:

```text
myapp.service
```

View logs:

```bash
journalctl -u myapp
```

Follow in real time:

```bash
journalctl -u myapp -f
```

Recent logs:

```bash
journalctl -u myapp --since "30 minutes ago"
```

Errors:

```bash
journalctl -u myapp -p err
```

### Important concept

`journalctl` does **not automatically collect every application's log file**.

An application can write logs to:

```text
/opt/myapp/logs/application.log
```

or to the systemd journal.

For example, a systemd-managed service may send its stdout/stderr to journald.

Therefore:

```text
Application
     |
     +----> application.log
     |
     +----> stdout/stderr
                  |
                  v
              journald
                  |
                  v
             journalctl
```

---

# 12. SELinux Log Analysis

SELinux troubleshooting is very important on RHEL.

## Check SELinux status

```bash
getenforce
```

Possible output:

```text
Enforcing
Permissive
Disabled
```

## Audit log

```bash
/var/log/audit/audit.log
```

## Search AVC denials

```bash
grep AVC /var/log/audit/audit.log
```

Better:

```bash
ausearch -m AVC -ts recent
```

You can also use:

```bash
sealert -a /var/log/audit/audit.log
```

if the `setroubleshoot-server` tools are installed.

### Typical SELinux troubleshooting

```text
Application cannot access file
             |
             v
Check normal permissions
             |
             v
Permissions look correct
             |
             v
Check SELinux
             |
             v
ausearch / audit.log
             |
             v
AVC denial found
             |
             v
Check SELinux context
             |
             v
restorecon / semanage
             |
             v
Retest application
```

---

# 13. Log Rotation

Logs should not grow indefinitely.

RHEL commonly uses:

```text
logrotate
```

Main configuration:

```bash
/etc/logrotate.conf
```

Additional configurations:

```bash
/etc/logrotate.d/
```

List configurations:

```bash
ls -l /etc/logrotate.d/
```

You may find configurations for services such as:

```text
httpd
sssd
syslog
```

Example rotated logs:

```text
messages
messages-20260901
messages-20260908
```

Compressed logs:

```text
messages-20260825.gz
```

## Search compressed logs

```bash
zgrep "error" messages-20260825.gz
```

---

# 14. Useful Log Analysis Commands

### journalctl

```bash
journalctl
journalctl -f
journalctl -n 50
journalctl -u <service>
journalctl -u <service> -f
journalctl -u <service> -n 100
journalctl -u <service> --since today
journalctl -u <service> --since "1 hour ago"
journalctl -b
journalctl -b -1
journalctl -b -p err
journalctl -p warning
journalctl -p err
```

### Traditional logs

```bash
tail -f /var/log/messages
tail -50 /var/log/messages
grep -i error /var/log/messages
grep -i failed /var/log/secure
```

### Kernel

```bash
dmesg -T
dmesg -T | grep -i error
dmesg -T | grep -Ei "disk|nvme|scsi|I/O"
dmesg -T | grep -Ei "oom|out of memory"
```

### SELinux

```bash
getenforce
grep AVC /var/log/audit/audit.log
ausearch -m AVC -ts recent
```

### Compressed logs

```bash
zgrep "error" *.gz
```

---

# 15. Real-Time L3 Troubleshooting Scenario

### Interviewer:

> Users report that the application suddenly stopped working. How would you troubleshoot?

### Strong L3 Answer

I would troubleshoot systematically rather than immediately checking one log file.

### Step 1 — Check application/service status

```bash
systemctl status myapp
```

### Step 2 — Check recent application logs

```bash
journalctl -u myapp --since "30 minutes ago"
```

### Step 3 — Check errors

```bash
journalctl -u myapp -p err
```

### Step 4 — Check system-level errors

```bash
journalctl -p err --since "30 minutes ago"
```

### Step 5 — Check kernel messages

```bash
dmesg -T | tail -100
```

### Step 6 — Check disk space

```bash
df -h
```

### Step 7 — Check inode usage

```bash
df -i
```

### Step 8 — Check memory

```bash
free -h
```

### Step 9 — Check CPU/load

```bash
uptime
top
```

### Step 10 — Check listening ports

```bash
ss -lntp
```

### Step 11 — Test the application

```bash
curl -v http://localhost:8080
```

### Step 12 — Correlate timestamps

Compare:

```text
Application logs
       +
systemd journal
       +
Kernel logs
       +
Security/SELinux logs
       +
Resource information
       +
Network information
```

Then:

```text
Identify root cause
       ↓
Apply corrective action
       ↓
Restart/recover if required
       ↓
Verify application
       ↓
Monitor
       ↓
Document root cause
```

---

# 16. L3 Troubleshooting Decision Tree

```text
              Application Problem
                     |
                     v
            Is service running?
               /          \
             NO            YES
             |              |
             v              v
      systemctl status    Check logs
             |              |
             v              v
      journalctl -u      Application
             |              |
             |              v
             |         System logs
             |              |
             |              v
             |          Kernel logs
             |              |
             +------+-------+
                    |
                    v
             Check resources
                    |
          +---------+---------+
          |         |         |
         CPU       RAM       Disk
          |         |         |
         top      free -h    df -h
                             df -i
                    |
                    v
              Check network
                    |
                    v
                ss / curl
                    |
                    v
             Check SELinux
                    |
                    v
            audit.log / AVC
                    |
                    v
              Find root cause
                    |
                    v
                  Fix
                    |
                    v
                Verify
```

---

# 17. Common L3 Log Scenarios

## Scenario 1 — Application stopped

Check:

```bash
systemctl status myapp
journalctl -u myapp --since "30 minutes ago"
journalctl -p err --since "30 minutes ago"
```

Then check:

```bash
df -h
free -h
dmesg -T
```

---

## Scenario 2 — Server suddenly rebooted

Check:

```bash
last reboot
```

Then:

```bash
journalctl -b -1
journalctl -b -1 -p err
```

Also:

```bash
dmesg -T
```

Look for:

* Kernel panic
* OOM
* Hardware problems
* Filesystem errors
* Power-related events
* Manual reboot
* Watchdog events

---

## Scenario 3 — Disk full

Check:

```bash
df -h
```

Then:

```bash
du -xhd1 /var
```

Check logs:

```bash
journalctl --disk-usage
```

Look for:

```text
/var/log
application logs
core dumps
temporary files
old packages
```

Also check inodes:

```bash
df -i
```

---

## Scenario 4 — SSH authentication failure

Check:

```bash
journalctl -u sshd
grep -i "failed" /var/log/secure
```

Investigate:

```text
Username
Source IP
PAM errors
Authentication method
Account status
Permissions
SELinux
Firewall
```

---

## Scenario 5 — Application cannot access a file

Check normal permissions:

```bash
ls -l /path/to/file
```

Check ownership:

```bash
ls -ld /path/to/directory
```

Check SELinux context:

```bash
ls -Z /path/to/file
```

Search AVC:

```bash
ausearch -m AVC -ts recent
```

---

# 18. Must-Know Commands for Interview

Memorize these first:

```bash
systemctl status <service>

journalctl -u <service>

journalctl -u <service> -f

journalctl -u <service> --since "30 minutes ago"

journalctl -p err

journalctl -b

journalctl -b -1

dmesg -T

tail -f /var/log/messages

grep -Ei "error|failed|failure" /var/log/messages

grep -i failed /var/log/secure

ausearch -m AVC -ts recent

df -h

df -i

free -h

ss -lntp
```

---

# 19. ⭐ Interview Mindset

### Weak answer

> "I will check `/var/log/messages`."

### Strong L3 answer

> "First I will determine whether the issue is application, service, OS, resource, security, storage, or network related. Then I will check the relevant service-specific journal and system logs, correlate events using timestamps, validate CPU, memory, filesystem and network health, identify the root cause, apply the corrective action, and finally verify that the service is healthy."

---

# 20. One-Minute Interview Revision

```text
LOG ANALYSIS
     |
     +-- Traditional logs
     |      |
     |      +-- /var/log/messages
     |      +-- /var/log/secure
     |      +-- /var/log/cron
     |      +-- /var/log/maillog
     |      +-- /var/log/audit/audit.log
     |
     +-- systemd
     |      |
     |      +-- journalctl
     |      +-- journalctl -u service
     |      +-- journalctl -f
     |      +-- journalctl -b
     |      +-- journalctl -b -1
     |      +-- journalctl -p err
     |
     +-- Kernel
     |      |
     |      +-- dmesg -T
     |      +-- OOM
     |      +-- Disk/I/O
     |      +-- Hardware
     |
     +-- SELinux
     |      |
     |      +-- ausearch
     |      +-- AVC
     |      +-- audit.log
     |
     +-- Log rotation
     |      |
     |      +-- logrotate
     |      +-- /etc/logrotate.conf
     |      +-- /etc/logrotate.d/
     |
     +-- Troubleshooting
            |
            +-- Identify
            +-- Filter
            +-- Correlate
            +-- Root cause
            +-- Fix
            +-- Verify
```

---

## ⭐ Golden Rule

> **Don't just read logs — correlate them.**

A good Linux L3 administrator connects:

```text
Application
    ↓
Service
    ↓
System
    ↓
Kernel
    ↓
Storage / Network / Security
    ↓
Root Cause
```

That is the difference between **L1/L2 log checking** and **L3 troubleshooting**.
