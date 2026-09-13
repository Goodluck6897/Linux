
# L3 Interview Scenario: Linux Server Cannot Resolve db.prod.example.com

---

## Scenario

> **Interviewer:** "The application team says the Linux server cannot resolve `db.prod.example.com`. What will you do?"

---

## Phase 1: Reproduce the Issue

Before making any changes, confirm the problem exists from the affected server itself.

### Command 1 – `getent` (uses system resolver, respects nsswitch.conf)
```bash
getent hosts db.prod.example.com
```
- **Expected (working):** `10.20.30.40  db.prod.example.com`
- **Expected (broken):** No output or an error

### Command 2 – `dig` (queries DNS directly, bypasses /etc/hosts)
```bash
dig db.prod.example.com
```
- **Expected (working):**
```
;; ANSWER SECTION:
db.prod.example.com.  300  IN  A  10.20.30.40
;; SERVER: 10.10.10.10#53
```
- **Expected (broken):**
```
;; status: SERVFAIL   (or NXDOMAIN or connection timed out)
```

### Command 3 – `nslookup` (alternative quick check)
```bash
nslookup db.prod.example.com
```
- **Expected (broken):** `** server can't find db.prod.example.com: NXDOMAIN`

> **Key Insight:** If `getent` fails but `dig` works → the problem is in the local resolver config.
> If both fail → the problem is DNS server, network, or the record itself.

---

## Phase 2: Systematic Checks (10-Point Checklist)

