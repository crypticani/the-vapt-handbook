# 🔴 The VAPT Handbook

> A battle-tested, execution-focused playbook for Vulnerability Assessment & Penetration Testing.
> Built by a DevOps engineer transitioning into offensive security.

---

## 🎯 What This Is

This is **not** a collection of notes. This is a **pentest playbook** — structured like a real engagement, written with an attacker's mindset, and packed with commands, payloads, and workflows you can execute immediately.

Every section follows the kill chain. Every tool has real usage. Every technique has a "why" behind it.

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

---

## 🛠️ Lab Environment Setup

### Required Tools
```bash
# Burp Suite Community Edition
# Download from https://portswigger.net/burp/communitydownload

# Docker (for vulnerable apps)
sudo apt install docker.io docker-compose -y

# OWASP Juice Shop
docker run -d -p 3000:3000 bkimminich/juice-shop

# DVWA
docker run -d -p 80:80 vulnerables/web-dvwa

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
