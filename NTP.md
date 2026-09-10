# NTP / Chrony — RHEL 8 Linux Administration

## 1. What is NTP?

**NTP (Network Time Protocol)** is used to synchronize the system clock with a reliable time source.

Accurate time is important for:

* Application logs
* Database transactions
* Authentication
* Kerberos
* SSL/TLS
* Cluster communication
* Monitoring
* Troubleshooting
* Distributed systems

```text
                  NTP Server
                 10.10.10.10
                      |
                      | UDP/123
                      |
                      v
                +-----------+
                | RHEL 8    |
                | Server     |
                +-----------+
                System Clock
```

---

# 2. NTP vs Chrony

In **RHEL 8**, `chrony` is the default and preferred time synchronization solution.

| Component | Description                      |
| --------- | -------------------------------- |
| NTP       | Time synchronization protocol    |
| Chrony    | NTP implementation               |
| `chronyd` | Chrony daemon                    |
| `chronyc` | Command-line administration tool |
| `ntpd`    | Legacy NTP daemon                |
| UDP 123   | NTP network port                 |

Think of it as:

```text
NTP       = Protocol
Chrony    = Implementation
chronyd   = Daemon
chronyc   = Management command
```

---

# 3. Chrony Components

## chronyd

`chronyd` is the background service responsible for synchronizing system time.

Check status:

```bash
systemctl status chronyd
```

Start:

```bash
systemctl start chronyd
```

Stop:

```bash
systemctl stop chronyd
```

Restart:

```bash
systemctl restart chronyd
```

Enable at boot:

```bash
systemctl enable chronyd
```

Enable and start:

```bash
systemctl enable --now chronyd
```

---

# 4. Chrony Configuration File

Main configuration file:

```text
/etc/chrony.conf
```

View:

```bash
cat /etc/chrony.conf
```

Example:

```text
pool 2.rhel.pool.ntp.org iburst
pool 3.rhel.pool.ntp.org iburst
pool 4.rhel.pool.ntp.org iburst
```

Corporate environment example:

```text
server ntp01.example.com iburst
server ntp02.example.com iburst
```

Using IP addresses:

```text
server 10.10.10.10 iburst
server 10.10.10.11 iburst
```

---

# 5. What is `iburst`?

Example:

```text
server 10.10.10.10 iburst
```

`iburst` allows Chrony to make several quick measurements when synchronization starts.

This helps the system synchronize faster initially.

### Interview answer

> `iburst` allows Chrony to make a burst of measurements when synchronization starts, helping the system synchronize quickly.

---

# 6. Check Current Time

Use:

```bash
date
```

Example:

```text
Thu Sep 10 14:52:10 CDT 2026
```

More detailed information:

```bash
timedatectl
```

Example:

```text
               Local time: Thu 2026-09-10 14:52:10 CDT
           Universal time: Thu 2026-09-10 20:52:10 UTC
                 RTC time: Thu 2026-09-10 20:52:10
                Time zone: America/Mexico_City
System clock synchronized: yes
              NTP service: active
```

---

# 7. Important `timedatectl` Commands

Display time configuration:

```bash
timedatectl
```

Display time zones:

```bash
timedatectl list-timezones
```

Set time zone:

```bash
timedatectl set-timezone America/Mexico_City
```

Enable NTP synchronization:

```bash
timedatectl set-ntp true
```

Disable NTP synchronization:

```bash
timedatectl set-ntp false
```

Check status:

```bash
timedatectl status
```

---

# 8. Check Chrony Synchronization

## chronyc tracking

One of the most important commands:

```bash
chronyc tracking
```

Example:

```text
Reference ID    : 10.10.10.10
Stratum         : 3
Ref time (UTC)  : Thu Sep 10 20:50:10 2026
System time     : 0.000123 seconds fast
Last offset     : -0.000234 seconds
RMS offset      : 0.001234 seconds
Leap status     : Normal
```

Important fields:

| Field        | Meaning                       |
| ------------ | ----------------------------- |
| Reference ID | Current NTP source            |
| Stratum      | Distance from reference clock |
| Ref time     | Last reference time           |
| System time  | Current clock offset          |
| Last offset  | Last measured offset          |
| RMS offset   | Average offset                |
| Leap status  | Leap-second status            |

---

# 9. What is Stratum?

Stratum indicates how far the system is from the reference clock.

Conceptually:

```text
Stratum 0
   |
   | Atomic clock / GPS
   v
Stratum 1
   |
   v
Stratum 2
   |
   v
Stratum 3
   |
   v
Linux Server
```

Lower stratum generally means the clock is closer to the original reference.

### Interview answer

> Stratum represents the distance from the reference clock. Stratum 0 represents highly accurate reference clocks such as atomic clocks or GPS sources.

---

# 10. Check NTP Sources

Use:

```bash
chronyc sources -v
```

Example:

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 10.10.10.10                   2   6   377    40   +123us
^+ 10.10.10.11                   2   6   377    35   -210us
```

Important symbols:

```text
*  = Currently selected source
+  = Good candidate
-  = Acceptable but not selected
?  = Source currently unreachable
x  = Source considered unreliable/bad
```

### Most important

```text
^* 10.10.10.10
```

means:

> Chrony has selected `10.10.10.10` as the current synchronization source.

---

# 11. Check NTP Sources Without Verbose Output

```bash
chronyc sources
```

Example:

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* ntp01.example.com             2   6   377    32   +100us
^+ ntp02.example.com             2   6   377    30   -200us
```

---

# 12. Force Immediate Time Correction

Use:

```bash
chronyc makestep
```

This tells Chrony to immediately correct a significant clock difference instead of gradually adjusting it.

Example:

```text
Server clock is 10 minutes wrong
            |
            v
     chronyc makestep
            |
            v
      Clock corrected
```

### Important

Suddenly changing system time can affect:

* Applications
* Databases
* Scheduled jobs
* Distributed systems
* Logs

Use it carefully in production.

---

# 13. NTP Network Port

NTP uses:

```text
UDP/123
```

Communication:

```text
Linux Server
     |
     | UDP 123
     v
NTP Server
```

If synchronization fails, verify that UDP/123 is allowed.

---

# 14. Check Firewall

Check firewalld:

```bash
firewall-cmd --state
```

List rules:

```bash
firewall-cmd --list-all
```

List services:

```bash
firewall-cmd --list-services
```

For an NTP server, UDP/123 must be allowed from the appropriate clients.

---

# 15. Test Network Connectivity

Basic connectivity:

```bash
ping 10.10.10.10
```

Important:

> Successful ping does NOT prove that NTP is working.

Why?

Because:

```text
ping       -> ICMP
NTP        -> UDP/123
```

You must verify UDP/123 separately.

---

# 16. Troubleshoot NTP Traffic with tcpdump

Use:

```bash
tcpdump -i eth0 udp port 123
```

More readable:

```bash
tcpdump -i eth0 -nn udp port 123
```

You should see NTP packets between the Linux server and NTP server.

Example troubleshooting:

```text
Linux Server
     |
     | UDP/123
     |
     X
 Firewall / Network ACL
     |
     X
NTP Server
```

---

# 17. Check Chrony Logs

Use:

```bash
journalctl -u chronyd
```

Follow logs live:

```bash
journalctl -u chronyd -f
```

Show recent logs:

```bash
journalctl -u chronyd --since "1 hour ago"
```

Show errors:

```bash
journalctl -u chronyd -p err
```

---

# 18. Check Chrony Configuration

```bash
cat /etc/chrony.conf
```

Look for:

```text
server
pool
```

Example:

```text
server 10.10.10.10 iburst
server 10.10.10.11 iburst
```

After changing configuration:

```bash
systemctl restart chronyd
```

Then verify:

```bash
chronyc sources -v
```

and:

```bash
chronyc tracking
```

---

# 19. Complete NTP Troubleshooting Flow

When a server has incorrect time:

```text
              Time is incorrect
                     |
                     v
             timedatectl / date
                     |
                     v
             Is chronyd running?
                  /       \
                No         Yes
                |           |
                v           v
          Start chronyd   chronyc tracking
                            |
                            v
                    chronyc sources -v
                            |
                            v
                     Is source ^* ?
                       /          \
                     No            Yes
                     |              |
                     v              v
              Check ^? source    Check offset
                     |
                     v
              Check /etc/chrony.conf
                     |
                     v
             Check network connectivity
                     |
                     v
                  UDP/123
                     |
                     v
               Firewall / ACL
                     |
                     v
             journalctl -u chronyd
                     |
                     v
              chronyc makestep
                     |
                     v
                  Verify
```

---

# 20. Real-Time L3 Troubleshooting Scenario

### Problem

> Application server time is incorrect.

### Step 1 — Check system time

```bash
date
timedatectl
```

### Step 2 — Check Chrony

```bash
systemctl status chronyd
```

If stopped:

```bash
systemctl start chronyd
```

### Step 3 — Check synchronization

```bash
chronyc tracking
```

### Step 4 — Check NTP sources

```bash
chronyc sources -v
```

If you see:

```text
^? 10.10.10.10
```

the source may be unreachable.

### Step 5 — Check configuration

```bash
cat /etc/chrony.conf
```

Verify:

```text
server 10.10.10.10 iburst
```

### Step 6 — Check connectivity

```bash
ping 10.10.10.10
```

Remember: ping only verifies ICMP.

### Step 7 — Check NTP traffic

```bash
tcpdump -i eth0 -nn udp port 123
```

### Step 8 — Check firewall

```bash
firewall-cmd --list-all
```

### Step 9 — Check logs

```bash
journalctl -u chronyd
```

### Step 10 — Correct clock if required

```bash
chronyc makestep
```

### Step 11 — Verify

```bash
chronyc tracking
chronyc sources -v
timedatectl
```

---

# 21. NTP Client

A RHEL server normally acts as an NTP client when synchronizing with an external/internal time source.

```text
              NTP Server
             10.10.10.10
                   |
                   | UDP/123
                   v
             +-----------+
             | RHEL 8    |
             | Client    |
             +-----------+
```

Configuration:

```bash
vi /etc/chrony.conf
```

Example:

```text
server 10.10.10.10 iburst
```

Restart:

```bash
systemctl restart chronyd
```

Verify:

```bash
chronyc sources -v
```

---

# 22. RHEL Server as an NTP Server

Chrony can also provide time to other systems.

Architecture:

```text
             Upstream NTP
                  |
                  v
          +---------------+
          | RHEL 8        |
          | Chrony Server |
          +---------------+
             /         \
            /           \
           v             v
       Linux-01       Linux-02
```

In `/etc/chrony.conf`, configure the clients that are allowed to synchronize.

Example:

```text
allow 10.10.20.0/24
```

Then restart:

```bash
systemctl restart chronyd
```

Verify:

```bash
chronyc clients
```

---

# 23. NTP vs PTP

## NTP

Network Time Protocol.

Used for general-purpose time synchronization.

Typical environments:

* Linux servers
* Application servers
* Databases
* Monitoring
* Enterprise infrastructure

Generally provides millisecond-level synchronization depending on network/environment.

---

## PTP

Precision Time Protocol.

Defined by:

```text
IEEE 1588
```

Used when much higher precision is required.

Common environments:

* Telecom
* 5G
* Industrial systems
* Financial systems
* High-precision systems

Conceptually:

```text
NTP
 |
 +-- General-purpose synchronization
 |
 +-- Linux servers
 |
 +-- Millisecond-level requirements


PTP
 |
 +-- High precision
 |
 +-- Microsecond/sub-microsecond requirements
 |
 +-- Specialized environments
```

### Interview answer

> For normal RHEL server administration, Chrony using NTP is the standard choice. PTP is used when applications require significantly higher time precision.

---

