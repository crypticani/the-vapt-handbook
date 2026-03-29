# 01 — Fundamentals: Tools & Real Usage

> These are the tools you'll use in every engagement. Learn them deeply — not just the commands, but what each flag does and when to use it.

---

## 🔴 Burp Suite (The Swiss Army Knife)

### Setup & Configuration

```
# Download: https://portswigger.net/burp/communitydownload
# Run:
java -jar burpsuite_community.jar

# Proxy Listener: 127.0.0.1:8080
# Browser Proxy: Configure Firefox/Chrome to use 127.0.0.1:8080
# CA Certificate: Navigate to http://burpsuite → Download → Import into browser
```

### Essential Tabs

| Tab | Purpose | When To Use |
|-----|---------|-------------|
| Proxy | Intercept/modify requests in real-time | Login testing, parameter tampering |
| Repeater | Send modified requests manually | Testing payloads, fuzzing individual params |
| Intruder | Automated payload delivery | Brute force, parameter fuzzing, SQLi |
| Decoder | Encode/decode data | Base64, URL encoding, hex |
| Comparer | Diff two responses | Comparing valid vs invalid responses |
| Target | Site map and scope | Attack surface mapping |

### Proxy Usage

```
# Intercept a request and modify it
1. Proxy → Intercept → Intercept is ON
2. Perform action in browser
3. Request appears in Burp
4. Modify any field → Forward
5. Check response in HTTP History

# Intercept responses (often missed):
Proxy → Options → Intercept Server Responses → Check "Intercept responses based on..."
```

### Repeater Workflows

```
# Testing SQL Injection manually
1. Capture request in Proxy
2. Right-click → Send to Repeater
3. Modify parameter: username=admin'
4. Click Send → Check response
5. Modify parameter: username=admin' OR 1=1-- -
6. Click Send → Compare responses
7. Use Ctrl+R to send same request repeatedly
```

### Intruder Attack Types

```
# Sniper: One payload position, one wordlist
# Use for: Testing one parameter at a time
Positions: username=§admin§&password=test
Payload: wordlist of usernames

# Battering Ram: Same payload in all positions
# Use for: When the same value goes in multiple places
Positions: user=§test§&confirm=§test§

# Pitchfork: Multiple positions, multiple lists (parallel)
# Use for: Credential stuffing (user1:pass1, user2:pass2)
Positions: username=§admin§&password=§password§
Payload 1: usernames.txt
Payload 2: passwords.txt

# Cluster Bomb: All combinations of all lists
# Use for: Brute force (try every password for every user)
# ⚠️ Very slow with large lists
Positions: username=§admin§&password=§password§
```

### Burp Extensions To Install

```
# Extensions → BApp Store → Install:

1. Param Miner          → Discovers hidden parameters
2. Autorize             → IDOR testing automation
3. JWT Editor           → Decode, modify, and forge JWTs
4. Hackvertor           → Advanced encoding/decoding
5. Logger++             → Enhanced HTTP logging
6. Active Scan++        → Better vulnerability detection
7. Retire.js            → Detect vulnerable JavaScript libraries
8. Upload Scanner       → File upload vulnerability testing
```

---

## 🔴 cURL (Command-Line HTTP Client)

### Essential Flags

```bash
# Basic GET request
curl https://target.com

# Include response headers
curl -i https://target.com

# Only show headers
curl -I https://target.com

# Follow redirects
curl -L https://target.com

# POST request with JSON data
curl -X POST https://target.com/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test"}'

# POST with form data
curl -X POST https://target.com/login \
  -d "username=admin&password=test"

# Custom headers
curl -H "Authorization: Bearer TOKEN" \
  -H "X-Forwarded-For: 127.0.0.1" \
  https://target.com/api/admin

# Send cookies
curl -b "session=abc123; admin=true" https://target.com/dashboard

# Save cookies to file, then reuse
curl -c cookies.txt https://target.com/login
curl -b cookies.txt https://target.com/dashboard

# Upload a file
curl -F "file=@payload.php" https://target.com/upload

# Ignore SSL errors (self-signed certs)
curl -k https://target.com

# Verbose output (shows TLS handshake, headers)
curl -v https://target.com

# Show only HTTP status code
curl -s -o /dev/null -w "%{http_code}" https://target.com

# Time the request
curl -s -o /dev/null -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\n" https://target.com

# Through a proxy (Burp)
curl -x http://127.0.0.1:8080 https://target.com
```

