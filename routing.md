# Linux Routing — L3 System Administrator Interview Guide

For a **Linux L3/System Administrator**, routing is about understanding how Linux decides **where packets should go**, how to troubleshoot connectivity, and how to configure **static/default routes**.

---

## 1. What is routing?

Routing is the process of deciding **which path a network packet should take** from source to destination.

Example:

```text
Linux Server
10.10.10.20
      |
      | eth0
      |
10.10.10.1  ← Default Gateway / Router
      |
      v
   Network
      |
      v
192.168.20.50
```

If the server wants to communicate with `192.168.20.50`, Linux checks its routing table.

---

# 2. Linux Routing Table

The most important command:

```bash
ip route
```

Example:

```text
default via 10.10.10.1 dev eth0
10.10.10.0/24 dev eth0 proto kernel scope link src 10.10.10.20
192.168.20.0/24 via 10.10.10.1 dev eth0
```

Understand each line.

### Default route

```text
default via 10.10.10.1 dev eth0
```

Means:

> If Linux doesn't have a more specific route, send the packet to `10.10.10.1` through `eth0`.

---

### Connected route

```text
10.10.10.0/24 dev eth0 proto kernel scope link src 10.10.10.20
```

Means:

```text
Network: 10.10.10.0/24
Interface: eth0
Source IP: 10.10.10.20
```

Linux automatically creates this route when an IP is configured on the interface.

---

### Static route

```text
192.168.20.0/24 via 10.10.10.1 dev eth0
```

Means:

> To reach `192.168.20.0/24`, use gateway `10.10.10.1` through `eth0`.

---

# 3. Important Routing Commands

### Display routing table

```bash
ip route
```

or:

```bash
ip r
```

---

### Display routes for IPv4

```bash
ip -4 route
```

### Display IPv6 routes

```bash
ip -6 route
```

---

### Find the route Linux will use

This is **very important for interviews**:

```bash
ip route get 8.8.8.8
```

Example:

```text
8.8.8.8 via 10.10.10.1 dev eth0 src 10.10.10.20
```

This tells you:

```text
Destination → 8.8.8.8
Gateway    → 10.10.10.1
Interface  → eth0
Source IP  → 10.10.10.20
```

---

# 4. Default Gateway

The default gateway is used when there is **no more specific route**.

Example:

```text
default via 192.168.1.1 dev eth0
```

Server:

```text
192.168.1.10
```

Gateway:

```text
192.168.1.1
```

Internet:

```text
8.8.8.8
```

Traffic:

```text
Server
192.168.1.10
     |
     v
192.168.1.1
 Default Gateway
     |
     v
 Internet
     |
     v
8.8.8.8
```

---

# 5. Adding a Static Route

Suppose:

```text
Server:     10.10.10.20
Gateway:    10.10.10.1
Destination network: 192.168.20.0/24
```

Add:

```bash
sudo ip route add 192.168.20.0/24 via 10.10.10.1
```

Or specify interface:

```bash
sudo ip route add 192.168.20.0/24 via 10.10.10.1 dev eth0
```

Verify:

```bash
ip route
```

---

# 6. Delete a Route

```bash
sudo ip route del 192.168.20.0/24
```

---

# 7. Add Default Route

```bash
sudo ip route add default via 10.10.10.1
```

Verify:

```bash
ip route
```

You should see:

```text
default via 10.10.10.1 dev eth0
```

---

# 8. Replace a Route

Instead of deleting and adding:

```bash
sudo ip route replace 192.168.20.0/24 via 10.10.10.254
```

`replace` is useful in automation because it works whether the route already exists or not.

---

# 9. Routing Metrics

Suppose you have two default routes:

```text
default via 10.10.10.1 dev eth0 metric 100
default via 10.20.10.1 dev eth1 metric 200
```

Linux generally prefers the route with the **lower metric** when otherwise comparable.

Therefore:

```text
metric 100 → preferred
metric 200 → backup
```

Check:

```bash
ip route
```

---

# 10. Longest Prefix Match

This is a **very important interview concept**.

Suppose routing table contains:

```text
10.0.0.0/8 via 10.10.10.1
10.20.0.0/16 via 10.10.10.2
10.20.30.0/24 via 10.10.10.3
default via 10.10.10.1
```

Destination:

```text
10.20.30.50
```

Which route is selected?

```text
10.20.30.0/24
```

Why?

