Absolutely — here is a **GitHub-ready `README.md` style** version focused on **Linux Administrator / L3 interview preparation**.

# Linux DNS — L3 System Administrator Interview Notes

DNS (**Domain Name System**) is one of the most important networking topics for a Linux System Administrator.

For an L3 interview, you should understand:

* DNS resolution flow
* `/etc/resolv.conf`
* `/etc/hosts`
* `/etc/nsswitch.conf`
* DNS record types
* `dig`, `nslookup`, `host`, `getent`
* Recursive vs authoritative DNS
* DNS ports
* DNS troubleshooting
* DNS timeout scenarios
* Forward and reverse DNS
* DNS TTL and caching

---

# 1. What is DNS?

DNS converts hostnames into IP addresses.

```text
www.example.com
       |
       v
      DNS
       |
       v
  10.10.10.50
```

Example:

```bash
ping google.com
```

Linux needs DNS to resolve:

```text
google.com → IP address
```

---

# 2. Linux DNS Resolution Flow

When an application needs to resolve a hostname:

```text
Application
     |
     v
glibc resolver
     |
     v
/etc/nsswitch.conf
     |
     v
/etc/hosts
     |
     | Not found
     v
DNS resolver
     |
     v
/etc/resolv.conf
     |
     v
DNS Server
     |
     v
IP Address
```

Example:

```text
Application
    |
    v
db.example.com
    |
    v
Linux Resolver
    |
    v
DNS Server
    |
    v
10.10.10.50
```

---

# 3. Important Linux DNS Files

## `/etc/resolv.conf`

Contains DNS resolver configuration.

```bash
cat /etc/resolv.conf
```

Example:

```text
search example.com
nameserver 10.10.10.10
nameserver 10.10.10.11
```

### Important parameters

| Parameter    | Purpose               |
| ------------ | --------------------- |
| `nameserver` | DNS server IP address |
| `search`     | Search domain         |
| `options`    | Resolver options      |

Example:

```text
nameserver 10.10.10.10
```

means:

> Use `10.10.10.10` as a DNS resolver.

---

# 4. `/etc/hosts`

Contains local static hostname-to-IP mappings.

```bash
cat /etc/hosts
```

Example:

```text
127.0.0.1       localhost
10.10.10.50     app01.example.com app01
10.10.10.60     db01.example.com db01
```

Now:

```bash
ping app01
```

can resolve using `/etc/hosts`.

---

# 5. `/etc/nsswitch.conf`

Controls how Linux performs name-service lookups.

```bash
cat /etc/nsswitch.conf
```

Look for:

```text
hosts: files dns myhostname
```

This indicates possible lookup sources/order:

```text
files
  |
  v
/etc/hosts
  |
  v
dns
  |
  v
DNS server
```

Example:

```text
hosts: files dns
```

Linux first checks:

```text
/etc/hosts
```

If the hostname is not found, it queries DNS.

---

# 6. `/etc/hosts` vs `/etc/resolv.conf`

| File                 | Purpose                            |
| -------------------- | ---------------------------------- |
| `/etc/hosts`         | Local static hostname mappings     |
| `/etc/resolv.conf`   | DNS resolver configuration         |
| `/etc/nsswitch.conf` | Controls name-service lookup order |

### Interview Question

**Q: What is the difference between `/etc/hosts` and `/etc/resolv.conf`?**

### Answer

`/etc/hosts` contains local hostname-to-IP mappings, while `/etc/resolv.conf` specifies DNS servers and resolver settings used to resolve names through DNS.

---

# 7. DNS Record Types

Know these for interviews:

| Record | Purpose                    |
| ------ | -------------------------- |
| A      | Hostname → IPv4            |
| AAAA   | Hostname → IPv6            |
| CNAME  | Alias → hostname           |
| MX     | Mail server                |
| NS     | Authoritative name server  |
| PTR    | IP → hostname              |
| TXT    | Text information           |
| SOA    | Zone authority information |
| SRV    | Service location           |

---

# 8. A Record

Maps hostname to IPv4.

```text
www.example.com → 10.10.10.20
```

Query:

```bash
dig A www.example.com
```

---

# 9. AAAA Record

Maps hostname to IPv6.

```bash
dig AAAA www.example.com
```

---

# 10. CNAME Record

CNAME creates an alias.

```text
app.example.com
       |
       v
server01.example.com
       |
       v
10.10.10.50
```

Example:

```text
app.example.com CNAME server01.example.com
```

Query:

```bash
dig CNAME app.example.com
```

---