### cURL For Pentesters

```bash
# Test for CORS misconfiguration
curl -H "Origin: https://evil.com" -I https://target.com/api/data 2>/dev/null | grep -i "access-control"

# Check for HTTP methods allowed
curl -X OPTIONS -I https://target.com/api/resource

# Test for path traversal
curl https://target.com/download?file=../../../../etc/passwd

# Test for SSRF
curl https://target.com/fetch?url=http://169.254.169.254/latest/meta-data/

# User-Agent SQL injection
curl -H "User-Agent: ' OR 1=1-- -" https://target.com

# Test for verb tampering
for method in GET POST PUT DELETE PATCH OPTIONS HEAD; do
  echo -n "$method: "
  curl -s -o /dev/null -w "%{http_code}" -X $method https://target.com/admin
  echo
done

# Brute force with curl (basic)
while read pass; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
    -d "username=admin&password=$pass" https://target.com/login)
  echo "$pass: $code"
done < /usr/share/wordlists/rockyou.txt
```

---

## 🔴 Developer Tools (Browser)

### Network Tab

```
# Open: F12 → Network tab
# Filter: XHR (to see API calls only)
# Check: "Preserve log" (to keep history across page loads)

# For each request, examine:
# - Headers tab: All request and response headers
# - Payload tab: POST body data
# - Response tab: Raw response body
# - Timing tab: How long each phase took
# - Cookies tab: Cookies sent and received
```

### Console Tab

```javascript
// Check for exposed JavaScript variables
window.__NEXT_DATA__          // Next.js — may contain API keys, user data
window.__NUXT__               // Nuxt.js — server-side data
window.__REDUX_STATE__        // Redux state — full application state

// Read all cookies
document.cookie

// Read localStorage
JSON.stringify(localStorage)

// Read sessionStorage
JSON.stringify(sessionStorage)

// Find all forms and their actions
document.querySelectorAll('form').forEach(f => console.log(f.action, f.method))

// Find all links
document.querySelectorAll('a').forEach(a => console.log(a.href))

// Find all input fields (including hidden)
document.querySelectorAll('input').forEach(i => console.log(i.name, i.type, i.value))

// Find all JavaScript files loaded
performance.getEntriesByType('resource')
  .filter(r => r.name.endsWith('.js'))
  .forEach(r => console.log(r.name))

// Detect JavaScript frameworks
typeof React !== 'undefined'   // React
typeof angular !== 'undefined' // Angular
typeof Vue !== 'undefined'     // Vue
typeof jQuery !== 'undefined'  // jQuery
```

### Sources Tab

```
# Look for:
# - Source maps (.map files) → Full original source code
# - Inline scripts with API keys or secrets
# - WebSocket connections
# - Service Worker files

# Search across all sources: Ctrl+Shift+F
# Search for: "api_key", "secret", "password", "token", "admin"
```

---

## 🔴 HTTPie (Better cURL for APIs)

```bash
# Install
pip3 install httpie

# GET request (colored, formatted output)
http https://target.com/api/users

# POST with JSON (default)
http POST https://target.com/api/login username=admin password=test

# Custom headers
http https://target.com/api/data "Authorization: Bearer TOKEN"

# Form data
http -f POST https://target.com/login username=admin password=test

# File upload
http -f POST https://target.com/upload file@payload.php

# Download response body
http https://target.com/api/data > response.json

# Through proxy
http --proxy=http:http://127.0.0.1:8080 https://target.com
```

---

## 🔴 Python `requests` Library

```python
import requests

# Disable SSL warnings for self-signed certs
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

# Basic GET
r = requests.get('https://target.com', verify=False)
print(r.status_code, r.headers, r.text)

# POST with JSON
r = requests.post('https://target.com/api/login',
    json={'username': 'admin', 'password': 'test'},
    verify=False)

# Session management (maintains cookies across requests)
s = requests.Session()
s.post('https://target.com/login',
    data={'username': 'admin', 'password': 'password'})
# Now authenticated for subsequent requests:
r = s.get('https://target.com/dashboard')

# Custom headers
r = requests.get('https://target.com/api/admin',
    headers={
        'Authorization': 'Bearer TOKEN',
        'X-Forwarded-For': '127.0.0.1'
    })

# Through Burp proxy
proxies = {'http': 'http://127.0.0.1:8080', 'https': 'http://127.0.0.1:8080'}
r = requests.get('https://target.com', proxies=proxies, verify=False)

# File upload
files = {'file': ('shell.php', open('shell.php', 'rb'), 'image/jpeg')}
r = requests.post('https://target.com/upload', files=files)

# Iterate over IDs (IDOR testing)
for i in range(1, 100):
    r = requests.get(f'https://target.com/api/users/{i}',
        headers={'Authorization': 'Bearer USER_TOKEN'})
    if r.status_code == 200:
        print(f"[+] User {i}: {r.json()}")
```

