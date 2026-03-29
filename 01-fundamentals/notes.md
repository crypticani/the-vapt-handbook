# 01 — Fundamentals: Core Concepts & Attacker Thinking

> Before you exploit anything, you need to understand how the web works at the wire level. This isn't theory for theory's sake — every concept here maps directly to an attack vector.

---

## 🔴 HTTP Deep Dive

### The Request Lifecycle (Attacker's View)

When a browser sends a request, here's what happens — and where you attack:

```
[User Input] → [Client-Side JS] → [HTTP Request] → [Load Balancer/WAF] → [Web Server] → [App Logic] → [Database/API] → [Response]
     ↑              ↑                    ↑                  ↑                  ↑              ↑             ↑
   DOM XSS    Client bypass       Header injection      WAF bypass        SSRF/IDOR      SQLi/CMDi    Info disclosure
```

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

**JWT Attack Vectors:**
1. **Algorithm confusion**: Change `alg` from `RS256` to `HS256` and sign with the public key
2. **None algorithm**: Set `alg` to `none` and remove the signature
3. **Weak secret**: Brute-force the HMAC secret with `hashcat` or `jwt_tool`
4. **Missing expiration**: Token never expires = persistent access
5. **Payload tampering**: Change `role: user` to `role: admin` if signature isn't verified

```bash
# Decode JWT quickly
echo 'JWT_TOKEN_HERE' | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool

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

### Mapping Input to Vulnerability Classes

| Input Destination | Primary Vulnerability | Secondary |
|-------------------|----------------------|-----------|
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

---

## 🔴 OWASP Top 10 — Mapped To Real Attacks

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

- [ ] Scope defined (IPs, domains, excluded targets)
- [ ] Written authorization obtained
- [ ] Rules of engagement clear (testing hours, DoS allowed?, social engineering?)
- [ ] Lab environment ready (Burp, VM, wordlists)
- [ ] Note-taking system set up (CherryTree, Obsidian, plain markdown)
- [ ] Screenshot tool ready
- [ ] Baseline requests captured for target
- [ ] Emergency contacts documented (if something breaks)