# 11. MX Record

MX records identify mail servers.

```bash
dig MX example.com
```

Example:

```text
example.com
     |
     v
mail.example.com
```

---

# 12. NS Record

NS records identify authoritative DNS servers.

```bash
dig NS example.com
```

Example:

```text
example.com
     |
     +---- ns1.example.com
     |
     +---- ns2.example.com
```

---

# 13. PTR Record

PTR is used for reverse DNS.

```text
IP Address → Hostname
```

Example:

```bash
dig -x 10.10.10.50
```

Forward DNS:

```text
server01.example.com
        |
        v
10.10.10.50
```

Reverse DNS:

```text
10.10.10.50
        |
        v
server01.example.com
```

---

# 14. TXT Record

TXT records contain text information.

```bash
dig TXT example.com
```

Common uses include:

* SPF
* Domain verification
* Email security
* Other application metadata

---

# 15. SOA Record

SOA = **Start of Authority**

Contains important information about a DNS zone.

```bash
dig SOA example.com
```

It includes information such as:

* Primary authoritative server
* Responsible administrator
* Serial number
* Refresh
* Retry
* Expire
* Minimum/negative caching information

---

# 16. SRV Record

SRV records identify services.

Conceptually:

```text
_service._protocol.example.com
```

Example:

```bash
dig SRV _ldap._tcp.example.com
```

Commonly used by services such as:

* LDAP
* Kerberos
* Active Directory
* Other service-discovery systems

---

# 17. DNS Tools

The most important commands for Linux administrators:

```text
dig
nslookup
host
getent
resolvectl
```

---

# 18. `dig`

`dig` is one of the most useful DNS troubleshooting commands.

Basic lookup:

```bash
dig google.com
```

A record:

```bash
dig A google.com
```

AAAA:

```bash
dig AAAA google.com
```

MX:

```bash
dig MX google.com
```

NS:

```bash
dig NS google.com
```

TXT:

```bash
dig TXT google.com
```

PTR:

```bash
dig -x 8.8.8.8
```

---

# 19. Query a Specific DNS Server

This is extremely important during troubleshooting.

```bash
dig @10.10.10.10 myapp.example.com
```

Meaning:

```text
Ask DNS server 10.10.10.10:

"What is the IP address of myapp.example.com?"
```

This helps determine whether the problem is with:

* Linux resolver
* DNS configuration
* Specific DNS server
* Network connectivity

---

# 20. Useful `dig` Commands

```bash
dig google.com
```

```bash
dig A google.com
```

```bash
dig AAAA google.com
```

```bash
dig MX google.com
```

```bash
dig NS google.com
```

```bash
dig TXT google.com
```

```bash
dig SOA google.com
```

```bash
dig -x 8.8.8.8
```

Specific DNS server:

```bash
dig @8.8.8.8 google.com
```

Short answer:

```bash
dig +short google.com
```

---

# 21. `nslookup`

Simple DNS lookup.

```bash
nslookup google.com
```

Specific DNS server:

```bash
nslookup google.com 8.8.8.8
```

Reverse lookup:

```bash
nslookup 8.8.8.8
```

---

# 22. `host`

Simple hostname lookup.

```bash
host google.com
```

Reverse lookup:

```bash
host 8.8.8.8
```

---

# 23. `getent hosts`

Very useful for Linux troubleshooting.

```bash
getent hosts google.com
```

Unlike `dig`, `getent` uses the system's configured name-service mechanisms.

Therefore compare:

```bash
dig google.com
```

with:

```bash
getent hosts google.com
```

This can help identify whether the issue is:

```text
DNS server
```

or:

```text
Linux NSS/resolver configuration
```

---

# 24. `nmcli

# List connections
nmcli connection show

# List active connections
nmcli connection show --active

# List network devices
nmcli device status

# Detailed device information
nmcli device show ens160

# Detailed connection information
nmcli connection show ens160

# Bring connection up
nmcli connection up ens160

# Bring connection down
nmcli connection down ens160

# Connect device
nmcli device connect ens160

# Disconnect device
nmcli device disconnect ens160

# Modify connection
nmcli connection modify ens160 ...

# Add connection
nmcli connection add ...

# Delete connection
nmcli connection delete ens160
The complete troubleshooting flow

