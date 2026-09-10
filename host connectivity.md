Absolutely. For a **Linux L3/System Administrator interview**, host connectivity means being able to troubleshoot why **one Linux server cannot communicate with another server, service, or the Internet**.

# Linux Host Connectivity — GitHub Notes

## 1. What is Host Connectivity?

Host connectivity is the ability of one host to communicate with another host over a network.

```text
+------------+       Network       +------------+
| Linux Host | ------------------> | Remote Host|
| 10.10.1.10 |                     | 10.10.1.20 |
+------------+                     +------------+
      |
      +-- Interface
      +-- IP address
      +-- Routing
      +-- ARP
      +-- DNS
      +-- Firewall
      +-- TCP/UDP
```

For troubleshooting, follow this order:

```text
Application
    ↓
DNS
    ↓
Port / TCP
    ↓
Firewall
    ↓
Routing
    ↓
ARP
    ↓
Network Interface
    ↓
Physical / Cloud Network
```

---

# 2. Basic Connectivity Commands

## Check IP address

```bash
ip addr
```

Short form:

```bash
ip a
```

Example:

```text
ens192:
    inet 10.10.1.10/24
```

Check a specific interface:

```bash
ip addr show ens192
```

---

## Check network interfaces

```bash
ip link
```

Look for:

```text
state UP
```

If interface is down:

```bash
ip link set ens192 up
```

---

# 3. Check Routing

```bash
ip route
```

Example:

```text
default via 10.10.1.1 dev ens192
10.10.1.0/24 dev ens192 proto kernel scope link src 10.10.1.10
```

Important:

```text
10.10.1.0/24
       ↓
Connected network

default via 10.10.1.1
       ↓
Default gateway
```

Check route to a specific destination:

```bash
ip route get 10.10.2.20
```

Example:

```text
10.10.2.20 via 10.10.1.1 dev ens192 src 10.10.1.10
```

This is a **very useful L3 troubleshooting command**.

---

# 4. Test Basic Connectivity — ping

```bash
ping 10.10.1.20
```

Test a specific number of packets:

```bash
ping -c 4 10.10.1.20
```

Test gateway:

```bash
ping -c 4 10.10.1.1
```

Test Internet:

```bash
ping -c 4 8.8.8.8
```

### Troubleshooting sequence

```text
ping gateway
      ↓
Works?
      ↓
   YES
      ↓
ping remote host
      ↓
Works?
      ↓
   YES
      ↓
ping 8.8.8.8
```

If gateway itself cannot be reached, investigate:

* Interface
* IP address
* subnet mask
* VLAN
* ARP
* switch/network
* firewall

---

# 5. ARP / Neighbor Table

For IPv4, Linux needs the MAC address of a local next-hop host.

Check neighbor table:

```bash
ip neigh
```

Example:

```text
10.10.1.1 dev ens192 lladdr 00:11:22:33:44:55 REACHABLE
```

Check a specific host:

```bash
ip neigh show 10.10.1.1
```

States include:

```text
REACHABLE
STALE
DELAY
PROBE
FAILED
INCOMPLETE
```

### Important interview point

If:

```bash
ping 10.10.1.1
```

fails and:

```bash
ip neigh
```

shows:

```text
10.10.1.1 INCOMPLETE
```

Linux is unable to resolve the destination's MAC address.

Investigate:

```text
NIC
  ↓
VLAN
  ↓
Switch
  ↓
Subnet
  ↓
ARP
```

---

# 6. DNS Troubleshooting

First test IP connectivity:

```bash
ping 8.8.8.8
```

Then test DNS:

```bash
getent hosts google.com
```

or:

```bash
dig google.com
```

Check DNS configuration:

```bash
cat /etc/resolv.conf
```

Example:

```text
nameserver 10.10.1.53
nameserver 8.8.8.8
```

On RHEL 8, also check NetworkManager:

```bash
nmcli dev show
```

Look for:

```text
IP4.DNS
```

### Important distinction

```text
ping 8.8.8.8
     ↓
works

ping google.com
     ↓
fails
```

Likely problem:

```text
DNS
```

Not basic IP connectivity.

---

# 7. Test TCP Port Connectivity

`ping` only tests ICMP. It does **not** prove that an application port is reachable.

For example:

```bash
nc -zv 10.10.1.20 443
```

or:

```bash
nc -zv 10.10.1.20 8080
```

Example successful result:

```text
Connection to 10.10.1.20 443 port [tcp/https] succeeded!
```

Failed:

```text
Connection refused
```

or:

```text
Connection timed out
```

These mean different things.

### Connection refused

Usually:

```text
Host reachable
     ↓
TCP reached destination
     ↓
Nothing listening on that port
```

