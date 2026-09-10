# Linux Performance Troubleshooting — L3 Interview Guide

For a **Linux L3 / System Administrator**, performance troubleshooting is one of the most important skills.

The key is not to randomly execute commands. Follow a structured troubleshooting methodology:

```text
                    Application is Slow
                           |
                           v
                 +---------------------+
                 | 1. Check CPU        |
                 +---------------------+
                           |
                           v
                 +---------------------+
                 | 2. Check Memory     |
                 +---------------------+
                           |
                           v
                 +---------------------+
                 | 3. Check Disk/I/O   |
                 +---------------------+
                           |
                           v
                 +---------------------+
                 | 4. Check Network    |
                 +---------------------+
                           |
                           v
                 +---------------------+
                 | 5. Check Processes  |
                 +---------------------+
                           |
                           v
                 +---------------------+
                 | 6. Check Logs       |
                 +---------------------+
                           |
                           v
                    Identify Root Cause
                           |
                           v
                         Fix
                           |
                           v
                       Verify
```

---

# 1. First Response to a Slow Server

When someone reports:

> "The Linux server is slow."

Do **not** immediately restart services or reboot the server.

Start with:

```bash
uptime
top
free -h
df -h
vmstat 1 5
```

These commands give a quick overview of:

* System load
* CPU utilization
* Memory utilization
* Filesystem capacity
* Processes
* Swap activity
* I/O activity

Then identify which resource is actually causing the problem.

---

# 2. CPU Troubleshooting

## 2.1 Check CPU Utilization

```bash
top
```

Important CPU fields:

| Field | Meaning           |
| ----- | ----------------- |
| `%us` | User-space CPU    |
| `%sy` | System/kernel CPU |
| `%id` | Idle CPU          |
| `%wa` | I/O wait          |
| `%st` | Steal time        |

Example:

```text
%Cpu(s): 85.0 us, 10.0 sy, 1.0 id, 4.0 wa
```

Interpretation:

```text
User CPU       = 85%
System CPU     = 10%
Idle           = 1%
I/O Wait       = 4%
```

The CPU is heavily utilized.

---

## 2.2 Find CPU-Consuming Processes

```bash
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head
```

Example:

```text
PID    PPID   CMD             %CPU
1234   1      java            85.2
5678   1      httpd           20.1
```

Using `top`:

```bash
top
```

Press:

```text
P
```

to sort processes by CPU utilization.

---

# 3. Load Average

Check:

```bash
uptime
```

Example:

```text
load average: 12.5, 10.2, 8.4
```

The three values represent:

```text
1 minute
5 minutes
15 minutes
```

Check CPU count:

```bash
nproc
```

Example:

```text
CPU cores = 4
Load      = 12
```

There may be significant contention.

However:

> **High load average does not automatically mean high CPU utilization.**

Load can also increase because processes are waiting on certain uninterruptible operations, commonly storage I/O.

---

# 4. Memory Troubleshooting

Start with:

```bash
free -h
```

Example:

```text
              total   used   free  shared  buff/cache  available
Mem:            16G    14G    500M    1G       1.5G        2G
Swap:            4G     3G      1G
```

Pay particular attention to:

```text
available
```

rather than only:

```text
free
```

Linux uses available RAM for filesystem caches, so low `free` memory by itself does not necessarily mean a memory problem.

---

# 5. Find Memory-Consuming Processes

```bash
ps -eo pid,ppid,cmd,%mem --sort=-%mem | head
```

Example:

```text
PID    PPID   CMD       %MEM
1234   1      java      65.2
2345   1      postgres  10.5
```

---

# 6. Check Swap Usage

```bash
free -h
```

Also:

```bash
swapon --show
```

Check swap activity:

```bash
vmstat 1 5
```

Important fields:

```text
si = swap in
so = swap out
```

Example:

```text
si   so
100  500
```

If `si` and `so` are continuously increasing, investigate memory pressure.

---

# 7. Disk Space Troubleshooting

First check filesystem usage:

```bash
df -h
```

Example:

```text
Filesystem        Size  Used  Avail  Use%
/dev/nvme0n1p2    100G   98G    2G   98%
```

The filesystem is almost full.

---

# 8. Find What Is Consuming Disk Space

Check directories:

```bash
du -xhd1 / | sort -h
```

For `/var`:

```bash
du -sh /var/*
```

For a particular directory:

```bash
du -sh /var/log/*
```

