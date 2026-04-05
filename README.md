# 🔴 The VAPT Handbook

> A battle-tested, execution-focused playbook for Vulnerability Assessment & Penetration Testing.
> Built by a DevOps engineer transitioning into offensive security.

---

## 🎯 What This Is

This is **not** a collection of notes. This is a **pentest playbook** — structured like a real engagement, written with an attacker's mindset, and packed with commands, payloads, and workflows you can execute immediately.

Every section follows the kill chain. Every tool has real usage. Every technique has a "why" behind it.

---

## 🚀 Quick Start (Get Hacking in 5 Minutes)

> 📋 **What You Will Do**
> - [ ] Verify prerequisites (Docker, Python, curl)
> - [ ] Launch Juice Shop and DVWA (your practice targets)
> - [ ] Confirm both targets are running and accessible
> - [ ] Install Burp Suite (your primary interception tool)

### Step 1: Verify Prerequisites

```bash
# Check Docker is installed and running
docker --version && docker info > /dev/null 2>&1 && echo "✅ Docker OK" || echo "❌ Docker not running"

# Check Python3
python3 --version && echo "✅ Python3 OK" || echo "❌ Python3 not found"

# Check curl
curl --version > /dev/null 2>&1 && echo "✅ curl OK" || echo "❌ curl not found"
```

> ✅ **Expected Output**
> ```
> Docker version 24.x.x (or newer), build xxxxxxx
> ✅ Docker OK
> Python 3.10.x (or newer)
> ✅ Python3 OK
> ✅ curl OK
> ```

> 🔧 **If Stuck**
> - `❌ Docker not running` → Start Docker: `sudo systemctl start docker` then `sudo systemctl enable docker`
> - `Permission denied` on Docker → Add yourself to docker group: `sudo usermod -aG docker $USER` then log out and log back in
> - `❌ Python3 not found` → Install: `sudo apt install python3 python3-pip -y`
> - On **Mac**: Install Docker Desktop from https://docker.com, Python via `brew install python3`

### Step 2: Launch Your Lab Targets

```bash
# Launch OWASP Juice Shop (intentionally vulnerable e-commerce app)
docker run -d -p 3000:3000 --name juiceshop bkimminich/juice-shop

# Launch DVWA (Damn Vulnerable Web Application)
docker run -d -p 80:80 --name dvwa vulnerables/web-dvwa
```

> ✅ **Expected Output**
> ```
> Unable to find image 'bkimminich/juice-shop:latest' locally    ← First time only
> latest: Pulling from bkimminich/juice-shop
> ...
> Status: Downloaded newer image for bkimminich/juice-shop:latest
> a1b2c3d4e5f6...    ← Container ID (yours will differ)
> ```

### Step 3: Verify Your Labs Are Running