This is what I want you to memorize for your Linux interview:
             INTERNET NOT WORKING
                     |
                     v
          nmcli device status
                     |
                     v
             Is interface UP?
               /          \
             NO            YES
             |              |
      nmcli con up       Check IP
                           |
                           v
             nmcli dev show ens160
                           |
                           v
                     Check gateway
                           |
                           v
                       ip route
                           |
                           v
                  ping <gateway>
                    /          \
                  FAIL         OK
                   |            |
             Fix network      ping 8.8.8.8
             /gateway            |
                                 v
                         Does 8.8.8.8 work?
                           /          \
                         NO            YES
                         |              |
                   Routing/firewall   DNS problem
                                      |
                                      v
                              nmcli dev show
                                      |
                                      v
                                      DNS
                                      |
                                      v
                              nslookup google.com

---

# 25. DNS Ports

DNS primarily uses:

```text
UDP 53
TCP 53
```

### UDP 53

Normal DNS queries commonly use UDP.

### TCP 53

TCP can be required for:

* Large DNS responses
* DNS responses that require TCP fallback
* Zone transfers
* Other DNS operations

---

# 26. Check DNS Port

Check UDP:

```bash
ss -lunp | grep ':53'
```

Check TCP:

```bash
ss -ltnp | grep ':53'
```

Test connectivity:

```bash
nc -vz 10.10.10.10 53
```

Remember:

> Testing TCP port 53 alone does not prove UDP DNS connectivity works.

---

# 27. Recursive DNS

A recursive DNS server finds the answer on behalf of the client.

Example:

```text
Linux Server
     |
     v
Recursive DNS
     |
     v
Root DNS
     |
     v
.com DNS
     |
     v
Authoritative DNS
     |
     v
Answer
```

The client normally asks the recursive resolver:

```text
"What is the IP of google.com?"
```

---

# 28. Authoritative DNS

An authoritative DNS server contains the actual DNS records for a zone.

Example:

```text
example.com
```

Authoritative DNS may contain:

```text
www.example.com → 10.10.10.20
db.example.com  → 10.10.10.30
app.example.com → 10.10.10.40
```

---

# 29. Recursive vs Authoritative DNS

| Recursive                 | Authoritative                      |
| ------------------------- | ---------------------------------- |
| Finds answers for clients | Owns DNS zone data                 |
| Performs recursion        | Provides authoritative answers     |
| Caches results            | Maintains zone records             |
| Usually used by clients   | Usually used by DNS infrastructure |

---

# 30. DNS Resolution Hierarchy

For:

```text
www.example.com
```

The DNS hierarchy is approximately:

```text
                    .
                    |
                 Root DNS
                    |
                   .com
                    |
              example.com
                    |
          Authoritative DNS
                    |
             www.example.com
                    |
                 IP Address
```

Caching can eliminate some of these queries.

---

# 31. DNS TTL

TTL = **Time To Live**

Example:

```text
www.example.com
A
10.10.10.50
TTL = 300
```

The record can generally be cached for:

```text
300 seconds
```

### Low TTL

```text
Fast propagation of changes
More DNS queries
```

### High TTL

```text
Longer caching
Fewer DNS queries
```

---

# 32. DNS Search Domain

Example `/etc/resolv.conf`:

```text
search example.com
```

Then:

```bash
ping server01
```

may result in a lookup such as:

```text
server01.example.com
```

Search domains are common in enterprise environments.

---

# 33. DNS Troubleshooting

## Scenario

Application team reports:

```text
"Linux server cannot resolve db.example.com"
```

Follow this sequence:

```text
1. Check /etc/resolv.conf
          |
          v
2. Check /etc/nsswitch.conf
          |
          v
3. getent hosts db.example.com
          |
          v
4. dig db.example.com
          |
          v
5. dig @DNS_SERVER db.example.com
          |
          v
6. Check network connectivity
          |
          v
7. Check UDP/TCP 53
          |
          v
8. Check firewall
          |
          v
9. Check DNS server
          |
          v
10. Check DNS records
```

---

# 34. Step 1 — Check `/etc/resolv.conf`

```bash
cat /etc/resolv.conf
```

Look for:

```text
nameserver 10.10.10.10
nameserver 10.10.10.11
```

Questions:

* Is the DNS server correct?
* Is it reachable?
* Is there a valid search domain?
* Is the configuration being managed dynamically?

---

# 35. Step 2 — Check NSS

```bash
grep '^hosts:' /etc/nsswitch.conf
```

Example:

```text
hosts: files dns
```

Check whether `/etc/hosts` contains a conflicting entry:

```bash
grep -w 'db.example.com' /etc/hosts
```

---

# 36. Step 3 — Test Using `getent`

```bash
getent hosts db.example.com
```

If it returns:

```text
10.10.10.50    db.example.com
```

