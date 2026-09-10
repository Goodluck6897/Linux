
# RHEL 8 Package Management

RHEL 8 primarily uses:

* **DNF** → high-level package manager
* **RPM** → low-level package manager
* **RPM packages** → `.rpm`
* **Repositories** → source of packages and dependencies

## Architecture

```text
                 RHEL 8 Package Management
                          |
              +-----------+-----------+
              |                       |
             DNF                     RPM
       High-level tool          Low-level tool
              |                       |
      Dependencies handled       Individual RPM
      Repository based           package operations
```

---

# 1. DNF — Most Important

DNF is the primary package-management tool in RHEL 8.

### Install

```bash
dnf install httpd
```

### Remove

```bash
dnf remove httpd
```

### Update One Package

```bash
dnf update httpd
```

### Update All Packages

```bash
dnf update
```

### Search for a Package

```bash
dnf search httpd
```

### Package Information

```bash
dnf info httpd
```

### List Installed Packages

```bash
dnf list installed
```

### Check a Particular Package

```bash
dnf list installed httpd
```

---
```
1. Show what httpd needs to run (Dependencies)
   #dnf repoquery --requires httpd
2. Show what else on your server depends on httpd (Reverse Dependencies)If httpd is already installed and you want to know what other packages
   rely on it (meaning if you delete httpd, these apps will break),
   #dnf repoquery --whatrequires httpd
3. Check what would be installed (Simulated Install)If you have not installed httpd yet but want to see a neat,
   human-readable summary of every package dependency that dnf will download alongside it, run a simulated installation:
   #dnf install httpd --assumeno
   What it does: The --assumeno flag forces dnf to build the transaction map, show you a neat tabular summary of all package dependencies,
   and automatically abort without changing anything on your system.
```
# 2. RPM Commands

RPM works directly with individual RPM packages and installed-package metadata.

### List All Installed Packages

```bash
rpm -qa
```

Search:

```bash
rpm -qa | grep httpd
```

### Check if a Package Is Installed

```bash
rpm -q httpd
```

### Package Information

```bash
rpm -qi httpd
```

### List Files Installed by a Package

```bash
rpm -ql httpd
```

### List Configuration Files

```bash
rpm -qc httpd
```

### List Documentation

```bash
rpm -qd httpd
```

---

# 3. Find Which Package Owns a File

Very common interview question.

Suppose:

```text
/usr/bin/curl
```

Run:

```bash
rpm -qf /usr/bin/curl
```

This answers:

> Which installed RPM package owns this file?

---

# 4. Find Which Package Provides a File

This is different from `rpm -qf`.

If the command/file is **not installed**, use DNF to search the configured repositories.

```bash
dnf provides /usr/bin/nmap
```

Or:

```bash
dnf provides '*/nmap'
```

### Remember

```text
rpm -qf <file>
        |
        +-- Which installed package owns this file?

dnf provides <file>
        |
        +-- Which available package provides this file?
```

---

# 5. Install a Local RPM

Suppose you downloaded:

```text
myapp-1.0.rpm
```

### Recommended

```bash
dnf install ./myapp-1.0.rpm
```

DNF can resolve dependencies using configured repositories.

### Using RPM

```bash
rpm -ivh myapp-1.0.rpm
```

RPM does not automatically resolve dependencies through repositories.

### Interview Difference

| DNF                        | RPM                                         |
| -------------------------- | ------------------------------------------- |
| High-level package manager | Low-level package manager                   |
| Repository-based           | Works directly with RPM packages            |
| Resolves dependencies      | Does not automatically resolve dependencies |
| Can install/update/remove  | Install/update/remove/query/verify          |

---

# 6. RPM Install / Upgrade / Remove

### Install

```bash
rpm -ivh package.rpm
```

Options:

```text
-i = install
-v = verbose
-h = hash/progress
```

### Upgrade

```bash
rpm -Uvh package.rpm
```

### Remove

```bash
rpm -e package
```

```text
-e = erase
```

---

# 7. Repository Management

Repository configuration files are normally located in:

```bash
/etc/yum.repos.d/
```

Check:

```bash
ls -l /etc/yum.repos.d/
```

### List Enabled Repositories

```bash
dnf repolist
```

### List All Repositories

```bash
dnf repolist all
```

---

# 8. RHEL 8 Repositories

The two major RHEL 8 repositories are:

```text
BaseOS
AppStream
```

## BaseOS

Contains core operating-system packages.

Examples:

* Core utilities
* Libraries
* Basic system functionality
* Kernel-related components

## AppStream

Contains additional applications, runtimes, and user-space components.

Examples:

```text
Python
PHP
Node.js
Databases
Development tools
```

### Interview Point

```text
BaseOS
   |
   +-- Core OS packages

AppStream
   |
   +-- Applications
   +-- Runtime environments
   +-- Developer tools
```

---

# 9. Enable / Disable Repository

On a subscribed RHEL system, `subscription-manager` is commonly used.

### Enable

```bash
subscription-manager repos --enable=<repository>
```

### Disable

```bash
subscription-manager repos --disable=<repository>
```

The exact repository ID depends on your RHEL subscription and release configuration.

---

# 10. DNF Cache

If repository metadata is causing problems:

```bash
dnf clean all
```

Then:

```bash
dnf makecache
```

### Common Troubleshooting Sequence