Because `/24` is the **most specific route**.

The general rule:

```text
Most specific route wins
        ↓
Longest prefix match
        ↓
Then route metric is considered among comparable routes
```

---

# 11. Routing Decision Example

Suppose:

```text
ip route

default via 10.10.10.1 dev eth0
10.10.10.0/24 dev eth0
10.20.0.0/16 via 10.10.10.2 dev eth0
10.20.30.0/24 via 10.10.10.3 dev eth0
```

Destination:

```bash
ping 10.20.30.50
```

Linux checks:

```text
10.20.30.50
      |
      v
10.20.30.0/24 ? YES
      |
      v
Gateway 10.10.10.3
      |
      v
eth0
```

It does **not** use the default gateway because a more specific route exists.

---

# 12. Gateway Must Be Reachable

Consider:

```bash
ip route add 192.168.20.0/24 via 172.16.1.1
```

But server has:

```text
eth0 = 10.10.10.20/24
```

and there is no route to:

```text
172.16.1.1
```

The gateway is not reachable from the server's current routing configuration.

You need to understand:

```text
Server
10.10.10.20
     |
     X
172.16.1.1
```

A gateway normally needs to be reachable through an appropriate connected or existing route.

---

# 13. Routing vs Switching

This is frequently asked.

### Switching

Works primarily within the **same Layer-2 network**.

```text
Server A
10.10.10.10
    |
    v
Switch
    |
    v
Server B
10.10.10.20
```

### Routing

Connects **different IP networks**.

```text
10.10.10.0/24
       |
       v
    Router
       |
       v
192.168.20.0/24
```

Linux can also perform routing if configured as a router.

---

# 14. Linux as a Router

Check whether IP forwarding is enabled:

```bash
sysctl net.ipv4.ip_forward
```

Output:

```text
net.ipv4.ip_forward = 0
```

Disabled.

Enable temporarily:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

For persistent configuration, configure it through `/etc/sysctl.conf` or an appropriate file under `/etc/sysctl.d/`.

---

# 15. Routing Troubleshooting — L3 Interview Scenario

### Scenario

Application server cannot connect to:

```text
192.168.50.100:443
```

As an L3 administrator, don't immediately assume the application is broken.

Follow a structured approach:

```text
Application
    |
    v
DNS?
    |
    v
IP connectivity?
    |
    v
Routing?
    |
    v
Firewall?
    |
    v
Port?
    |
    v
Remote server?
```

---

## Step 1 — Check IP configuration

```bash
ip addr
```

Check:

```text
IP address
Subnet mask/prefix
Interface status
```

---

## Step 2 — Check routing

```bash
ip route
```

Look for a route toward:

```text
192.168.50.0/24
```

---

## Step 3 — Ask Linux which route it will use

```bash
ip route get 192.168.50.100
```

Example:

```text
192.168.50.100 via 10.10.10.1 dev eth0 src 10.10.10.20
```

This is one of the **best troubleshooting commands**.

---

## Step 4 — Test gateway

```bash
ping -c 4 10.10.10.1
```

If gateway fails:

```text
Server
  |
  X
Gateway
```

Investigate:

* Interface
* VLAN
* IP configuration
* ARP
* Switch
* Network connectivity

---

## Step 5 — Check ARP/neighbor table

Modern command:

```bash
ip neigh
```

Example:

```text
10.10.10.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

Useful states:

```text
REACHABLE
STALE
DELAY
PROBE
FAILED
```

---

# 16. Traceroute

To identify where packets stop:

```bash
traceroute 192.168.50.100
```

If not installed on RHEL:

```bash
sudo dnf install traceroute
```

Alternative:

```bash
tracepath 192.168.50.100
```

For TCP-based testing:

```bash
traceroute -T -p 443 192.168.50.100
```

This can be useful when ICMP/UDP traceroute is filtered.

---

# 17. TCP Connectivity Test

Don't rely only on `ping`.

For an application using HTTPS:

```bash
nc -vz 192.168.50.100 443
```

or:

```bash
curl -v https://192.168.50.100
```

You can determine whether the problem is:

```text
Routing
   ↓
Firewall
   ↓
TCP port
   ↓