Common causes:

* Application logs
* System logs
* Core dumps
* Temporary files
* Backup files
* Large database files

---

# 9. `df` vs `du` Difference

This is a common L3 interview question.

### `df`

Shows filesystem-level disk usage:

```bash
df -h
```

### `du`

Shows space consumed by files/directories:

```bash
du -sh /var/*
```

---

# 10. Important Scenario — `df` Full but `du` Doesn't Match

Suppose:

```bash
df -h
```

shows:

```text
/dev/nvme0n1p2   100G   100G   0G   100%
```

But:

```bash
du -sh /
```

does not account for 100 GB.

Check deleted-but-open files:

```bash
lsof +L1
```

Example:

```text
java   1234   app   10w   REG   100G   /var/log/app.log (deleted)
```

The file was deleted, but the Java process still has the file descriptor open.

The filesystem space will not be released until the process closes the file descriptor.

Possible resolution:

* Restart the application during an approved maintenance window
* Or close/release the relevant file descriptor if appropriate

---

# 11. Disk I/O Troubleshooting

A server can be slow even when CPU utilization is low.

Install `sysstat` if required:

```bash
dnf install sysstat
```

Run:

```bash
iostat -xz 1
```

Important columns:

| Column  | Meaning                     |
| ------- | --------------------------- |
| `r/s`   | Reads per second            |
| `w/s`   | Writes per second           |
| `rkB/s` | Read throughput             |
| `wkB/s` | Write throughput            |
| `await` | Average I/O request latency |
| `%util` | Device utilization          |

---

# 12. Understanding `await`

`await` represents the average time an I/O request spends waiting for completion.

High `await` can indicate storage latency.

Investigate:

```text
High await
     |
     +-- Storage latency?
     +-- Heavy workload?
     +-- Disk contention?
     +-- SAN/NAS issue?
     +-- Cloud volume limitation?
     +-- Application generating excessive I/O?
```

---

# 13. Understanding `%util`

Example:

```text
Device      %util
nvme0n1     99%
```

The device is heavily utilized.

However, do not interpret `%util` in isolation.

SSD/NVMe devices can process many operations concurrently, so investigate:

```text
IOPS
throughput
latency
queue depth
application workload
storage type
```

---

# 14. Find Processes Performing I/O

Using `iotop`:

```bash
iotop
```

Or:

```bash
pidstat -d 1
```

Example:

```text
PID    kB_rd/s   kB_wr/s
1234   50000     1000
5678   1000      80000
```

This helps identify processes generating heavy disk I/O.

---

# 15. `vmstat` — Very Important

Run:

```bash
vmstat 1
```

Example:

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs   us sy id wa
 8  2      0   500M   ...    ...    0    0  5000  8000  ...  ...   60 10  5 25
```

Important fields:

| Field | Meaning                            |
| ----- | ---------------------------------- |
| `r`   | Runnable processes waiting for CPU |
| `b`   | Processes blocked                  |
| `si`  | Swap in                            |
| `so`  | Swap out                           |
| `bi`  | Blocks read                        |
| `bo`  | Blocks written                     |
| `in`  | Interrupts                         |
| `cs`  | Context switches                   |
| `us`  | User CPU                           |
| `sy`  | System CPU                         |
| `id`  | Idle CPU                           |
| `wa`  | I/O wait                           |

---

# 16. `vmstat` Interview Scenario

Suppose:

```text
us = 20%
sy = 5%
id = 5%
wa = 70%
```

This suggests the system is spending significant time waiting for I/O.

Check:

```bash
iostat -xz 1
```

Then identify which device is busy.

Next:

```bash
pidstat -d 1
```

Identify the process generating the I/O.

---

# 17. Network Performance Troubleshooting

Check interfaces:

```bash
ip -s link
```

Look for:

```text
RX errors
TX errors
RX dropped
TX dropped
```

High errors/drops may indicate:

* NIC problems
* Driver problems
* Network congestion
* Duplex/configuration issues
* Physical/network infrastructure problems

---

# 18. Check Network Connections

```bash
ss -s
```

Shows socket statistics.

Check listening ports:

```bash
ss -lntp
```

Example:

```text
LISTEN  0  128  0.0.0.0:8080  0.0.0.0:*  users:(("java",pid=1234))
```

Check a specific port:

```bash
ss -lntp | grep 8080
```

---

# 19. Network Connectivity

Basic connectivity:

```bash
ping <server>
```

Check routing:

```bash
ip route
```

Trace network path:

```bash
tracepath <server>
```

or:

```bash
traceroute <server>
```

---

# 20. DNS Troubleshooting

Check DNS resolution:

```bash
dig example.com
```

Check resolver configuration:

```bash
cat /etc/resolv.conf
```

Check DNS service status if applicable:

```bash
resolvectl status
```

Typical symptoms:

```text
Application is slow
       |
       v
