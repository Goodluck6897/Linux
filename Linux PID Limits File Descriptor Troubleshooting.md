
# Linux PID Limits & File Descriptor Troubleshooting – L3 Interview Notes

---

## 27. File Descriptor Problems

### What is a File Descriptor (FD)?

A **File Descriptor** is a unique integer that the kernel assigns to every open file, socket, pipe, or network connection. Every process has a **limit** on how many FDs it can open. When the limit is reached, the application **cannot create** new:

- Files
- Sockets
- Network connections
- Pipes
- Database connections

### Typical Application Error

```
java.io.IOException: Too many open files
[error] accept4(): Too many open files
socket: Too many open files
```

---

### Where to Check File Descriptor Limits

#### 1. Shell-Level Limit (Current User)

```bash
ulimit -n
```
**Example output:**
```
1024        ← Current user can open max 1024 files per process
```

#### 2. Soft and Hard Limits

```bash
ulimit -Sn    # Soft limit (current effective limit)
ulimit -Hn    # Hard limit (maximum ceiling)
```
**Example output:**
```
Soft: 1024
Hard: 65536
```

| Type | Meaning |
|---|---|
| **Soft limit** | Default limit applied to the process (can be increased by user up to hard limit) |
| **Hard limit** | Maximum ceiling (only root can increase) |

#### 3. Per-Process Limit (Running Process)

```bash
cat /proc/<PID>/limits
```
**Example:**
```bash
cat /proc/1234/limits
```
**Example output:**
```
Limit                     Soft Limit   Hard Limit   Units
Max open files            1024         65536        files      ← LOOK FOR THIS
Max processes             4096         63704        processes
Max locked memory         65536        65536        bytes
```

> **Key line:** `Max open files` — this tells you the FD limit for that specific process.

#### 4. Check How Many Files a Process Has Open

```bash
lsof -p <PID> | wc -l
```
**Example:**
```bash
lsof -p 1234 | wc -l
```
**Example output:**
```
987     ← Process 1234 has 987 open file descriptors
```

> ⚠️ If this number is **close to the soft limit (1024)**, the process is about to hit "Too many open files"!

#### 5. See What Files Are Open

```bash
lsof -p <PID> | head -20
```
**Example output:**
```
COMMAND  PID   USER   FD   TYPE   DEVICE  SIZE/OFF  NODE  NAME
java     1234  app    0u   CHR    1,3     0t0       6     /dev/null
java     1234  app    1u   REG    8,1     45678     123   /var/log/app.log
java     1234  app    6u   IPv4   98765   0t0       TCP   10.0.0.5:8080->10.0.0.6:443 (ESTABLISHED)
java     1234  app    7u   IPv4   98766   0t0       TCP   10.0.0.5:8080->10.0.0.7:3306 (ESTABLISHED)
```

| FD Type | Meaning |
|---|---|
| `REG` | Regular file |
| `IPv4/IPv6` | Network socket |
| `CHR` | Character device |
| `FIFO` | Pipe |
| `unix` | Unix domain socket |

#### 6. Alternative: Check via /proc

```bash
# Count open FDs for a process
ls /proc/<PID>/fd | wc -l

# List all open FDs
ls -la /proc/<PID>/fd
```
**Example:**
```bash
ls /proc/1234/fd | wc -l
```
**Output:**
```
987
```

#### 7. System-Wide File Descriptor Information

```bash
cat /proc/sys/fs/file-nr
```
**Example output:**
```
3456    0    1048576
│       │    │
│       │    └── Maximum FDs the system can allocate (system-wide limit)
│       └────── Number of allocated but unused FDs
└────────────── Number of currently allocated FDs
```

#### 8. System-Wide Maximum Limit

```bash
cat /proc/sys/fs/file-max
```
**Example output:**
```
1048576     ← System can handle ~1 million open files total
```

---

### How to Increase File Descriptor Limits

#### Temporary (Current Session Only)

```bash
# Increase soft limit (up to hard limit)
ulimit -n 65536
```

#### Permanent: Per-User in `/etc/security/limits.conf`