### 1. Check `/etc/resolv.conf`
```bash
cat /etc/resolv.conf
```
**Example output (correct):**
```
nameserver 10.10.10.10
nameserver 10.10.10.11
search prod.example.com example.com
```
**Common problems:**
- Wrong or missing `nameserver` entries
- Missing `search` domain (so short names don't resolve)
- File overwritten by DHCP or NetworkManager
- Symlink broken (e.g., systemd-resolved manages it)

**Fix example:**
```bash
# If managed by systemd-resolved:
ls -la /etc/resolv.conf
# Output: /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

# Check actual resolved config:
resolvectl status
```

### 2. Check `/etc/nsswitch.conf`
```bash
grep hosts /etc/nsswitch.conf
```
**Example output (correct):**
```
hosts:  files dns myhostname
```
**Meaning:** System checks `/etc/hosts` first → then DNS → then local hostname.

**Common problems:**
- `dns` missing from the line → DNS is never queried
- Wrong order (e.g., `hosts: myhostname files` — DNS not listed)

**Fix example:**
```bash
# Ensure dns is present:
sudo vi /etc/nsswitch.conf
# Change to: hosts: files dns myhostname
```

### 3. Check `/etc/hosts`
```bash
cat /etc/hosts
```
**Example output:**
```
127.0.0.1   localhost
10.20.30.99 db.prod.example.com    # ← Stale/wrong entry!
```
**Common problems:**
- Hardcoded wrong IP for the hostname
- Stale entry overriding DNS

**Fix example:**
```bash
# Remove or correct the stale entry
sudo sed -i '/db.prod.example.com/d' /etc/hosts
```

### 4. Identify the Configured DNS Server
```bash
cat /etc/resolv.conf | grep nameserver
# OR (for systemd-resolved systems):
resolvectl status | grep "DNS Servers"
```
**Example output:**
```
nameserver 10.10.10.10
nameserver 10.10.10.11
```
> Note the DNS server IP — you'll need it for the next steps.

### 5. DNS Query Using `dig` (Default Server)
```bash
dig db.prod.example.com +short
```
**Example output (working):** `10.20.30.40`
**Example output (broken):** *(empty or SERVFAIL)*

**With full detail:**
```bash
dig db.prod.example.com +trace
```
This traces the full resolution path from root → TLD → authoritative server.

### 6. Direct Query to Specific DNS Server
```bash
dig @10.10.10.10 db.prod.example.com
```
**Example output (working):**
```
;; ANSWER SECTION:
db.prod.example.com.  300  IN  A  10.20.30.40
;; SERVER: 10.10.10.10#53
```
**Example output (broken):**
```
;; connection timed out; no servers could be reached
```

**Also try the secondary DNS:**
```bash
dig @10.10.10.11 db.prod.example.com
```

> **Key Insight:** If `dig @DNS_SERVER` works but `dig` alone doesn't → `/etc/resolv.conf` is pointing to the wrong server.

### 7. Network Connectivity to DNS Server
```bash
# Check routing
ip route
ip route get 10.10.10.10

# Ping the DNS server
ping -c 3 10.10.10.10

# Traceroute
traceroute 10.10.10.10
```
**Example output (working):**
```
10.10.10.10 via 10.20.30.1 dev eth0 src 10.20.30.50
```
**Example output (broken):**
```
RTNETLINK answers: Network is unreachable
```

### 8. Check UDP/TCP Port 53 Connectivity
```bash
# Test UDP port 53 (primary DNS transport)
nc -vzu 10.10.10.10 53

# Test TCP port 53 (used for large responses / zone transfers)
nc -vz 10.10.10.10 53
```
**Example output (working):**
```
Connection to 10.10.10.10 53 port [tcp/domain] succeeded!
```
**Example output (broken):**
```
nc: connect to 10.10.10.10 port 53 (tcp) failed: Connection timed out
```

**Alternative with `ss` or `telnet`:**
```bash
# Check if local DNS client port is open
ss -ulnp | grep 53

# Telnet test
telnet 10.10.10.10 53
```

### 9. Check Firewall Rules
```bash
# For firewalld (RHEL/CentOS 7+):
sudo firewall-cmd --list-all

# Check if DNS service is allowed:
sudo firewall-cmd --list-services | grep dns

# For iptables:
sudo iptables -L -n -v | grep 53

# For nftables:
sudo nft list ruleset | grep 53
```
**Example output (DNS blocked):**
```
DROP  udp  --  0.0.0.0/0  0.0.0.0/0  udp dpt:53
```

**Fix example:**
```bash
# Allow DNS through firewalld:
sudo firewall-cmd --add-service=dns --permanent
sudo firewall-cmd --reload

# Or with iptables:
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --dport 53 -j ACCEPT
```

> **Don't forget:** Also check **Security Groups / NACLs** if this is a cloud server (AWS, Azure, etc.)

### 10. Check DNS Records / Zone on the DNS Server
```bash
# Check if the record exists at all:
dig db.prod.example.com ANY

# Check the SOA (Start of Authority) for the zone:
dig prod.example.com SOA

# Check the authoritative nameserver:
dig prod.example.com NS
```
**Example output (record missing):**
```
;; status: NXDOMAIN   ← Record does not exist in the zone
```

**If you have access to the DNS server (e.g., BIND):**
```bash
# Check zone file:
sudo cat /var/named/prod.example.com.zone

# Look for the A record:
grep "db" /var/named/prod.example.com.zone
# Expected: db  IN  A  10.20.30.40

# If record was recently added, check if zone was reloaded:
sudo rndc reload prod.example.com
```

---

## Phase 3: Root Cause Determination (Decision Tree)

| Symptom | Root Cause | Fix |
|---|---|---|
| `getent` fails, `dig` works | Local resolver misconfiguration | Fix `/etc/nsswitch.conf` or `/etc/hosts` |
| `dig` fails, `dig @DNS_SERVER` works | Wrong nameserver in `/etc/resolv.conf` | Update `/etc/resolv.conf` |
| `dig @DNS_SERVER` fails, `ping` works | DNS service down or record missing | Restart DNS service; add record |
| `ping` to DNS fails, `ip route` OK | Firewall blocking traffic | Open port 53 UDP/TCP |
| `ip route get` fails | Network/routing issue | Fix routing table or interface config |
| `dig` returns `NXDOMAIN` everywhere | DNS record/zone not configured | Add A record to DNS zone |
| `dig` returns `SERVFAIL` | DNS server error (zone misconfigured) | Check zone file syntax, reload zone |

---

## Phase 4: Final Verification

After applying the fix, verify end-to-end:

### Step 1 – DNS Resolution
```bash
getent hosts db.prod.example.com
# Expected: 10.20.30.40  db.prod.example.com

dig db.prod.example.com +short
# Expected: 10.20.30.40
```

### Step 2 – Application-Level Connectivity
```bash
# Test database port (e.g., MySQL 3306, PostgreSQL 5432)
nc -vz db.prod.example.com 3306
# Expected: Connection to db.prod.example.com 3306 port [tcp/mysql] succeeded!

# Or use the application's own test:
mysql -h db.prod.example.com -u appuser -p -e "SELECT 1;"
# Expected: Connected successfully
```

### Step 3 – Confirm with the Application Team
```bash
# Restart the application if it caches DNS:
sudo systemctl restart myapp.service

# Check application logs:
tail -f /var/log/myapp/application.log
```

---

## Quick Reference: Command Cheat Sheet

| Step | Command | Purpose |
|---|---|---|
| Reproduce | `getent hosts db.prod.example.com` | Test system resolver |
| Reproduce | `dig db.prod.example.com` | Test DNS directly |
| Resolver | `cat /etc/resolv.conf` | Check DNS server config |
| NSSwitch | `grep hosts /etc/nsswitch.conf` | Check resolution order |
| Hosts file | `cat /etc/hosts` | Check static entries |
| Direct DNS | `dig @10.10.10.10 db.prod.example.com` | Query specific DNS server |
| Network | `ip route get 10.10.10.10` | Check routing |
| Port 53 | `nc -vz 10.10.10.10 53` | Test DNS port |
| Firewall | `firewall-cmd --list-all` | Check firewall rules |
| Zone/Record | `dig prod.example.com SOA` | Check DNS zone |
| Verify | `nc -vz db.prod.example.com 3306` | Test app connectivity |

---

## Interview Tips

- **Always reproduce first** — never assume the report is accurate.
- **Work layer by layer** — local config → network → DNS server → DNS record.
- **Explain your reasoning** — say *why* you're running each command.
- **Mention cloud context** — if applicable, mention Security Groups, VPC DNS settings (e.g., AWS `enableDnsSupport`), Route 53 private hosted zones.
- **End with verification** — always confirm the fix works at both the DNS and application level.
- **Communicate** — mention that you'd keep the application team informed throughout the process.


############
# 50. Golden Rule for L3 DNS Troubleshooting

Always separate **name resolution** from **network/application connectivity**:

```text
             Application Issue
                    |
                    v
             Can DNS resolve?
              /            \
            NO              YES
            |                |
            v                v
      Troubleshoot DNS   Check connectivity
                             |
                             v
                       Correct IP/Port?
                         /        \
                       NO          YES
                       |            |
                       v            v
                  Network/DNS   Application
                    issue         issue
```

The key commands to remember for the interview are:

```bash
cat /etc/resolv.conf
grep '^hosts:' /etc/nsswitch.conf
getent hosts hostname
dig hostname
dig @DNS_SERVER hostname
dig -x IP
ip route
nc -vz DNS_SERVER 53
```
###############
