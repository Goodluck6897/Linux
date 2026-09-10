The diagram traces how an application’s request to resolve www.example.com moves from your local system config files out into the public internet. Here are the exact gateway needs within this flow: [1] (https://wiki.archlinux.org/title/Domain_name_resolution), [2] (https://www.freecodecamp.org/news/how-dns-works-the-internets-address-book/)

```text
Application
    |
glibc / NSS
    |
/etc/nsswitch.conf
    |
/etc/resolv.conf
    |
    v
 ====== BOUNDARY 1: Local Link / Internal Network Gateway ======
    |  -> Requires a Default Gateway (Router) to route UDP/TCP Port 53 traffic 
    |     out of the host to an external Local/Recursive DNS Server.
    v
DNS Server (Recursive resolution)
    |
    v
 ====== BOUNDARY 2: Internet Edge / Border Gateway ======
    |  -> Requires a NAT/Border Gateway & Firewall policies to allow the 
    |     Recursive Resolver to traverse the open internet and reach Root, 
    |     TLD, and Authoritative DNS servers worldwide.
    v
Root DNS -> .com TLD -> Authoritative DNS -> IP Address Returned

```