# 24. Important Commands Cheat Sheet

| Purpose                | Command                            |
| ---------------------- | ---------------------------------- |
| Current time           | `date`                             |
| Time configuration     | `timedatectl`                      |
| Chrony status          | `systemctl status chronyd`         |
| Start Chrony           | `systemctl start chronyd`          |
| Restart Chrony         | `systemctl restart chronyd`        |
| Enable Chrony          | `systemctl enable chronyd`         |
| Chrony configuration   | `cat /etc/chrony.conf`             |
| Synchronization status | `chronyc tracking`                 |
| NTP sources            | `chronyc sources -v`               |
| Force correction       | `chronyc makestep`                 |
| Chrony logs            | `journalctl -u chronyd`            |
| Live logs              | `journalctl -u chronyd -f`         |
| NTP traffic            | `tcpdump -i eth0 -nn udp port 123` |
| Firewall               | `firewall-cmd --list-all`          |

---

# 25. Top Interview Questions

### Q1. What is NTP?

> NTP is a protocol used to synchronize system clocks over a network.

### Q2. What does RHEL 8 use for NTP?

> RHEL 8 uses Chrony as the default and preferred time synchronization solution.

### Q3. What is `chronyd`?

> `chronyd` is the background daemon that performs time synchronization.

### Q4. What is `chronyc`?

> `chronyc` is the command-line client used to query and manage Chrony.

### Q5. Where is Chrony configured?

```text
/etc/chrony.conf
```

### Q6. What port does NTP use?

```text
UDP/123
```

### Q7. How do you check synchronization?

```bash
chronyc tracking
timedatectl
```

### Q8. How do you check NTP sources?

```bash
chronyc sources -v
```

### Q9. What does `*` mean?

> It indicates the NTP source currently selected by Chrony.

### Q10. What does `?` mean?

> The source is currently unreachable or there is insufficient information to use it.

### Q11. What is `iburst`?

> It allows Chrony to make several quick measurements when synchronization starts, helping faster initial synchronization.

### Q12. How do you force immediate correction?

```bash
chronyc makestep
```

### Q13. How do you troubleshoot NTP?

```text
timedatectl
    ↓
systemctl status chronyd
    ↓
chronyc tracking
    ↓
chronyc sources -v
    ↓
cat /etc/chrony.conf
    ↓
Network / UDP 123
    ↓
Firewall
    ↓
journalctl -u chronyd
    ↓
chronyc makestep
    ↓
Verify
```

---

# 26. L3 Interview — One-Minute Answer

If the interviewer asks:

> "How do you troubleshoot time synchronization on a RHEL 8 server?"

Answer:

> "First, I check the current time and synchronization status using `timedatectl`. Then I verify that `chronyd` is running using `systemctl status chronyd`. I use `chronyc tracking` to check synchronization and `chronyc sources -v` to identify the selected NTP source. If the source is unreachable, I check `/etc/chrony.conf`, network connectivity, firewall rules, and UDP port 123. I also check `journalctl -u chronyd` for errors. If the clock has a significant offset, I can use `chronyc makestep`, and finally I verify synchronization again using `chronyc tracking` and `timedatectl`."

---

# 27. Must-Memorize Commands

For a **RHEL 8 Linux L3 interview**, remember these first:

```bash
timedatectl

systemctl status chronyd

cat /etc/chrony.conf

chronyc tracking

chronyc sources -v

chronyc makestep

journalctl -u chronyd

tcpdump -i eth0 -nn udp port 123
```

### Golden troubleshooting sequence

```text
TIME
 ↓
timedatectl
 ↓
CHRONYD
 ↓
chronyc tracking
 ↓
NTP SOURCES
 ↓
chronyc sources -v
 ↓
CONFIG
 ↓
/etc/chrony.conf
 ↓
NETWORK
 ↓
UDP/123
 ↓
FIREWALL
 ↓
LOGS
 ↓
journalctl -u chronyd
 ↓
FIX
 ↓
chronyc makestep
 ↓
VERIFY
```