```bash
# Verify Juice Shop
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000 && echo " ← Juice Shop"

# Verify DVWA
curl -s -o /dev/null -w "%{http_code}" http://localhost:80 && echo " ← DVWA"

# Check both containers are up
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

> ✅ **Expected Output**
> ```
> 200 ← Juice Shop
> 302 ← DVWA (redirects to setup page on first run)
> NAMES        STATUS          PORTS
> dvwa         Up 30 seconds   0.0.0.0:80->80/tcp
> juiceshop    Up 45 seconds   0.0.0.0:3000->3000/tcp
> ```

> 🔧 **If Stuck**
> - `Connection refused` → Container may not be ready yet. Wait 30 seconds and retry.
> - `Port already in use` → Another service is on that port. Stop it: `sudo lsof -i :3000` to find PID, then `kill PID`. Or use a different port: `docker run -d -p 3001:3000 --name juiceshop bkimminich/juice-shop`
> - Container keeps restarting → Check logs: `docker logs juiceshop` or `docker logs dvwa`
> - `Cannot connect to Docker daemon` → Docker isn't running: `sudo systemctl start docker`

### Step 4: Open In Browser & Set Up DVWA

```
1. Open http://localhost:3000 → You should see the Juice Shop storefront
2. Open http://localhost:80   → You should see the DVWA login page
3. DVWA first-time setup:
   a. Login with admin / password
   b. Click "Create / Reset Database" at the bottom
   c. Login again with admin / password
   d. Go to "DVWA Security" → Set to "Low" (we'll increase as you progress)
```

> ✅ **Expected Output**
> - **Juice Shop**: A colorful e-commerce page with products like "Apple Juice" and a search bar
> - **DVWA**: A red-themed dashboard with vulnerability categories listed on the left sidebar

> 💡 **Why This Matters**
> Juice Shop and DVWA are your **safe, legal** practice environments. Every technique in this handbook will be demonstrated against one of these targets. You'll never have to wonder "where do I run this?" — the answer is always one of these two.

### Step 5: Install Remaining Tools

```bash
# Download Burp Suite Community Edition
# Go to: https://portswigger.net/burp/communitydownload
# Download the Linux (.sh) installer, then:
chmod +x burpsuite_community_linux*.sh
./burpsuite_community_linux*.sh
# Follow the installer prompts (Next → Next → Finish)

# Install command-line tools (if not on Kali Linux)
sudo apt install nmap gobuster nikto sqlmap whatweb wfuzz ffuf -y

# Install Python libraries
pip3 install requests beautifulsoup4 pwntools
```

> ✅ **Expected Output (verify tools work)**
> ```bash
> nmap --version    # Nmap version 7.92+
> ffuf -V           # ffuf version 2.x.x
> sqlmap --version  # sqlmap/1.7+
> ```

> 🔧 **If Stuck**
> - `E: Unable to locate package ffuf` → Install from Go: `go install github.com/ffuf/ffuf/v2@latest`
> - Burp Suite won't start → Ensure Java is installed: `sudo apt install default-jre -y`
> - `pip3 install` fails → Try: `pip3 install --user requests beautifulsoup4 pwntools`

### ✅ Quick Start Complete!

You're ready. Proceed to `01-fundamentals/` to begin.

**Stopping your labs when you're done for the day:**
```bash
docker stop juiceshop dvwa
```

**Restarting next time:**
```bash
docker start juiceshop dvwa
```

---

## 📁 Structure

```
the-vapt-handbook/
├── 01-fundamentals/        # HTTP internals, OWASP, input flow tracing
├── 02-recon/               # Passive/active recon, attack surface mapping
├── 03-web-exploitation/    # SQLi, XSS, SSRF, auth flaws, file uploads
├── 04-privilege-escalation/# Linux privesc — SUID, cron, PATH, kernel
├── 05-post-exploitation/   # Post-shell ops, creds, persistence, pivoting
├── 06-cloud-security/      # AWS, Kubernetes, CI/CD attack vectors
├── 07-ctf-writeups/        # Writeup templates, methodology documentation
├── 08-bug-bounty/          # Recon pipelines, automation, report writing
```

Each directory contains:

| File | Purpose |
|------|---------|
| `notes.md` | Core concepts + attacker thinking |
| `labs.md` | Hands-on exercises (Juice Shop, DVWA, local labs) |
| `tools.md` | Tools with real usage (commands, flags, examples) |
| `payloads.md` | Attack payloads (where applicable) |
| `methodology.md` | Step-by-step operational approach |

---

## 🧠 How To Use This

1. **Follow the order**: `01` → `08`. Each section builds on the previous.
2. **Execute everything**: Don't just read — run every command, intercept every request, break every lab.
3. **Build muscle memory**: Repetition with tools like Burp Suite, Nmap, and ffuf is how you get fast.
4. **Think like an attacker**: Every section has "Attacker Thinking" blocks. Internalize them.
5. **Use during engagements**: This is designed to be referenced mid-pentest.
6. **Check "Expected Output"**: After every command, compare your output to the expected output box. If they don't match, use the "If Stuck" block.
7. **Don't skip labs**: Reading without doing is useless. Each lab builds real skill.

---

## 🛠️ Lab Environment Setup (Detailed)

### Required Tools
```bash
# Burp Suite Community Edition
# Download from https://portswigger.net/burp/communitydownload

# Docker (for vulnerable apps)
sudo apt install docker.io docker-compose -y

# OWASP Juice Shop
docker run -d -p 3000:3000 --name juiceshop bkimminich/juice-shop

# DVWA
docker run -d -p 80:80 --name dvwa vulnerables/web-dvwa

# Kali Linux tools (if not on Kali)
sudo apt install nmap gobuster nikto sqlmap whatweb wfuzz -y

# Python tools
pip3 install requests beautifulsoup4 pwntools
```

### Recommended Platforms
- [HackTheBox](https://www.hackthebox.com/) — Real-world machines
- [TryHackMe](https://tryhackme.com/) — Guided learning paths
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — Web vuln mastery
- [PentesterLab](https://pentesterlab.com/) — Badge-based progression
- [VulnHub](https://www.vulnhub.com/) — Downloadable vulnerable VMs

---

## ⚠️ Legal Disclaimer

Everything in this repository is for **authorized testing and educational purposes only**. Never test systems without explicit written permission. Unauthorized access to computer systems is illegal under laws including the Computer Fraud and Abuse Act (CFAA), IT Act 2000, and equivalent legislation worldwide.

**Always get written authorization before testing.**

---

## 📜 License

MIT — Use it, share it, break things (legally).
