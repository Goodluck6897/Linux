https://kodekloud.com/blog/linux-interview-questions/


| Old knowledge            | Modern RHEL                        |
| ------------------------ | ---------------------------------- |
| `service`                | `systemctl`                        |
| `chkconfig`              | `systemctl enable/disable`         |
| `yum`                    | `dnf` / `yum` compatibility        |
| `ntpd`                   | `chronyd`                          |
| iptables                 | firewalld/nftables                 |
| ifconfig                 | `ip`                               |
| netstat                  | `ss`                               |
| `route`                  | `ip route`                         |
| `rhui`/traditional repos | subscription/repository management |
| Python 2                 | Python 3                           |
| legacy network scripts   | NetworkManager/nmcli               |
| cgroups v1               | cgroups v2                         |
| older boot tooling       | GRUB2/systemd                      |
| SysV init                | systemd                            |


[root@rhel ~]# chronyc tracking
Reference ID    : 00000000 ()
Stratum         : 0
Ref time (UTC)  : Thu Jan 01 00:00:00 1970
System time     : 0.000000000 seconds fast of NTP time
Last offset     : +0.000000000 seconds
RMS offset      : 0.000000000 seconds
Frequency       : 0.000 ppm slow
Residual freq   : +0.000 ppm
Skew            : 0.000 ppm
Root delay      : 1.000000000 seconds
Root dispersion : 1.000000000 seconds
Update interval : 0.0 seconds
Leap status     : Not synchronised
[root@rhel ~]# 

Common Health Indicators

**Healthy Sync:** Leap status: Normal, System time within milliseconds, and Reference ID set to a valid host.
**Unsynchronized:** Reference ID: 00000000 () or Leap status: Not synchronized indicates chronyd is unreachable, rejected all sources, or still accumulating baseline metrics.

redirections
####
| Command | What It Does |
|---|---|
| `command > file` | Redirect stdout to file |
| `command 2> file` | Redirect stderr to file |
| `command 2>/dev/null` | Discard all errors |
| `command &> file` | Redirect both stdout + stderr to file |
| `command > file 2>&1` | Same as above (older syntax) |.  XXXXX
| `command >> file` | Append stdout to file |
| `command 2>> file` | Append stderr to file |
| `command > out.txt 2> err.txt` | Separate stdout and stderr into different files |


FIND Command
####
#find / -name "Foo.txt" 2>/dev/null
#find / -iname "*foo*txt" 2>/dev/null

| Option | Meaning | Case Sensitive? |
|---|---|---|
| `-name` | Match filename pattern | ✅ **Yes** (case-sensitive) |
| `-iname` | Match filename pattern | ❌ **No** (case-**i**nsensitive) |

3. Find everything
The ls -R command lists the contents of a directory recursively, meaning that it doesn't just list the target you provide for it, but also descends into every subdirectory within that target (and every subdirectory in each subdirectory, and so on.) The find command has that function too, by way of the -ls option:
#find ~/Documents -ls

4. Find by content
A find command doesn't have to perform just one task. In fact, one of the options in find enables you to execute a different command on whatever results find returns. This can be especially useful when you need to search for a file by content rather than by name, or you need to search by both.

#find ~/Documents/ -name "*txt" -exec grep -Hi penguin {} \;
/home/seth/Documents/Foo.txt:I like penguins.
/home/seth/Documents/foo.txt:Penguins are fun.
{} — gets substituted with each matched file path.
\; — signals the end of the -exec expression.

root@rhel ~]# find /root/ -name "*txt" -exec ls -l {} \;
-rw-r--r--. 1 root root 0 Sep  3 18:56 /root/foo.txt
-rw-r--r--. 1 root root 60 Sep  3 19:05 /root/errors.txt
[root@rhel ~]# 

root@rhel ~]# find /root/ -name "*txt" -exec ls -l {} \;
-rw-r--r--. 1 root root 0 Sep  3 18:56 /root/foo.txt
-rw-r--r--. 1 root root 60 Sep  3 19:05 /root/errors.txt
[root@rhel ~]# 
#find ~/Public/ -maxdepth 1 -type d

SUID (Set User ID)
SUID is a special permission that allows a user to execute a file with the permissions of the file's owner, rather than the user running it.

Numeric value: 4000
Symbol: s in the owner's execute position
Common example: /usr/bin/passwd — regular users can change their own password because passwd runs as root (the file owner)

# Using symbolic mode
chmod u+s filename

# Using numeric mode
chmod 4755 filename