DNS lookup takes several seconds
       |
       v
Investigate DNS configuration/connectivity
```

---

# 21. Process Troubleshooting

List processes:

```bash
ps -ef
```

CPU:

```bash
ps -eo pid,cmd,%cpu --sort=-%cpu | head
```

Memory:

```bash
ps -eo pid,cmd,%mem --sort=-%mem | head
```

Process details:

```bash
ps -fp <PID>
```

---

# 22. Open Files

Check files opened by a process:

```bash
lsof -p <PID>
```

Check network files/sockets:

```bash
lsof -i -P -n
```

Count open files:

```bash
lsof -p <PID> | wc -l
```

---

# 23. Thread Troubleshooting

List threads:

```bash
ps -eLf
```

For a particular process:

```bash
ps -L -p <PID>
```

For CPU-heavy Java applications:

```bash
top -H -p <PID>
```

This allows you to identify CPU-consuming threads.

---

# 24. Systemd Performance

Check service status:

```bash
systemctl status httpd
```

Check overall boot time:

```bash
systemd-analyze
```

Find services that took the longest to start:

```bash
systemd-analyze blame
```

Check service logs:

```bash
journalctl -u httpd
```

Follow logs:

```bash
journalctl -u httpd -f
```

---

# 25. System Logs During Performance Problems

Check errors from the current boot:

```bash
journalctl -p err -b
```

Kernel messages:

```bash
dmesg -T
```

Search for errors:

```bash
dmesg -T | grep -i error
```

Look for:

```text
OOM
I/O errors
Disk errors
Filesystem errors
NIC errors
Kernel errors
Hardware errors
```

---

# 26. OOM — Out Of Memory

OOM means:

> Out Of Memory

Check:

```bash
journalctl -k | grep -i oom
```

or:

```bash
dmesg -T | grep -i oom
```

Example:

```text
Out of memory: Killed process 1234 (java)
```

Investigate:

```bash
free -h
```

```bash
ps aux --sort=-%mem | head
```

Potential causes:

* Application memory leak
* Too many processes
* JVM heap too large
* Insufficient RAM
* Container memory limit
* Incorrect application configuration

Remember:

> High filesystem cache usage alone does not necessarily mean memory exhaustion.

---

# 27. File Descriptor Problems

Check shell limit:

```bash
ulimit -n
```

For a process:

```bash
cat /proc/<PID>/limits
```

Look for:

```text
Max open files
```

Check number of open files:

```bash
lsof -p <PID> | wc -l
```

System-wide information:

```bash
cat /proc/sys/fs/file-nr
```

Typical application error:

```text
Too many open files
```

This can prevent applications from creating:

* Files
* Sockets
* Network connections

---

# 28. Zombie Processes

Find zombie processes:

```bash
ps aux | awk '$8 ~ /Z/'
```

Or:

```bash
ps -eo pid,ppid,state,cmd | grep ' Z '
```

A zombie process has already exited, but its parent has not collected its exit status.

You normally do **not** kill the zombie directly.

Find the parent:

```bash
ps -o pid,ppid,state,cmd -p <PID>
```

Then investigate the parent process.

---

# 29. High CPU — Real-Time Scenario

### Problem

A Java application is slow.

Run:

```bash
top
```

You see:

```text
java   80% CPU
```

Do not immediately restart Java.

Find the PID:

```bash
pgrep -f java
```

Check threads:

```bash
top -H -p <PID>
```

or:

```bash
ps -L -p <PID> -o pid,tid,%cpu,cmd --sort=-%cpu
```

Identify the problematic thread.

Then work with the application/JVM team to investigate:

* Thread dumps
* Garbage collection
* Infinite loops
* Application workload
* Database calls
* External dependencies

---

# 30. High I/O — Real-Time Scenario

### Symptoms

```text
Application is slow

