Great question, Venkatadry! The myhostname module does not check any file. Here's how it works:

How myhostname Works

myhostname is a systemd NSS plugin (nss-myhostname) that resolves hostnames without reading any file. Instead, it works programmatically:

It resolves these specific names by querying the kernel directly:

What It ResolvesHow It Gets the ValueExampleThe system's own hostnameCalls gethostname() from the kernelmyserver.prod.example.com → 127.0.0.1localhostHardcoded in the pluginlocalhost → 127.0.0.1_gatewayReads the default gateway from the routing table_gateway → 10.20.30.1

What It ResolvesHow It Gets the ValueExample

What It Resolves

How It Gets the Value

Example

The system's own hostnameCalls gethostname() from the kernelmyserver.prod.example.com → 127.0.0.1

The system's own hostname

Calls gethostname() from the kernel

myserver.prod.example.com → 127.0.0.1

localhostHardcoded in the pluginlocalhost → 127.0.0.1

localhost

Hardcoded in the plugin

localhost → 127.0.0.1

_gatewayReads the default gateway from the routing table_gateway → 10.20.30.1

_gateway

Reads the default gateway from the routing table

_gateway → 10.20.30.1

Where does the hostname itself come from?

The hostname that myhostname resolves is set by:

# Check current hostname (from kernel memory, not a file):
hostname
hostnamectl

# The hostname is typically configured in:
cat /etc/hostname          # ← Static hostname (read at boot)

So while /etc/hostname sets the hostname at boot time, the myhostname plugin reads it from the kernel at runtime — not from any file.

Quick Proof

# Change hostname in memory (no file change):
sudo hostnamectl set-hostname testbox.local

# myhostname immediately resolves it:
getent hosts testbox.local
# Output: 127.0.0.1  testbox.local

# Even though /etc/hosts has NO entry for testbox.local

Why Does It Exist?

It acts as a safety net / fallback — ensuring the machine can always resolve its own hostname, even if:

- /etc/hosts is empty or misconfigured

- DNS server is unreachable

- /etc/resolv.conf is broken

That's why it's placed last in the chain: hosts: files dns myhostname — it only kicks in if everything else fails.