How to identify:
ls -l /usr/bin/passwd
# Output: -rwsr-xr-x 1 root root ... /usr/bin/passwd
#            ^ 's' here means SUID is set
Security Warning: SUID on shell scripts or untrusted binaries is a major security risk. An attacker could exploit it to gain root access.

--
SGID (Set Group ID)
SGID works similarly to SUID but applies to the group level. It behaves differently on files vs. directories.

Numeric value: 2000
Symbol: s in the group's execute position
On a file: The file executes with the group permissions of the file's group owner.

On a directory (more common): Any new files/subdirectories created inside will inherit the group of the parent directory, not the creating user's primary group. This is very useful for shared project folders.

How to set SGID:

# On a file
chmod g+s filename

# On a directory
chmod 2775 /shared/project

# Numeric mode
chmod 2755 filename

How to set SGID:
# On a file
chmod g+s filename

# On a directory
chmod 2775 /shared/project

# Numeric mode
chmod 2755 filename

---
Sticky Bit
The Sticky Bit is a permission set on directories that ensures only the file owner, directory owner, or root can delete or rename files within it — even if others have write permission.

Numeric value: 1000
Symbol: t in the others' execute position
Classic example: /tmp directory
***How to set Sticky Bit:**

# Symbolic mode
chmod +t /shared/directory

# Numeric mode
chmod 1777 /shared/directory

Why it matters: Without the sticky bit on /tmp, any user could delete any other user's temporary files since /tmp is world-writable.
----

ACL (Access Control Lists)
ACLs provide fine-grained permissions beyond the traditional owner/group/others model. They let you grant specific permissions to individual users or groups.

Prerequisites:

The filesystem must be mounted with acl option (most modern distros enable this by default)
Packages: acl (provides getfacl and setfacl)

| Command | Purpose |
|---------|---------|
| `getfacl` | View ACLs on a file/directory |
| `setfacl` | Set or modify ACLs |

# Grant read+write to a specific user
setfacl -m u:john:rw filename

# Grant read to a specific group
setfacl -m g:developers:r filename

# Set default ACL on a directory (inherited by new files)
setfacl -d -m u:john:rwx /shared/project

# Remove ACL for a specific user
setfacl -x u:john filename

# Remove ALL ACLs
setfacl -b filename

######**SELinux**


How SELinux Labels Work (The Secret Sauce)SELinux makes decisions based on labels attached to every single file, process, and user on the system. These labels follow a format: **user:role:type:level**.The most important part of this label is the Type (called Type Enforcement).If you run ls -Z (the -Z flag is how you view SELinux contexts), you will see something like this:

**ls -Z /var/www/html/index.html
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
**

Look at httpd_sys_content_t. This label tells SELinux: "This file belongs to the Apache Web Server." If the Apache process (which runs under the type httpd_t) tries to read this file, SELinux says "Yes." If a malicious hacker hijacks a different process and tries to read it, SELinux says "No."

getenforce
# Output: Enforcing, Permissive, or Disabled



| Output | Meaning | Security Status |
| :--- | :--- | :--- |
| **`Enforcing`** | SELinux is active and aggressively guarding your system. Any action that violates your security policies is blocked immediately and logged. | 🔒 Fully Protected (Default & Recommended) |
| **`Permissive`** | SELinux is active but won't block anything. It lets every action slide, but it logs a warning (an AVC denial) whenever a rule is broken. This is mostly used for debugging and troubleshooting apps. | ⚠️ Auditing Only (No actual blocking) |
| **`Disabled`** | SELinux is completely turned off. The security policies are not loaded, and the system is not actively labeling files. | ❌ Unprotected |


The primary log file for SELinux is /var/log/audit/audit.log