```bash
sudo vi /etc/security/limits.conf
```
```
# <domain>   <type>   <item>       <value>
*             soft     nofile       65536
*             hard     nofile       65536
appuser       soft     nofile       131072
appuser       hard     nofile       131072
```

> ⚠️ User must **log out and log back in** for changes to take effect.

#### Permanent: Per-Service in Systemd

```bash
sudo systemctl edit httpd
```
```
[Service]
LimitNOFILE=65536
```
```bash
sudo systemctl daemon-reload
sudo systemctl restart httpd

# Verify
systemctl show httpd | grep LimitNOFILE
```

#### Permanent: System-Wide Kernel Limit

```bash
# Temporary
sudo sysctl -w fs.file-max=2097152

# Permanent
echo "fs.file-max = 2097152" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### Troubleshooting Workflow: "Too many open files"

```bash
# Step 1: Identify the affected process
systemctl status myapp
# or
ps aux | grep myapp
# Note the PID (e.g., 1234)

# Step 2: Check the process limit
cat /proc/1234/limits | grep "open files"
# Output: Max open files  1024  65536  files

# Step 3: Check how many files are open
lsof -p 1234 | wc -l
# Output: 1020  ← Almost at the 1024 limit!

# Step 4: Identify what's consuming FDs
lsof -p 1234 | awk '{print $5}' | sort | uniq -c | sort -rn
# Output:
#   650  IPv4       ← Too many network connections!
#   200  REG        ← Regular files
#   150  unix       ← Unix sockets

# Step 5: Check for FD leaks (connections not being closed)
lsof -p 1234 | grep CLOSE_WAIT | wc -l
# Output: 400  ← 400 connections stuck in CLOSE_WAIT = FD LEAK!

# Step 6: Fix — Increase limit + fix the application
sudo systemctl edit myapp
# Add: LimitNOFILE=65536
sudo systemctl daemon-reload
sudo systemctl restart myapp

# Step 7: Verify
cat /proc/<NEW_PID>/limits | grep "open files"
# Output: Max open files  65536  65536  files
```

---

## 48. PID Limits

### What is a PID Limit?

Every process in Linux gets a unique **Process ID (PID)**. The system has a **maximum number of PIDs** it can assign. If the limit is reached, **no new processes can be created** — applications, logins, and commands will fail.

### Typical Error

```
-bash: fork: retry: Resource temporarily unavailable
-bash: fork: Resource temporarily unavailable
Cannot allocate memory
```

---

### Where to Check PID Limits

#### 1. System-Wide Maximum PID Limit (Kernel Level)

```bash
cat /proc/sys/kernel/pid_max
```
**Example output:**
```
32768       ← Default on 32-bit systems
4194304     ← Default on 64-bit systems (can go up to 4,194,304)
```

#### 2. Check Current Number of Running Processes

```bash
ps aux | wc -l

# Or check from /proc
ls -d /proc/[0-9]* | wc -l
```

#### 3. Check Threads Limit (threads-max)

```bash
cat /proc/sys/kernel/threads-max
```
**Example output:**
```
63704       ← Maximum number of threads system-wide
```

> Each thread also consumes a PID, so this is equally important.

---

### Per-User PID Limits

#### 4. Check Using `ulimit`

```bash
ulimit -u
```
**Example output:**
```
4096        ← This user can create max 4096 processes
```

#### 5. Check in `/etc/security/limits.conf`

```bash
cat /etc/security/limits.conf
```
**Example entries:**
```
# <domain>   <type>   <item>     <value>
*             soft     nproc      4096
*             hard     nproc      63704
appuser       soft     nproc      8192
appuser       hard     nproc      16384
```

| Field | Meaning |
|---|---|
| `*` | Applies to **all users** |
| `appuser` | Applies to specific user |
| `soft` | Default limit (user can increase up to hard) |
| `hard` | Maximum limit (only root can increase) |
| `nproc` | Number of processes |

#### 6. Check in `/etc/security/limits.d/` (Override Directory)

```bash
cat /etc/security/limits.d/20-nproc.conf
```
**Example output (RHEL default):**
```
*          soft    nproc     4096
root       soft    nproc     unlimited
```

> ⚠️ Files in `limits.d/` **override** `limits.conf`. Always check both!

---

### Systemd-Based PID Limits (cgroups)

#### 7. System-Wide Systemd Limit

```bash
cat /etc/systemd/system.conf | grep -i taskmax
```
**Example output:**
```
DefaultTasksMax=4915      ← Default max tasks per service unit
```

#### 8. Per-Service PID Limit

```bash
systemctl show <service_name> | grep TasksMax
```
**Example:**
```bash
systemctl show httpd | grep TasksMax
```
**Output:**
```
TasksMax=4915
```

#### 9. Check Current Tasks Used by a Service

```bash
systemctl status httpd
```
**Example output:**
```
● httpd.service - The Apache HTTP Server
   Tasks: 213 (limit: 4915)     ← 213 out of 4915 used