Application
```

---

# 18. Packet Capture

When routing is suspicious:

```bash
sudo tcpdump -i eth0 host 192.168.50.100
```

For HTTPS:

```bash
sudo tcpdump -i eth0 host 192.168.50.100 and port 443
```

You can determine:

```text
Did packet leave the server?
Did response return?
Is there retransmission?
Is TCP handshake completing?
```

For example:

```text
Client                    Server
  |                         |
  | ---- SYN ------------> |
  | <--- SYN/ACK ---------- |
  | ---- ACK ------------> |
```

If you see SYN leaving but no SYN/ACK returning, investigate the network path, firewall, routing, or remote side.

---

# 19. Policy Routing

Most Linux administrators first work with the **main routing table**.

But Linux also supports policy routing.

Check:

```bash
ip rule
```

Example:

```text
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

You can have different routing decisions based on:

* Source IP
* Destination
* Firewall mark
* Other policy conditions

This becomes important on:

```text
Multi-homed servers
VPN gateways
Network appliances
Complex enterprise networks
```

---

# 20. Multiple Network Interfaces

Example:

```text
Server
   |
   +--- eth0 → 10.10.10.20
   |
   +--- eth1 → 192.168.10.20
```

Potential problems:

* Wrong default gateway
* Asymmetric routing
* Incorrect source IP
* Reverse path filtering
* Policy routing issues

Check:

```bash
ip addr
ip route
ip rule
ip route get <destination>
```

---

# 21. Asymmetric Routing

Example:

```text
Client
  |
  v
Router A
  |
  v
Server
  |
  v
Router B
  |
  v
Client
```

Request and response use different paths.

This can cause problems with:

* Firewalls
* Stateful load balancers
* Reverse path filtering
* Applications expecting symmetric paths

When troubleshooting, capture traffic on both interfaces:

```bash
tcpdump -i any host <IP>
```

---

# 22. RHEL 8 — Persistent Routes

For RHEL 8, networking is normally managed using **NetworkManager**.

Use:

```bash
nmcli
```

List connections:

```bash
nmcli connection show
```

Add a persistent route:

```bash
nmcli connection modify eth0 +ipv4.routes "192.168.20.0/24 10.10.10.1"
```

Bring the connection up:

```bash
nmcli connection up eth0
```

Verify:

```bash
ip route
```

**Interview point:** `ip route add` changes the running kernel routing table but is not, by itself, a persistent configuration.

---

# 23. Important Commands to Memorize

```bash
ip addr
ip link
ip route
ip route get <destination>
ip rule
ip neigh
ping <IP>
traceroute <IP>
tracepath <IP>
ss -tulnp
nc -vz <IP> <PORT>
tcpdump -i eth0
```

For RHEL 8:

```bash
nmcli connection show
nmcli device status
nmcli connection show <connection>
```

---

# 24. L3 Interview Troubleshooting Flow

Memorize this:

```text
Connectivity problem
       |
       v
1. ip addr
       |
       v
2. ip link
       |
       v
3. ip route
       |
       v
4. ip route get <destination>
       |
       v
5. ip neigh
       |
       v
6. ping gateway
       |
       v
7. ping destination
       |
       v
8. traceroute / tracepath
       |
       v
9. nc / curl
       |
       v
10. tcpdump
       |
       v
11. Firewall
       |
       v
12. Remote server/network
```

## ⭐ Interview Questions

### Q1. How do you check the routing table?

```bash
ip route
```

### Q2. How do you determine which route Linux will use?

```bash
ip route get <destination-ip>
```

### Q3. What is the default route?

```text
default via <gateway> dev <interface>
```

It is used when no more-specific route matches the destination.

### Q4. Which route wins when multiple routes match?

**The most specific route — longest prefix match.**

### Q5. How do you add a temporary static route?

```bash
ip route add 192.168.20.0/24 via 10.10.10.1
```

### Q6. How do you remove it?

```bash
ip route del 192.168.20.0/24
```

### Q7. How do you check whether Linux is forwarding packets?

```bash
sysctl net.ipv4.ip_forward
```

### Q8. How do you troubleshoot a server that cannot reach another network?

Start with:

```bash
ip addr
ip route
ip route get <destination>
ip neigh
ping <gateway>
traceroute <destination>
nc -vz <destination> <port>
tcpdump -i <interface>
```

### ⭐ Most important L3 commands

If you remember only five, remember:

```bash
ip addr
ip route
ip route get <destination>
ip neigh
tcpdump -i <interface>
```

These five commands cover a **large portion of real-world Linux network/routing troubleshooting**.