[root@rhel ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
[root@rhel ~]# 


Permanent configuration file:

#cat /etc/selinux/configKey Concepts
1. Security Context (Labels)
Every file, process, and port has a security label with four parts:
user:role:type:level
Example:


[root@rhel ~]# ls -Z /etc/passwd
system_u:object_r:passwd_file_t:s0 /etc/passwd
[root@rhel ~]# 


[root@rhel ~]# ls -Z /etc/ssh/
     system_u:object_r:etc_t:s0 moduli                  system_u:object_r:sshd_key_t:s0 ssh_host_ed25519_key.pub
     system_u:object_r:etc_t:s0 ssh_config              system_u:object_r:sshd_key_t:s0 ssh_host_rsa_key
     system_u:object_r:etc_t:s0 ssh_config.d            system_u:object_r:sshd_key_t:s0 ssh_host_rsa_key.pub
system_u:object_r:sshd_key_t:s0 ssh_host_ecdsa_key           system_u:object_r:etc_t:s0 sshd_config
system_u:object_r:sshd_key_t:s0 ssh_host_ecdsa_key.pub       system_u:object_r:etc_t:s0 sshd_config.d
system_u:object_r:sshd_key_t:s0 ssh_host_ed25519_key
[root@rhel ~]# 


root@rhel ~]# ps -eZ | head
LABEL                               PID TTY          TIME CMD
system_u:system_r:init_t:s0           1 ?        00:00:18 systemd
system_u:system_r:kernel_t:s0         2 ?        00:00:00 kthreadd
system_u:system_r:kernel_t:s0         3 ?        00:00:00 pool_workqueue_release
system_u:system_r:kernel_t:s0         4 ?        00:00:00 kworker/R-rcu_gp
system_u:system_r:kernel_t:s0         5 ?        00:00:00 kworker/R-sync_wq


mkdir /testselinux
[root@rhel ~]# touch /testselinux/file
[root@rhel ~]# sudo chcon -t httpd_sys_content_t /testselinux^C
[root@rhel ~]# ls -l /testselinux/
total 0
-rw-r--r--. 1 root root 0 Sep  4 18:52 file
[root@rhel ~]# ls -ld /testselinux/
drwxr-xr-x. 2 root root 18 Sep  4 18:52 /testselinux/

[root@rhel ~]# sudo chcon -t httpd_sys_content_t /testselinux

[root@rhel ~]# ls -Zd /testselinux
unconfined_u:object_r:httpd_sys_content_t:s0 /testselinux



In SELinux, files and processes use different types of labels:

* Files and Directories get Object Labels (like httpd_sys_content_t).
* Running Processes get Domain Labels (like httpd_t).

Because they are different, you have to use different commands to see them.
## 1. How to see the Process Label (httpd_t)
To see the SELinux label of the actual running Apache web server process, use the -Z flag with the ps command:

ps -eZ | grep httpd


* Expected Output: You will see lines starting with system_u:system_r:httpd_t:s0, proving the Apache process runs inside the httpd_t domain.

## 2. How SELinux connects the two
SELinux sits in the background with a massive checklist (the policy). When the Apache process tries to look inside your directory, SELinux checks its list for a rule that matches both labels:

📜 SELinux Policy Rule: Allow a process of type httpd_t to read a directory of type httpd_sys_content_t.

Because that rule exists in RHEL, the access succeeds. If another process running as a different domain (like ftpd_t for an FTP server) tries to access it, SELinux checks the list, finds no matching allow rule, and blocks it.
------------------------------







## The Real-World Use Case
Imagine you change your Apache web server settings so it loads website files from a custom folder like /website instead of the default /var/www/html.
By default, SELinux will label your new /website folder as generic files (default_t), and Apache will be blocked from reading them, causing a 403 Forbidden Error.
Here is how you fix that permanently using semanage fcontext and restorecon:
## Step 1: Tell SELinux the new permanent policy rule
You use semanage fcontext to write a new rule into the SELinux policy database.

sudo semanage fcontext -a -t httpd_sys_content_t "/website(/.*)?"


* -a: Means add a new rule.
* -t httpd_sys_content_t: Specifies the web server type label.
* "/website(/.*)?": This is a regular expression. It tells SELinux to apply this rule to the /website directory and every file or subfolder inside it.

## Step 2: Apply the new rule to the actual files
Running semanage only updates the central database; it doesn't change the files on your disk yet. To force the files to read the new database rule, you run restorecon:

sudo restorecon -R -v /website


* -R: Means recursive (apply to all subfolders and files).
* -v: Means verbose (show you the changes on the screen).

Now, even if the server reboots or the file system is completely relabeled, SELinux will always remember that /website belongs to Apache.
------------------------------
Would you like to see how to use semanage for other things, like changing a network port (e.g., making SSH run on port 2222 instead of 22), or would you prefer to see how to delete a rule you no longer need?


Special Variables
----
#!/bin/bash

echo "Script name : $0"
echo "First arg   : $1"
echo "Second arg  : $2"
echo "Total args  : $#"
echo "All args    : $@"
echo "Process ID  : $$"


$ ./myscript.sh hello world
Script name : ./myscript.sh
First arg   : hello
Second arg  : world
Total args  : 2
All args    : hello world
Process ID  : 5678




