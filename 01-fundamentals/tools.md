# 01 — Fundamentals: Tools & Real Usage

> Tools are extensions of your skill, not replacements. Learn what each tool does manually first, then automate.

> 📋 **What You Will Do In This Section**
> - [ ] Install and configure Burp Suite for intercepting traffic
> - [ ] Use curl to craft and send custom HTTP requests
> - [ ] Run Nmap to scan Juice Shop and interpret the results
> - [ ] Use ffuf to discover hidden directories and files
> - [ ] Use Developer Tools (F12) to inspect client-side security

---

## 🔴 Burp Suite (Your Primary Weapon)

> 💡 **Why This Matters**
> Burp Suite sits between your browser and the target, letting you see, modify, and replay every request. 90% of web pentesting happens in Burp. It's the single most important tool you'll use.

### Installation
```bash
# Download from https://portswigger.net/burp/communitydownload
# Linux:
chmod +x burpsuite_community_linux*.sh
./burpsuite_community_linux*.sh

# Verify it runs:
burpsuite &
```

> ✅ **Expected Output**
> Burp Suite splash screen appears → Click "Next" → "Start Burp" → You see the Dashboard tab.

> 🔧 **If Stuck**
> - `burpsuite: command not found` → Run the installer binary directly: `./BurpSuiteCommunity`
> - Java errors → Install Java: `sudo apt install default-jre -y`
> - UI is tiny on HiDPI → Right-click Burp icon → Properties → Add `-Dsun.java2d.uiScale=2.0` to Java args

### Core Workflows

#### Workflow 1: Passive Traffic Recording

```
1. Start Burp → Proxy → Intercept is OFF
2. Configure browser proxy to 127.0.0.1:8080
3. Browse the target application normally
4. All traffic appears in Proxy → HTTP History
5. Review: Filter by domain, method, status code, response length
```

> ✅ **Expected Output**
> After browsing Juice Shop for 5 minutes, you should see 50-100+ requests in HTTP History. Sort by URL to find API endpoints, filter by status to find errors.

#### Workflow 2: Intercepting and Modifying Requests

```
1. Proxy → Intercept → Turn ON
2. In browser, perform an action (login, search, submit form)
3. Burp catches the request before it leaves your machine
4. Modify any part (URL, headers, body)
5. Click "Forward" to send the modified request
6. Check HTTP History for the response
```

#### 🧪 Try It Now — Modify a Juice Shop Search

```
1. Turn Intercept ON in Burp
2. In Juice Shop, search for "apple"
3. In Burp, you'll see: GET /rest/products/search?q=apple
4. Change "apple" to: ' OR 1=1--
5. Click Forward
6. In browser, you should see ALL products (SQL injection worked!)
```

> ✅ **Expected Output**
> The search page shows ALL products instead of just apple products — the `' OR 1=1--` payload made the SQL query return everything.

> 🔧 **If Stuck**
> - Nothing appears in intercept → Make sure your browser is configured to use Burp's proxy (127.0.0.1:8080)
> - Intercepting too many requests (images, CSS) → Right-click in HTTP History → "Add to scope". Then Options → "And URL is in target scope" to only intercept relevant requests

#### Workflow 3: Repeater (Manual Testing)

```
1. Find a request in HTTP History
2. Right-click → "Send to Repeater" (Ctrl+R)
3. In Repeater tab: modify the request
4. Click "Send" → see the response immediately
5. Modify again, Send again — rapid testing loop
```

#### Workflow 4: Intruder (Automated Parameter Testing)

```
1. Send a request to Intruder (Ctrl+I)
2. Positions tab: Highlight the value you want to fuzz → Click "Add §"
3. Payloads tab: Load a wordlist or list of test values
4. Click "Start Attack"
5. Sort results by Status Code or Response Length to find anomalies

Example: Fuzz user IDs for IDOR
Position: GET /api/Users/§1§
Payload: Numbers 1-100
→ Sort by response length — different sizes = different users' data
```

> ✅ **Expected Output (IDOR Fuzzing)**
> ```
> Payload | Status | Length
> 1       | 200    | 523     ← Admin user
> 2       | 200    | 487     ← Regular user
> 3       | 200    | 502     ← Another user
> ...
> 99      | 404    | 31      ← User doesn't exist
> ```
> Different response lengths at `200` status = different user profiles being returned. Each one is an IDOR find.

