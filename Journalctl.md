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
--
journalctl -u sshd
journalctl -u sshd -f
journalctl -u sshd --since today
journalctl -p err
journalctl -b -1
journalctl -k -p err