```

#### 10. Check via cgroup Directly

```bash
# Max allowed
cat /sys/fs/cgroup/pids/system.slice/httpd.service/pids.max

# Currently used
cat /sys/fs/cgroup/pids/system.slice/httpd.service/pids.current
```

---

### How to Modify PID Limits

#### Increase Kernel PID Max (Temporary)
```bash
sudo sysctl -w kernel.pid_max=4194304
```

#### Increase Kernel PID Max (Permanent)
```bash
echo "kernel.pid_max = 4194304" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### Increase Per-User Process Limit
```bash
sudo vi /etc/security/limits.conf
```
```
appuser    soft    nproc    8192
appuser    hard    nproc    16384
```
> ⚠️ User must **log out and log back in** for changes to take effect.

#### Increase Per-Service TasksMax
```bash
sudo systemctl edit httpd
```
```
[Service]
TasksMax=8192
```
```bash
sudo systemctl daemon-reload
sudo systemctl restart httpd
```

---

### Troubleshooting: "Cannot fork" or "Resource temporarily unavailable"

```bash
# Step 1: Check who's consuming the most processes
ps -eo user --sort user | uniq -c | sort -rn | head
```
**Example output:**
```
  3500  appuser      ← This user has 3500 processes!
   250  root
    45  nobody
```

```bash
# Step 2: Check what those processes are
ps -u appuser -o pid,ppid,cmd --sort=-pid | head -20

# Step 3: Check the user's limit
su - appuser -c "ulimit -u"

# Step 4: Check system-wide
cat /proc/sys/kernel/pid_max
ps aux | wc -l
```

---

## Quick Comparison: File Descriptors vs PID Limits

| Aspect | File Descriptors (FD) | PID Limits |
|---|---|---|
| **What it limits** | Open files, sockets, connections per process | Number of processes/threads system-wide or per user |
| **Typical error** | `Too many open files` | `fork: Resource temporarily unavailable` |
| **Check current usage** | `lsof -p <PID> \| wc -l` | `ps aux \| wc -l` |
| **Check limit** | `ulimit -n` | `ulimit -u` |
| **Per-process limit** | `cat /proc/<PID>/limits` → `Max open files` | `cat /proc/<PID>/limits` → `Max processes` |
| **System-wide limit** | `cat /proc/sys/fs/file-max` | `cat /proc/sys/kernel/pid_max` |
| **System-wide usage** | `cat /proc/sys/fs/file-nr` | `ls -d /proc/[0-9]* \| wc -l` |
| **limits.conf item** | `nofile` | `nproc` |
| **Systemd setting** | `LimitNOFILE=` | `TasksMax=` |
| **Config file** | `/etc/security/limits.conf` | `/etc/security/limits.conf` |
| **Kernel parameter** | `fs.file-max` | `kernel.pid_max` |

---

## Interview Tips

- **Always check at 3 levels:** Kernel → User → Process/Service
- **Compare usage vs limit** — knowing the limit alone isn't enough; check how close you are
- **Identify the root cause** — is it a legitimate load increase or a leak (FD leak / fork bomb)?
- **Know the difference** — `nofile` (FD) vs `nproc` (PID) in limits.conf
- **Mention systemd** — modern systems use `LimitNOFILE` and `TasksMax` in service units
- **Communicate** — explain that you'd coordinate with the application team to fix leaks, not just increase limits