---

## 🔴 Wget (Recursive Downloads)

```bash
# Mirror a website
wget --mirror --convert-links --adjust-extension --page-requisites \
  --no-parent https://target.com -P ./mirror/

# Download specific file types
wget -r -A "*.pdf,*.doc,*.xls,*.bak,*.sql" https://target.com

# Ignore robots.txt
wget -e robots=off -r https://target.com

# Through proxy
wget -e use_proxy=yes -e http_proxy=127.0.0.1:8080 https://target.com
```

---

## 🔴 OpenSSL (TLS/SSL Testing)

```bash
# Connect to a server and show the TLS handshake
openssl s_client -connect target.com:443

# Show certificate details
openssl s_client -connect target.com:443 </dev/null 2>/dev/null | openssl x509 -noout -text

# Show certificate expiry
openssl s_client -connect target.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates

# Show certificate chain
openssl s_client -connect target.com:443 -showcerts

# Test specific TLS version
openssl s_client -connect target.com:443 -tls1_2
openssl s_client -connect target.com:443 -tls1_1  # Should fail if old TLS disabled

# Check for specific cipher
openssl s_client -connect target.com:443 -cipher 'RC4'  # Should fail

# Comprehensive TLS scan
testssl.sh target.com
# Install: git clone https://github.com/drwetter/testssl.sh.git
```

---

## 🔴 HashID & Hash Cracking

```bash
# Identify hash type
hashid 'e10adc3949ba59abbe56e057f20f883e'
# or
hash-identifier

# Common hashes you'll find:
# MD5:    32 hex chars          e10adc3949ba59abbe56e057f20f883e
# SHA1:   40 hex chars          7c4a8d09ca3762af61e59520943dc26494f8941b
# SHA256: 64 hex chars          8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
# bcrypt: $2a$10$...            $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy

# Crack with hashcat
# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
# SHA1
hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt
# SHA256
hashcat -m 1400 hash.txt /usr/share/wordlists/rockyou.txt
# bcrypt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# Crack with John the Ripper
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt  # Show cracked passwords
```

---

## 📋 Tool Installation Checklist

```bash
# Essential tools
sudo apt update && sudo apt install -y \
  nmap \
  nikto \
  sqlmap \
  gobuster \
  dirb \
  wfuzz \
  whatweb \
  curl \
  wget \
  openssl \
  netcat-openbsd \
  python3-pip \
  git \
  jq

# Python tools
pip3 install \
  requests \
  httpie \
  beautifulsoup4 \
  pwntools \
  jwt_tool

# From GitHub
git clone https://github.com/ticarpi/jwt_tool.git
git clone https://github.com/drwetter/testssl.sh.git
git clone https://github.com/danielmiessler/SecLists.git

# Burp Suite
# Download from https://portswigger.net/burp/communitydownload

# Wordlists
sudo apt install -y seclists
# Default location: /usr/share/seclists/
```

---

## 🧠 Attacker Thinking: Tool Selection

| Scenario | Tool | Why |
|----------|------|-----|
| Quick header check | `curl -I` | Fastest, no setup |
| Deep request manipulation | Burp Repeater | Visual, iterative, easy to compare |
| Automated parameter fuzzing | Burp Intruder or `wfuzz` | Scale beyond manual testing |
| API testing | `httpie` or `requests` | Clean output, session management |
| TLS analysis | `testssl.sh` | Comprehensive, scriptable |
| Credential cracking | `hashcat` | GPU-accelerated, fast |
| Website cloning | `wget --mirror` | Offline analysis, hidden content discovery |
| Scripted exploitation | Python `requests` | Full control, custom logic |