the Linux name-service configuration can resolve the hostname.

---

# 37. Step 4 — Test Using `dig`

```bash
dig db.example.com
```

Look at:

```text
ANSWER SECTION
```

Example:

```text
db.example.com.    300    IN    A    10.10.10.50
```

---

# 38. Step 5 — Query DNS Directly

```bash
dig @10.10.10.10 db.example.com
```

This bypasses ambiguity about which resolver is being used.

If this works:

```text
dig @10.10.10.10 db.example.com
        |
        v
DNS server responds
```

but:

```bash
dig db.example.com
```

fails, investigate the local resolver configuration.

---

# 39. DNS Timeout Scenario

Example:

```bash
dig db.example.com
```

returns something similar to:

```text
communications error to 10.10.10.10#53: timed out
```

Investigate:

```text
Linux
  |
  +-- DNS server reachable?
  |
  +-- Routing?
  |
  +-- Firewall?
  |
  +-- UDP 53?
  |
  +-- TCP 53?
  |
  +-- DNS service running?
```

Check route:

```bash
ip route
```

Check connectivity:

```bash
ping 10.10.10.10
```

Check TCP port:

```bash
nc -vz 10.10.10.10 53
```

Check firewall:

```bash
firewall-cmd --list-all
```

---

# 40. IP Works but Hostname Fails

Example:

```bash
ping 10.10.10.50
```

works.

But:

```bash
ping app.example.com
```

fails.

Likely area:

```text
DNS / hostname resolution
```

Check:

```bash
cat /etc/resolv.conf
```

```bash
cat /etc/nsswitch.conf
```

```bash
getent hosts app.example.com
```

```bash
dig app.example.com
```

```bash
dig @DNS_SERVER app.example.com
```

---

# 41. DNS Works but Application Fails

Example:

```bash
dig db.example.com
```

returns:

```text
10.10.10.50
```

DNS is working.

But:

```bash
nc -vz db.example.com 3306
```

fails.

Do not continue troubleshooting DNS blindly.

Now investigate:

```text
DNS
 |
 +-- ✓ Resolution
 |
 v
10.10.10.50
 |
 X
TCP 3306
```

Check:

```bash
ip route
```

```bash
ss -lntp
```

```bash
firewall-cmd --list-all
```

```bash
nc -vz 10.10.10.50 3306
```

Possible causes:

* Firewall
* Routing
* Application down
* Wrong port
* Security group/network ACL
* Service not listening

---

# 42. DNS Cache

Caching can cause stale DNS results.

Possible caching components include:

```text
systemd-resolved
nscd
dnsmasq
```

Check what is actually running.

For example:

```bash
systemctl status nscd
```

or:

```bash
systemctl status systemd-resolved
```

Do not assume a particular caching service exists.

---

# 43. Forward vs Reverse DNS

### Forward DNS

```text
Hostname
   |
   v
IP
```

Example:

```text
server01.example.com
        ↓
10.10.10.50
```

Test:

```bash
dig server01.example.com
```

### Reverse DNS

```text
IP
 |
 v
Hostname
```

Test:

```bash
dig -x 10.10.10.50
```

---

# 44. Common DNS Errors

## NXDOMAIN

```text
NXDOMAIN
```

Means the queried DNS name does not exist according to the responding DNS server.

Investigate:

* Typo in hostname
* Wrong DNS zone
* Missing DNS record
* Wrong search domain
* DNS delegation

---

## SERVFAIL

```text
SERVFAIL
```

Generally means the DNS server could not successfully complete the query.

Possible causes include:

* DNSSEC problems
* Broken delegation
* Authoritative DNS problems
* Upstream DNS failures
* DNS server configuration issues

---

## REFUSED

```text
REFUSED
```

The DNS server refuses to answer the query, often due to configuration or access policy.

---

## TIMEOUT

```text
timed out
```

Investigate:

```text
Network
Firewall
Routing
UDP/TCP 53
DNS server availability
```

---

# 45. DNS Troubleshooting Decision Tree

```text
                 DNS Failure
                     |
                     v
          Can hostname resolve?
                /        \
              YES         NO
               |           |
               v           v
       Check application   Check resolver
               |           |
               |       /etc/resolv.conf
               |           |
               |       /etc/nsswitch.conf
               |           |
               |        getent hosts
               |           |
               |          dig
               |           |
               |     dig @DNS_SERVER
               |           |
               |           v
               |      DNS server?
               |       /       \
               |     YES        NO
               |      |          |
               |      v          v
               |    Records    Network/
               |    /Zone      Firewall
               |
               v
         Check TCP/UDP
         application port
```