---

## 🔴 curl (Command-Line HTTP Client)

> 💡 **Why This Matters**
> curl is available on virtually every system. When Burp isn't available (SSH into a server, scripting, CI/CD testing), curl is your go-to. It's also faster for quick checks.

### Essential Flags

```bash
# GET request with verbose output (shows request AND response headers)
curl -v http://localhost:3000

# POST request with JSON body
curl -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test"}'

# Only response headers (HEAD request)
curl -sI http://localhost:3000

# Follow redirects
curl -L http://localhost:3000/some-redirect

# Custom headers
curl -H "Authorization: Bearer TOKEN" \
     -H "X-Forwarded-For: 127.0.0.1" \
     http://localhost:3000/api/Users/

# Save response to file
curl -o response.html http://localhost:3000

# Show only HTTP status code
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000

# Include cookies
curl -b "session=abc123; role=admin" http://localhost:3000

# POST with form data
curl -X POST -d "username=admin&password=test" http://localhost/login.php

# Upload a file
curl -X POST -F "file=@shell.php" http://target.com/upload

# Silent mode with JSON pretty-print
curl -s http://localhost:3000/api/Products/ | python3 -m json.tool
```

#### 🧪 Try It Now — Juice Shop API Exploration with curl

```bash
# 1. Get all products
curl -s http://localhost:3000/api/Products/ | python3 -m json.tool | head -30

# 2. Get a specific product
curl -s http://localhost:3000/api/Products/1 | python3 -m json.tool

# 3. Check what HTTP methods are allowed
curl -s -X OPTIONS http://localhost:3000/api/Products/ -I | grep -i "allow"
```

> ✅ **Expected Output**
> ```json
> // Products list will show items like:
> {
>     "id": 1,
>     "name": "Apple Juice (1000ml)",
>     "description": "The all-time classic.",
>     "price": 1.99,
>     ...
> }
> ```

---

## 🔴 Nmap (Network Scanner)

> 💡 **Why This Matters**
> Nmap tells you WHAT is running on the target — open ports, services, versions. You can't attack a service you don't know exists. Port scanning is always step 1 of any network-level engagement.

```bash
# Quick scan — top 1000 ports
nmap localhost

# Full port scan (all 65535 ports) — slower but thorough
nmap -p- localhost

# Version detection (what software is running on each port)
nmap -sV localhost

# Aggressive scan (version + OS + scripts + traceroute)
nmap -A localhost

# Scan specific ports
nmap -p 80,443,3000,8080 localhost

# Scan with default scripts (safe vulnerability checks)
nmap -sC -sV localhost

# Output to all formats (normal, XML, grepable)
nmap -sV -oA juice_shop_scan localhost
```

#### 🧪 Try It Now — Scan Juice Shop

```bash
# Scan Juice Shop's port
nmap -sV -p 3000 localhost
```

> ✅ **Expected Output**
> ```
> Starting Nmap 7.92 ( https://nmap.org )
> Nmap scan report for localhost (127.0.0.1)
> PORT     STATE SERVICE VERSION
> 3000/tcp open  http    Node.js Express framework
>
> Service detection performed.
> ```
> **What this tells you:** Port 3000 is open, running HTTP, powered by Node.js Express. This confirms our earlier response header analysis.

> 🔧 **If Stuck**
> - `nmap: command not found` → Install: `sudo apt install nmap -y`
> - All ports show as "filtered" → You might be scanning through a firewall. For local Docker targets, scan `127.0.0.1` or `localhost`
> - Scan is very slow → For local targets, add `-T4` for faster timing: `nmap -T4 -sV localhost`

---

## 🔴 ffuf (Fast Web Fuzzer)

> 💡 **Why This Matters**
> Web applications hide pages, API endpoints, backup files, and admin panels that aren't linked anywhere. ffuf finds them by brute-forcing paths with wordlists. This is how you discover attack surface that the developers thought was secret.

```bash
# Install
sudo apt install ffuf -y
# Or: go install github.com/ffuf/ffuf/v2@latest

# Basic directory discovery
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,403

# With specific extensions
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -e .js,.json,.xml,.bak,.old,.php \
  -mc 200,301,302,403

# Filter out responses by size (remove noise)
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301 \
  -fs 0

# Speed control (threads and delay)
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 50 -rate 100

# Parameter fuzzing
ffuf -u "http://localhost:3000/rest/products/search?FUZZ=test" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200 -fs 0
```