Check server:

```bash
ss -lntp
```

### Connection timed out

Could indicate:

```text
Firewall
Routing
Security Group
Network ACL
Network problem
```

---

# 8. Check Listening Ports

```bash
ss -lnt
```

With process information:

```bash
ss -lntp
```

Example:

```text
LISTEN 0 128 0.0.0.0:8080
```

Means application is listening on port `8080` on all IPv4 interfaces.

Check a specific port:

```bash
ss -lntp | grep :8080
```

---

# 9. Check Established Connections

```bash
ss -nt
```

Example:

```text
ESTAB 0 0 10.10.1.10:45000 10.10.1.20:443
```

Useful for troubleshooting:

```bash
ss -ant
```

Look for:

```text
ESTAB
SYN-SENT
SYN-RECV
TIME-WAIT
CLOSE-WAIT
```

---

# 10. TCP Connection Troubleshooting

Suppose:

```bash
nc -zv 10.10.1.20 443
```

times out.

Think about the TCP handshake:

```text
Client                         Server

SYN ------------------------>

       <-------------------- SYN-ACK

ACK ------------------------>
```

If SYN leaves but no SYN-ACK returns:

```text
Client
  |
  | SYN
  ↓
Network
  |
  X
  |
Server
```

Investigate:

* Firewall
* Routing
* Security Group
* Network ACL
* Server firewall
* Server application

---

# 11. Traceroute

Find the path to a remote host:

```bash
traceroute 10.10.2.20
```

If not installed:

```bash
dnf install traceroute
```

Alternative:

```bash
tracepath 10.10.2.20
```

Useful for identifying where traffic stops.

```text
Host
 ↓
Router 1
 ↓
Router 2
 ↓
Router 3
 X
Remote network
```

---

# 12. Check Gateway

```bash
ip route | grep default
```

Example:

```text
default via 10.10.1.1 dev ens192
```

Test gateway:

```bash
ping -c 4 10.10.1.1
```

If gateway is unreachable:

```text
Check:
  ↓
IP address
  ↓
Subnet mask
  ↓
Interface
  ↓
ARP
  ↓
VLAN
  ↓
Network connection
```

---

# 13. Hostname Resolution

Check hostname:

```bash
hostname
```

Fully qualified hostname:

```bash
hostname -f
```

Check resolution:

```bash
getent hosts server01.example.com
```

Check `/etc/hosts`:

```bash
cat /etc/hosts
```

Example:

```text
10.10.1.20 server01.example.com server01
```

### Resolution order

Check:

```bash
cat /etc/nsswitch.conf
```

Look for:

```text
hosts: files dns
```

This means:

```text
/etc/hosts
     ↓
DNS
```

---

# 14. Firewall — firewalld

Check status:

```bash
systemctl status firewalld
```

Check active zones:

```bash
firewall-cmd --get-active-zones
```

List rules:

```bash
firewall-cmd --list-all
```

List ports:

```bash
firewall-cmd --list-ports
```

List services:

```bash
firewall-cmd --list-services
```

Example:

```bash
firewall-cmd --list-ports
```

Output:

```text
8080/tcp 8443/tcp
```

---

# 15. SELinux

Sometimes the network connection reaches the server but the application still cannot operate correctly.

Check:

```bash
getenforce
```

Check logs:

```bash
ausearch -m AVC -ts recent
```

Important:

```text
Network connectivity
        ↓
Firewall
        ↓
SELinux
        ↓
Application
```

Don't immediately disable SELinux.

Avoid:

```bash
setenforce 0
```

as a permanent solution.

---

# 16. Check NetworkManager

On RHEL 8:

```bash
systemctl status NetworkManager
```

Check devices:

```bash
nmcli device status
```

Check connections:

```bash
nmcli connection show
```

Show detailed device information:

```bash
nmcli device show ens192
```

---

# 17. Check Interface Errors

```bash
ip -s link
```

Look for:

```text
RX errors
TX errors
dropped
overruns
carrier
```

More detailed:

```bash
ethtool ens192
```

Check NIC statistics:

```bash
ethtool -S ens192
```

Potential issues:

```text
Packet drops
CRC errors
Link problems
Duplex problems
NIC errors
```

---

# 18. Packet Capture — tcpdump

One of the most important L3 troubleshooting tools.

```bash
tcpdump -i ens192
```

Capture traffic to a host:

```bash
tcpdump -i ens192 host 10.10.1.20
```

Capture port 443:

```bash
tcpdump -i ens192 port 443
```

Capture ICMP:

```bash
tcpdump -i ens192 icmp
```

Capture TCP:

```bash
tcpdump -i ens192 tcp
```

More readable:

```bash
tcpdump -i ens192 -nn host 10.10.1.20
```

### Example

You run:

```bash
nc -zv 10.10.1.20 443
```

and it times out.

Run:

```bash
tcpdump -i ens192 -nn host 10.10.1.20 and port 443
```

If you see:

```text
10.10.1.10 > 10.10.1.20: SYN
10.10.1.10 > 10.10.1.20: SYN
10.10.1.10 > 10.10.1.20: SYN
```

but no:

```text
10.10.1.20 > 10.10.1.10: SYN, ACK
```

then investigate the path/server/firewall.

---

# 19. Important Troubleshooting Flow

Use this in an interview:

```text
              Connectivity Problem
                       |
                       v
              Check interface
                 ip link
                       |
                       v
                Check IP address
                  ip addr
                       |
                       v
                Check routing
                 ip route
                       |
                       v
                 Check ARP
                  ip neigh
                       |
                       v
                Ping gateway
                       |
                       v
                Ping remote IP
                       |
                       v
                  Check DNS
              getent / dig
                       |
                       v
               Check TCP port
                  nc / curl
                       |
                       v
              Check firewall
               firewalld
                       |
                       v
              Check application
                 ss -lntp
                       |
                       v
              Packet capture
                 tcpdump
```

---

# 20. `curl` for Application Connectivity

For HTTP/HTTPS:

```bash
curl -v http://10.10.1.20:8080
```

HTTPS:

```bash
curl -vk https://server.example.com
```

Useful because `curl` can show:

```text
DNS resolution
TCP connection
TLS handshake
HTTP response
```

For example:

```bash
curl -v https://example.com
```

can help distinguish:

```text
DNS problem
      ↓
TCP problem
      ↓
TLS problem
      ↓
HTTP/application problem
```

---

# 21. Common Interview Scenarios

## Scenario 1 — Cannot ping remote server

Check:

```bash
ip addr
ip link
ip route
ip neigh
ping gateway
ping remote-ip
```

Then:

```bash
traceroute remote-ip
tcpdump -i ens192 host remote-ip
```

---

## Scenario 2 — Ping works but application doesn't

Example:

```bash
ping 10.10.1.20
```

works.

But:

```bash
curl http://10.10.1.20:8080
```

fails.

Check:

```bash
nc -zv 10.10.1.20 8080
```

On server:

```bash
ss -lntp | grep :8080
```

Then check:

```bash
firewall-cmd --list-all
```

and application logs.

---

## Scenario 3 — IP works but hostname doesn't

```bash
ping 10.10.1.20
```

works.

But:

```bash
ping server01.example.com
```

fails.

Focus on:

```text
DNS
```

Check:

```bash
cat /etc/resolv.conf
getent hosts server01.example.com
dig server01.example.com
cat /etc/nsswitch.conf
```

---

## Scenario 4 — Port times out

```bash
nc -zv 10.10.1.20 443
```

times out.

Check:

```text
1. Routing
2. Firewall
3. Security Group / ACL
4. Server firewall
5. Application listening state
6. tcpdump
```

---

## Scenario 5 — Port refused

```bash
nc -zv 10.10.1.20 8080
```

returns:

```text
Connection refused
```

Usually means:

```text
Host reachable
     ↓
TCP reached host
     ↓
Port is not accepting connections
```

Check:

```bash
ss -lntp | grep :8080
```

Then investigate the application.

---

# 22. Most Important Commands to Remember

For an L3 interview, remember these:

```bash
ip addr
ip link
ip route
ip route get <IP>
ip neigh
ping
traceroute
tracepath
ss -lntp
ss -ant
nc -zv <IP> <PORT>
curl -v
dig
getent hosts
cat /etc/resolv.conf
cat /etc/hosts
firewall-cmd --list-all
systemctl status firewalld
nmcli device status
ethtool
tcpdump
```

# 23. Golden L3 Troubleshooting Rule

When somebody says:

> **"Server A cannot connect to Server B."**

Don't randomly restart services.

Think layer by layer:

```text
Layer 1
Physical / NIC
     ↓
Layer 2
VLAN / MAC / ARP
     ↓
Layer 3
IP / Subnet / Routing
     ↓
Layer 4
TCP / UDP / Port / Firewall
     ↓
Layer 7
DNS / TLS / HTTP / Application
```

**Interview answer:**

> "I troubleshoot host connectivity layer by layer. First I verify the interface and IP configuration, then routing and ARP, followed by gateway and remote-host connectivity. If IP connectivity works, I check DNS and the required TCP/UDP port using `nc`, `ss`, or `curl`. Then I check host firewalls and application listeners. If the issue is still unclear, I use `tcpdump` to determine whether packets are leaving the source and whether responses are returning."