---

# 46. Production DNS Troubleshooting Example

### Problem

Application cannot connect to:

```text
db.prod.example.com
```

### Step 1

```bash
getent hosts db.prod.example.com
```

### Step 2

```bash
dig db.prod.example.com
```

### Step 3

```bash
dig @10.10.10.10 db.prod.example.com
```

### Step 4

If DNS works:

```bash
nc -vz db.prod.example.com 3306
```

### Step 5

If DNS fails:

```bash
cat /etc/resolv.conf
```

```bash
grep '^hosts:' /etc/nsswitch.conf
```

### Step 6

Check network:

```bash
ip route
```

### Step 7

Check firewall:

```bash
firewall-cmd --list-all
```

### Step 8

Verify DNS server and DNS record.

---

# 47. L3 Interview Questions

## Q1. What is DNS?

DNS translates hostnames into IP addresses and provides other DNS information through records.

---

## Q2. What is `/etc/resolv.conf`?

It contains DNS resolver configuration such as:

```text
nameserver
search
options
```

---

## Q3. What is `/etc/hosts`?

Local static hostname-to-IP mapping.

---

## Q4. What is `/etc/nsswitch.conf`?

It controls the sources/order Linux uses for name-service lookups.

---

## Q5. What is an A record?

```text
Hostname → IPv4
```

---

## Q6. What is an AAAA record?

```text
Hostname → IPv6
```

---

## Q7. What is a CNAME?

An alias from one hostname to another hostname.

---

## Q8. What is an MX record?

Identifies mail servers for a domain.

---

## Q9. What is an NS record?

Identifies authoritative DNS servers for a DNS zone.

---

## Q10. What is a PTR record?

Used for reverse DNS:

```text
IP → hostname
```

---

## Q11. Which command do you use to troubleshoot DNS?

Primary tool:

```bash
dig
```

Other useful commands:

```bash
nslookup
host
getent
resolvectl
```

---

## Q12. How do you query a specific DNS server?

```bash
dig @10.10.10.10 hostname.example.com
```

---

## Q13. What ports does DNS use?

```text
UDP 53
TCP 53
```

---

## Q14. What is recursive DNS?

A recursive resolver performs DNS lookups on behalf of clients and commonly caches the results.

---

## Q15. What is authoritative DNS?

An authoritative DNS server provides authoritative answers for zones it hosts.

---

# 48. ⭐ L3 Interview Scenario

### Interviewer:

> "The application team says the Linux server cannot resolve `db.prod.example.com`. What will you do?"

### Strong Answer

```text
First, I will reproduce the issue using getent and dig.

Then I will check:

1. /etc/resolv.conf
2. /etc/nsswitch.conf
3. /etc/hosts
4. DNS server configured
5. DNS query using dig
6. Direct query using dig @DNS_SERVER
7. Network connectivity
8. UDP/TCP port 53
9. Firewall
10. DNS records / zone
```

Example:

```bash
getent hosts db.prod.example.com

dig db.prod.example.com

dig @10.10.10.10 db.prod.example.com

ip route

nc -vz 10.10.10.10 53

firewall-cmd --list-all
```

Then determine whether the problem is:

```text
Linux resolver
      OR
Network
      OR
Firewall
      OR
DNS server
      OR
DNS record/zone
```

Finally, verify:

```bash
getent hosts db.prod.example.com
```

and test the application connection separately.

---

# 49. Quick Revision Cheat Sheet

```text
=================================================
             LINUX DNS CHEAT SHEET
=================================================

Important files:

/etc/hosts
/etc/resolv.conf
/etc/nsswitch.conf

Important commands:

dig
nslookup
host
getent
resolvectl

DNS records:

A       → IPv4
AAAA    → IPv6
CNAME   → Alias
MX      → Mail
NS      → Name server
PTR     → Reverse DNS
TXT     → Text
SOA     → Zone authority
SRV     → Service

DNS ports:

UDP 53
TCP 53

Forward DNS:

Hostname → IP

Reverse DNS:

IP → Hostname

Important commands:

cat /etc/resolv.conf

cat /etc/hosts

grep '^hosts:' /etc/nsswitch.conf

getent hosts hostname

dig hostname

dig @DNS_SERVER hostname

dig -x IP

dig +short hostname

ip route

nc -vz DNS_SERVER 53

ss -lunp | grep ':53'

ss -ltnp | grep ':53'

=================================================
```

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