#### 🧪 Try It Now — Fuzz Juice Shop

```bash
# Discover hidden paths on Juice Shop
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302 -t 20 2>/dev/null | head -30
```

> ✅ **Expected Output**
> ```
> assets                  [Status: 301, Size: 179, Words: 7]
> ftp                     [Status: 200, Size: 11070, Words: 466]
> profile                 [Status: 200, Size: 6788, Words: 392]
> robots.txt              [Status: 200, Size: 28, Words: 3]
> ...
> ```
> **Key discoveries:**
> - `/ftp` → An FTP directory listing! May contain sensitive files.
> - `/robots.txt` → Lists paths the site doesn't want search engines to index — often sensitive paths.
> - `/profile` → User profile page (check for IDOR).

> 🔧 **If Stuck**
> - `ffuf: command not found` → Install: `sudo apt install ffuf -y` or download from https://github.com/ffuf/ffuf/releases
> - `SecLists not found` → Install: `sudo apt install seclists -y` or `git clone https://github.com/danielmiessler/SecLists.git /usr/share/seclists`
> - Too many results → Add `-fs SIZE` to filter out common response sizes (e.g., `-fs 6788` to hide the default page)

---

## 🔴 Developer Tools (Browser F12)

> 💡 **Why This Matters**
> The browser's Developer Tools reveal client-side code, JavaScript logic, API calls, stored data, and session tokens. Many vulnerabilities are found entirely within the browser — no additional tools needed.

### Key Tabs

```
Console      → Execute JavaScript, test for DOM XSS
Network      → Watch live HTTP requests/responses (alternative to Burp for quick checks)
Application  → View cookies, localStorage, sessionStorage, service workers
Elements     → Inspect/modify HTML in real-time
Sources      → Read JavaScript source code, set breakpoints
```

#### 🧪 Try It Now — Explore Juice Shop in DevTools

```
1. Open Juice Shop in browser (http://localhost:3000)
2. Press F12 to open Developer Tools
3. Console tab → Type: document.cookie
   → See what cookies are accessible to JavaScript

4. Application tab → Cookies → http://localhost:3000
   → See all cookies with their flags (HttpOnly? Secure? SameSite?)

5. Application tab → Local Storage → http://localhost:3000
   → Look for the JWT token stored here after login

6. Network tab → Search for something in Juice Shop
   → Watch the API call appear in real-time
   → Click on it to see request/response details
```

> ✅ **Expected Output**
> ```javascript
> // Console → document.cookie
> "language=en; cookieconsent_status=dismiss; token=eyJ..."
> // If the token is accessible via document.cookie, and there's an XSS,
> // an attacker could steal it!
>
> // Local Storage will show:
> // token: "eyJhbGciOiJSUzI1NiI..." (the JWT)
> // This means JavaScript has access to the auth token
> ```

---

## 📋 Tool Cheatsheet

| Task | Tool | Quick Command |
|------|------|---------------|
| Intercept/modify requests | Burp Suite | Proxy → Intercept ON |
| Quick HTTP request | curl | `curl -v URL` |
| Port scanning | Nmap | `nmap -sV -p- target` |
| Directory discovery | ffuf | `ffuf -u URL/FUZZ -w wordlist` |
| Technology detection | whatweb | `whatweb URL` |
| Response headers | curl | `curl -sI URL` |
| Cookie/token inspection | Browser DevTools | F12 → Application → Cookies |
| JavaScript analysis | Browser DevTools | F12 → Sources |
| API exploration | curl | `curl -s URL/api/ \| jq` |

---

## 🧠 If You're Stuck With Tools

1. **Burp won't intercept**: Browser proxy must be `127.0.0.1:8080`. Install Burp CA cert. Try Firefox first (easiest proxy setup).
2. **curl returns empty output**: Add `-v` for verbose. Check if the URL is correct (include `http://`).
3. **Nmap says all ports filtered**: You might be behind a firewall. For local Docker targets, scan `127.0.0.1`.
4. **ffuf returns thousands of results**: Use `-fs SIZE` to filter common response sizes. Use `-mc 200,301` to only show valid responses.
5. **Can't find SecLists**: Install: `sudo apt install seclists` or clone from GitHub.
6. **Python script import errors**: Install missing packages: `pip3 install requests beautifulsoup4`.
