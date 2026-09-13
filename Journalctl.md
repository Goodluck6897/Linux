journalctl is very important because it is the main command used to query logs managed by systemd/journald.

journalctl -u sshd - past events, audit old login attempts, or debug an issue that happened hours or days ago.
journalctl -u sshd -f - Use this when you want to monitor live activity, watch a user attempt to log in right now, or troubleshoot a connection in real-time.


# 1. Service status
systemctl status <service>

# 2. Service logs
journalctl -u <service>

# 3. Recent logs
journalctl -u <service> -n 100

# 4. Live logs
journalctl -u <service> -f

# 5. Today's logs
journalctl -u <service> --since today

# 6. Errors
journalctl -u <service> -p err

# 7. Current boot
journalctl -b

# 8. Previous boot
journalctl -b -1

# 9. Kernel errors
journalctl -k -p err

# 10. Search
journalctl | grep -i error

7. Logs for a particular process

If you know the PID:
#journalctl _PID=1234

8. Logs for a particular user
   journalctl _UID=1000

14. Disk usage of journal
    journalctl --disk-usage

15. Clean old journal logs
    For example, keep only 7 days: #journalctl --vacuum-time=7d
    Keep only 500 MB:#journalctl --vacuum-size=500M

16. Check journald service
    #systemctl status systemd-journald
    Check whether it is running:
    #systemctl is-active systemd-journald
    
--
```
journalctl -u sshd
journalctl -u sshd -f
journalctl -u sshd --since today
journalctl -p err
journalctl -b -1
journalctl -k -p err
```

  journalctl does not automatically collect every Java application's log file.
It shows logs that are sent to systemd-journal

1. Java application runs as a systemd service — most common

Suppose you have:

/opt/myapp/myapp.jar

and a systemd unit:
```
[Unit]
Description=My Java Application
After=network.target

[Service]
ExecStart=/usr/bin/java -jar /opt/myapp/myapp.jar
Restart=always

[Install]
WantedBy=multi-user.target
```

When Java writes to stdout/stderr:
```
System.out.println("Application started");
System.err.println("Database connection failed");
```
systemd captures that output and sends it to journald.

Then:

journalctl -u myapp


###
For a **Linux L3 interview**, `journalctl` is very important because it is the main command used to query logs managed by **systemd/journald**.

## 1. Basic `journalctl`

```bash
journalctl
```

Shows all available journal logs.

```bash
journalctl -n
```

Shows the latest 10 log entries.

```bash
journalctl -n 50
```

Shows the latest 50 entries.

```bash
journalctl -f
```

**Follow logs live**, similar to:

```bash
tail -f /var/log/messages
```

Press `Ctrl+C` to exit.

---

## 2. Logs for a specific service

This is one of the most important interview commands.

```bash
journalctl -u sshd
```

Logs for SSH service.

```bash
journalctl -u sshd -f
```

Follow SSH logs in real time.

```bash
journalctl -u sshd -n 100
```

Last 100 SSH log entries.

```bash
journalctl -u nginx --since today
```

Today's nginx logs.

Multiple services:

```bash
journalctl -u nginx -u sshd
```

---

## 3. Logs since a specific time

```bash
journalctl --since "1 hour ago"
```

```bash
journalctl --since "30 minutes ago"
```

```bash
journalctl --since today
```

```bash
journalctl --since yesterday
```

Specific date/time:

```bash
journalctl --since "2026-09-10 09:00:00"
```

Time range:

```bash
journalctl --since "2026-09-10 09:00:00" \
           --until "2026-09-10 10:00:00"
```

Very useful for troubleshooting:

> "The application failed around 9:30 AM. Show me what happened between 9:20 and 9:40."

```bash
journalctl --since "09:20" --until "09:40"
```

---

## 4. Boot-related logs

### Current boot

```bash
journalctl -b
```

Shows logs from the current boot.

### Previous boot

```bash
journalctl -b -1
```

### Two boots ago

```bash
journalctl -b -2
```

### List available boots

```bash
journalctl --list-boots
```

Example:

```text
-2  abc123  Mon ...
-1  def456  Tue ...
 0  ghi789  Wed ...
```

This is **very useful after a server reboot**.

For example:

```bash
journalctl -b -1 -p err
```

Find errors from the **previous boot**.

---

# 5. Priority / severity

`journalctl` supports log priorities:

| Priority | Meaning |
| -------- | ------- |
| `0`      | emerg   |
| `1`      | alert   |
| `2`      | crit    |
| `3`      | err     |
| `4`      | warning |
| `5`      | notice  |
| `6`      | info    |
| `7`      | debug   |

### Only errors

```bash
journalctl -p err
```

### Errors from current boot

```bash
journalctl -b -p err
```

### Errors and above

```bash
journalctl -p 0..3
```

### Warnings and errors

```bash
journalctl -p warning..err
```

---

# 6. Kernel logs

```bash
journalctl -k
```

Shows kernel messages.

Equivalent conceptually to:

```bash
dmesg
```

Current boot kernel errors:

```bash
journalctl -k -b -p err
```

Very useful for:

* disk problems
* filesystem errors
* network interface problems
* driver problems
* kernel issues

---

# 7. Logs for a particular process

If you know the PID:

```bash
journalctl _PID=1234
```

Example:

```bash
journalctl _PID=5678
```

---

# 8. Logs for a particular user

```bash
journalctl _UID=1000
```

Useful when troubleshooting user-specific processes.

---

# 9. Search logs

You can pipe to `grep`:

```bash
journalctl | grep -i error
```

For example:

```bash
journalctl -u sshd | grep -i failed
```

Better:

```bash
journalctl -u sshd --since today | grep -i failed
```

---

# 10. Show logs in reverse order

```bash
journalctl -r
```

Newest entries first.

Very useful when you want to see the latest event immediately.

---

# 11. Don't use a pager

Normally:

```bash
journalctl
```

opens in `less`.

Use:

```bash
journalctl --no-pager
```

For example:

```bash
journalctl -u sshd --no-pager
```

This is useful for scripts and troubleshooting.

---

# 12. Show detailed log fields

```bash
journalctl -o verbose
```

This shows fields such as:

```text
_PID=
_UID=
_SYSTEMD_UNIT=
_COMM=
_EXE=
_HOSTNAME=
_MESSAGE=
```

Useful when doing advanced troubleshooting.

---

# 13. Show only the message

```bash
journalctl -o cat
```

Instead of displaying all metadata, it shows mainly the actual log message.

---

# 14. Disk usage of journal

```bash
journalctl --disk-usage
```

Example:

```text
Archived and active journals take up 1.2G in the file system.
```

---

# 15. Clean old journal logs

For example, keep only 7 days:

```bash
journalctl --vacuum-time=7d
```

Keep only 500 MB:

```bash
journalctl --vacuum-size=500M
```

Keep only 10 archived files:

```bash
journalctl --vacuum-files=10
```

**Interview point:** Don't randomly delete `/var/log/journal`. Use `journalctl --vacuum-*` for journal cleanup.

---

# 16. Check journald service

```bash
systemctl status systemd-journald
```

Check whether it is running:

```bash
systemctl is-active systemd-journald
```

---

# 17. Important real-time troubleshooting examples

### Scenario 1: SSH is failing

```bash
systemctl status sshd
journalctl -u sshd -n 100
journalctl -u sshd -f
```

Then:

```bash
journalctl -u sshd --since "30 minutes ago" | grep -i failed
```

---

### Scenario 2: Server suddenly rebooted

First:

```bash
journalctl --list-boots
```
[root@rhel ~]# journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY                 
  0 b7b892e47dda4ebaa4cfd4b2d7e4c330 Sun 2026-09-13 04:55:45 PDT Sun 2026-09-13 06:50:09 PDT
[root@rhel ~]# 

FIRST ENTRY - last reboot time
LAST ENTRY - latest log time
Then inspect previous boot:

```bash
journalctl -b -1
```

Look for errors:

```bash
journalctl -b -1 -p err
```

Kernel errors:

```bash
journalctl -k -b -1 -p err
```

---

### Scenario 3: Disk/filesystem problem

```bash
journalctl -k | grep -Ei "disk|error|xfs|ext4|nvme|I/O"
```

Or:

```bash
journalctl -k -p err
```

---

### Scenario 4: Network problem

```bash
journalctl -k | grep -Ei "network|link|eth|ens|enp"
```

For NetworkManager:

```bash
journalctl -u NetworkManager
```

Live:

```bash
journalctl -u NetworkManager -f
```

---

### Scenario 5: Application suddenly stopped

```bash
systemctl status myapp
```

Then:

```bash
journalctl -u myapp --since "1 hour ago"
```

Errors:

```bash
journalctl -u myapp -p err
```

---