```bash
dnf clean all
dnf makecache
dnf repolist
```

Then retry:

```bash
dnf install <package>
```

---

# 11. DNF Transaction History

Very important for L3 troubleshooting.

DNF keeps a history of package transactions.

```bash
dnf history
```

Example:

```text
ID | Command
-------------------------
20 | install httpd
19 | update
18 | install vim
```

### Get Transaction Details

```bash
dnf history info 20
```

### Undo a Transaction

```bash
dnf history undo 20
```

> ⚠️ Never blindly undo a production transaction. First review which packages will be changed and the potential impact.

---

# 12. Verify Package Integrity

Useful for troubleshooting modified or corrupted package files.

```bash
rpm -V httpd
```

RPM compares installed files against the package's recorded metadata.

For example, if someone manually modifies a configuration file, verification can report the difference.

---

# 13. Check Package Version

```bash
rpm -q httpd
```

More information:

```bash
rpm -qi httpd
```

Or:

```bash
dnf info httpd
```

---

# 14. Find Available Package Versions

```bash
dnf --showduplicates list httpd
```

Useful when you need to determine whether an older or newer version is available.

---

# 15. Dependency Troubleshooting

Suppose:

```bash
dnf install myapp
```

returns:

```text
Problem: package myapp requires libXYZ
```

### L3 Troubleshooting Approach

```text
dnf install myapp
        |
        v
Dependency error
        |
        v
dnf repolist
        |
        v
Are repositories available?
        |
        v
dnf provides '*/libXYZ'
        |
        v
Which package provides dependency?
        |
        v
Check repository/package availability
        |
        v
dnf clean all
dnf makecache
        |
        v
Retry installation
```

Useful commands:

```bash
dnf repolist
dnf provides '*/libXYZ'
dnf clean all
dnf makecache
```

---

# 16. Real-Time L3 Interview Scenario

### Interviewer:

> "I installed a package yesterday and today the application is not working. What will you check?"

### Good Answer

> "First, I would check the DNF transaction history to determine whether the package or one of its dependencies was recently installed or updated."

```bash
dnf history
```

Then:

```bash
dnf history info <ID>
```

Check the package:

```bash
rpm -qi <package>
```

Check dependencies:

```bash
dnf repoquery --requires <package>
```

Then check the application:

```bash
systemctl status <service>
journalctl -u <service>
```

If the update is confirmed as the cause, I would consider a controlled rollback or downgrade after checking the impact.

---

# 17. RHEL 8 Commands to Memorize

## DNF

```bash
dnf install <package>
dnf remove <package>
dnf update <package>
dnf update
dnf search <package>
dnf info <package>
dnf list installed
dnf provides <file>
dnf repolist
dnf repolist all
dnf clean all
dnf makecache
dnf history
dnf history info <ID>
dnf history undo <ID>
```

## RPM

```bash
rpm -qa
rpm -q <package>
rpm -qi <package>
rpm -ql <package>
rpm -qc <package>
rpm -qd <package>
rpm -qf <file>
rpm -V <package>
rpm -ivh <package.rpm>
rpm -Uvh <package.rpm>
rpm -e <package>
```

---

# ⭐ RHEL 8 L3 Must-Know

If you have limited preparation time, concentrate on these areas:

1. **DNF install / remove / update**
2. **RPM vs DNF**
3. **Package dependencies**
4. **Repository configuration**
5. **`dnf repolist`**
6. **`dnf history`**
7. **`rpm -qf` vs `dnf provides`**
8. **Package installation troubleshooting**
9. **`rpm -V` package verification**
10. **BaseOS vs AppStream**

## 🧠 Quick Interview Cheat Sheet

```text
Install package
    |
    +-- dnf install httpd

Remove package
    |
    +-- dnf remove httpd

Update package
    |
    +-- dnf update httpd

Is package installed?
    |
    +-- rpm -q httpd

Package information
    |
    +-- rpm -qi httpd

Files installed by package
    |
    +-- rpm -ql httpd

Which installed package owns this file?
    |
    +-- rpm -qf /path/to/file

Which available package provides this file?
    |
    +-- dnf provides /path/to/file

Repositories
    |
    +-- dnf repolist

Package transaction history
    |
    +-- dnf history

Verify package
    |
    +-- rpm -V httpd

Local RPM
    |
    +-- dnf install ./package.rpm
```

## 🎯 One-Line Interview Answers

**DNF vs RPM?**

> DNF is a high-level package manager that handles repositories and dependencies, while RPM works directly with individual RPM packages and package metadata.

**How do you find which package owns a file?**

```bash
rpm -qf /path/to/file
```

**How do you find which available package provides a file?**

```bash
dnf provides /path/to/file
```

**Where are repository files stored?**

```text
/etc/yum.repos.d/
```

**How do you check enabled repositories?**

```bash
dnf repolist
```

**How do you check package transaction history?**

```bash
dnf history
```

**How do you verify an installed package?**

```bash
rpm -V <package>
```

**How do you install a local RPM while allowing dependency resolution?**

```bash
dnf install ./package.rpm
```

You can paste this directly into a GitHub `README.md`. If you're building a **complete RHEL 8 L3 interview repository**, the next notes should logically be **systemd → permissions → filesystems/LVM → logs/journalctl → networking → DNS/NTP → performance troubleshooting → SELinux**.