CPU = 20%
Memory = Normal
Disk utilization = 100%
```

Run:

```bash
iostat -xz 1
```

Then:

```bash
iotop
```

or:

```bash
pidstat -d 1
```

Determine:

```text
Which disk?
     |
     v
Which process?
     |
     v
Read or write?
     |
     v
What is causing the workload?
```

Possible causes:

* Database activity
* Backup jobs
* Excessive application logging
* Large file operations
* Storage latency
* Heavy write workload

---

# 31. High Load Average — Interview Scenario

### Interviewer:

> The server load average is 15. What will you do?

### Strong L3 answer:

```text
I wouldn't conclude that CPU is the problem just from load average.

First I would check:

uptime
nproc
top
vmstat 1
iostat -xz 1

Then I would determine whether the load is caused by:

1. CPU contention
2. Disk I/O
3. Blocked processes
4. Memory pressure

After identifying the bottleneck, I would identify the responsible
process, check logs and configuration, apply the appropriate fix,
and then verify that the performance has returned to normal.
```

---

# 32. Complete Troubleshooting Flow

```text
                  SERVER IS SLOW
                        |
                        v
                     uptime
                        |
                        v
                       top
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
        CPU           Memory          I/O
          |             |             |
          v             v             v
         ps          free -h       iostat
          |             |             |
          v             v             v
       Threads       Swap?        pidstat
          |             |             |
          +-------------+-------------+
                        |
                        v
                    Network
                        |
                        v
                   ip -s link
                        |
                        v
                       ss
                        |
                        v
                     Routing
                        |
                        v
                       DNS
                        |
                        v
                      Logs
                        |
                        v
                   journalctl
                        |
                        v
                     dmesg
                        |
                        v
                  ROOT CAUSE
                        |
                        v
                       FIX
                        |
                        v
                     VERIFY
```

---

# 33. Performance Command Cheat Sheet

| Requirement         | Command                 |
| ------------------- | ----------------------- |
| Load average        | `uptime`                |
| CPU                 | `top`                   |
| CPU process         | `ps ... --sort=-%cpu`   |
| CPU count           | `nproc`                 |
| Memory              | `free -h`               |
| Swap                | `swapon --show`         |
| Memory/process      | `ps ... --sort=-%mem`   |
| Overall performance | `vmstat 1`              |
| Disk space          | `df -h`                 |
| Directory usage     | `du -sh`                |
| Disk I/O            | `iostat -xz 1`          |
| I/O process         | `iotop`                 |
| Process I/O         | `pidstat -d 1`          |
| Network interface   | `ip -s link`            |
| Network sockets     | `ss -s`                 |
| Listening ports     | `ss -lntp`              |
| Routing             | `ip route`              |
| DNS                 | `dig`                   |
| Process details     | `ps -fp <PID>`          |
| Open files          | `lsof -p <PID>`         |
| Threads             | `ps -L -p <PID>`        |
| Service             | `systemctl status`      |
| Service logs        | `journalctl -u`         |
| Boot analysis       | `systemd-analyze`       |
| Slow services       | `systemd-analyze blame` |
| Kernel logs         | `dmesg -T`              |
| System errors       | `journalctl -p err -b`  |

---

# 34. Commands to Memorize for L3

```bash
uptime
top
nproc

free -h
swapon --show
vmstat 1

df -h
du -sh
lsof +L1

iostat -xz 1
iotop
pidstat -d 1

ip -s link
ip route
ss -s
ss -lntp
ping
tracepath
dig

ps -ef
ps -fp <PID>
ps -L -p <PID>
lsof -p <PID>

systemctl status <service>
systemd-analyze
systemd-analyze blame
journalctl -u <service>
journalctl -p err -b
dmesg -T
```

---

# 35. Golden L3 Troubleshooting Methodology

Always remember:

```text
             MEASURE
                |
                v
       IDENTIFY BOTTLENECK
                |
                v
        IDENTIFY PROCESS
                |
                v
       CHECK LOGS / CONFIG
                |
                v
          FIND ROOT CAUSE
                |
                v
              FIX
                |
                v
            VERIFY
                |
                v
          DOCUMENT
```

## Interview Golden Rule

Never say:

> "The server is slow, so I'll restart it."

Instead say:

> **"I will first measure the system and identify whether the bottleneck is CPU, memory, storage I/O, network, or a specific process. Once I identify the root cause, I will apply the appropriate corrective action and verify the result."**

This demonstrates **L3-level troubleshooting rather than command memorization**.
