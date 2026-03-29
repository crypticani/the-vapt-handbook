# 01 — Fundamentals: Hands-On Labs

> Every lab builds muscle memory. Don't skip any. The goal isn't to "complete" the lab — it's to understand what's happening at each step.

---

## 🔧 Lab Environment Setup

```bash
# Start Juice Shop
docker run -d -p 3000:3000 --name juiceshop bkimminich/juice-shop

# Start DVWA
docker run -d -p 80:80 --name dvwa vulnerables/web-dvwa

# Verify both are running
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000  # Should return 200
curl -s -o /dev/null -w "%{http_code}" http://localhost        # Should return 200 or 302

# Configure Burp Suite:
# 1. Open Burp → Proxy → Options → Add listener on 127.0.0.1:8080
# 2. Configure browser to use 127.0.0.1:8080 as HTTP/HTTPS proxy
# 3. Visit http://burpsuite to install Burp's CA certificate
# 4. Import the CA cert into your browser's certificate store
```

---

## 🧪 Lab 1: Intercept and Analyze a Login Request

### Objective
Intercept the Juice Shop login request, understand every component, and identify attack surfaces.

### Steps

**Step 1: Capture the login request**
1. Open Burp Suite → Proxy → Intercept → Turn ON
2. Navigate to `http://localhost:3000/#/login`
3. Enter credentials: `test@test.com` / `password123`
4. Click "Log in"
5. Examine the intercepted request in Burp

**Step 2: Dissect the request**

You should see something like:
```http
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Cookie: language=en; cookieconsent_status=dismiss
Content-Length: 50

{"email":"test@test.com","password":"password123"}
```

**Document the following:**

| Component | Value | Attack Potential |
|-----------|-------|------------------|
| Endpoint | `/rest/user/login` | API endpoint — check for other REST endpoints |
| Method | POST | Try other methods (GET, PUT) |
| Content-Type | application/json | Try XML (XXE), form-urlencoded |
| Cookies | language, cookieconsent | Cookie manipulation, injection |
| Password field | Sent in body | Check if sent in plaintext, hashing? |
| Response | Check for JWT/session | Token analysis |

**Step 3: Analyze the response**
1. Forward the request in Burp
2. Check `HTTP History` tab for the response
3. Look for:
   - Authentication token format (JWT?)
   - Error messages on wrong password (verbose?)
   - Response headers (security headers present?)

**Step 4: Test with wrong credentials**
```http
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"email":"admin@juice-sh.op","password":"wrongpassword"}
```

Note the error message. Does it say "Invalid email" or "Invalid password"? Or just "Invalid email or password"? The former leaks valid usernames.

### ✅ Success Criteria
- [ ] You can explain every line of the request
- [ ] You identified the authentication token format
- [ ] You checked for verbose error messages
- [ ] You documented security headers (present or missing)

---

## 🧪 Lab 2: Map ALL Input Points in a Web Application

### Objective
Systematically identify every input point in Juice Shop. Real pentests start here.

### Steps

**Step 1: Passive crawl with Burp Spider**
1. In Burp → Target → right-click on `http://localhost:3000` → "Scan" (or "Spider" in CE)
2. Browse the application manually while Burp logs all requests
3. Click through every page, form, and function

**Step 2: Document all input points**

Create this matrix:

```markdown
| # | URL                        | Method | Parameters            | Input Type    | Notes                    |
|---|----------------------------|--------|-----------------------|---------------|--------------------------|
| 1 | /rest/user/login           | POST   | email, password       | JSON body     | Auth endpoint            |
| 2 | /rest/products/search      | GET    | q (query parameter)   | URL param     | Search function — SQLi?  |
| 3 | /api/Users                 | POST   | email, password, etc  | JSON body     | Registration             |
| 4 | /rest/user/change-password | GET    | current, new, repeat  | URL params    | Password change (GET?!)  |
| 5 | /profile                   | POST   | username              | Form data     | Profile update — XSS?    |
| 6 | /rest/products/{id}/reviews| PUT    | message               | JSON body     | Product review — stored? |
| 7 | /api/Feedbacks             | POST   | comment, rating       | JSON body     | Feedback submission      |
| 8 | /rest/basket/{id}          | GET    | id (path param)       | URL path      | IDOR?                    |
```

**Step 3: Test hidden parameters**

Use Burp's Param Miner extension or manually test for hidden parameters:
```bash
# Common hidden parameters to test
?debug=true
?admin=true
?test=true
?role=admin
?internal=1
```

**Step 4: Check for API documentation**
```bash
# Common API doc endpoints
curl http://localhost:3000/api
curl http://localhost:3000/api-docs
curl http://localhost:3000/swagger.json
curl http://localhost:3000/swagger-ui.html
curl http://localhost:3000/.well-known/openapi.json
```

### ✅ Success Criteria
- [ ] You found at least 15 unique input points
- [ ] You documented which accept URL params vs body params vs headers
- [ ] You checked for API documentation endpoints
- [ ] You identified at least 3 interesting attack surfaces

---

## 🧪 Lab 3: Modify and Replay Requests

### Objective
Modify intercepted requests to test how the application handles unexpected input.

### Steps

**Step 1: Capture and modify a search request**
1. In Juice Shop, use the search bar to search for "apple"
2. Capture the request in Burp:
```http
GET /rest/products/search?q=apple HTTP/1.1
Host: localhost:3000
```

3. Send to Repeater (Ctrl+R)
4. Modify the `q` parameter with each of these payloads:

```
q=apple'                          # SQL injection test (look for error)
q=apple" OR 1=1--                 # SQL injection
q=<script>alert(1)</script>       # XSS test
q=apple%00                        # Null byte injection
q=../../../etc/passwd             # Path traversal
q=apple{{7*7}}                    # SSTI test (look for "49" in response)
q=                                # Empty input — does it return everything?
q=AAAA...AAAA (5000 chars)        # Buffer overflow / length handling
```

5. For each payload, document:
   - HTTP status code
   - Response body (error message? Stack trace?)
   - Response length (different = interesting)

**Step 2: Replay with modified headers**
```http
GET /rest/products/search?q=apple HTTP/1.1
Host: localhost:3000
User-Agent: ' OR 1=1-- -
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: evil.com
```

**Step 3: Method tampering**
```http
# Original
GET /rest/products/search?q=apple HTTP/1.1

# Try
POST /rest/products/search HTTP/1.1
Content-Type: application/json
{"q": "apple"}

# Try
PUT /rest/products/search HTTP/1.1
Content-Type: application/json
{"q": "apple"}
```

### ✅ Success Criteria
- [ ] You sent at least 5 different payloads through the search parameter
- [ ] You identified different response behaviors for different inputs
- [ ] You tested at least 3 header modifications
- [ ] You tried at least 2 different HTTP methods

---

## 🧪 Lab 4: Response Analysis & Information Gathering

### Objective
Extract maximum intelligence from HTTP responses.

### Steps

**Step 1: Collect server information**
```bash
# Use curl for clean response headers
curl -sI http://localhost:3000

# Expected output analysis:
# Server: nginx/1.x or express
# X-Powered-By: Express (INFORMATION LEAK — should be removed)
# X-Content-Type-Options: nosniff (good — prevents MIME sniffing)
# X-Frame-Options: SAMEORIGIN (good — prevents clickjacking)

# What's MISSING?
# Content-Security-Policy → CSP not set → XSS easier
# Strict-Transport-Security → HSTS not set
```

**Step 2: Force error messages**
```bash
# Send malformed requests to get verbose errors
curl http://localhost:3000/rest/products/search?q='
curl http://localhost:3000/rest/products/search?q=%27
curl http://localhost:3000/api/Users/undefined
curl -X POST http://localhost:3000/rest/user/login -H "Content-Type: application/json" -d '{}'
curl -X POST http://localhost:3000/rest/user/login -H "Content-Type: application/json" -d '{"email":null}'
```

**Step 3: Compare response sizes**
```bash
# Correct password attempt vs wrong password — different response size = username oracle
curl -s http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@juice-sh.op","password":"wrong"}' | wc -c

curl -s http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"nonexistent@test.com","password":"wrong"}' | wc -c

# Different sizes = the server is distinguishing between valid and invalid emails
```

### ✅ Success Criteria
- [ ] You identified the web framework in use
- [ ] You found at least 2 missing security headers
- [ ] You extracted an error message that reveals internal information
- [ ] You tested for username enumeration via response differences

---

## 🧪 Lab 5: Cookie and Session Analysis

### Objective
Analyze session management implementation and identify weaknesses.

### Steps

**Step 1: Register and login to Juice Shop**
1. Register a new account at `http://localhost:3000/#/register`
2. Login and capture the response
3. Extract the session token from the response

**Step 2: Decode the JWT**
```bash
# The login response contains a JWT token
# Copy the token and decode it:

# Method 1: Command line
echo 'PASTE_JWT_HERE' | cut -d'.' -f1 | base64 -d 2>/dev/null | python3 -m json.tool
echo 'PASTE_JWT_HERE' | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# Method 2: Use jwt.io in browser
# Method 3: jwt_tool
python3 jwt_tool.py PASTE_JWT_HERE
```

**Step 3: Identify JWT weaknesses**
```bash
# Check the algorithm in the header
# If it says HS256, try:

# 1. Algorithm None attack
python3 jwt_tool.py PASTE_JWT_HERE -X a

# 2. Key brute force
python3 jwt_tool.py PASTE_JWT_HERE -C -d /usr/share/wordlists/rockyou.txt

# 3. Manual tampering
# Decode payload, modify {"role":"customer"} to {"role":"admin"}, re-encode
```

**Step 4: Test session handling**
1. Login in two browser tabs — do you get different tokens?
2. Log out in one tab — does the token still work in the other?
3. Copy the token, clear cookies, paste the token back — still authenticated?

### ✅ Success Criteria
- [ ] You decoded the JWT and identified all claims
- [ ] You identified the signing algorithm
- [ ] You tested at least 2 JWT attack vectors
- [ ] You tested session invalidation on logout

---

## 🧪 Lab 6: DVWA — Hands-On With Difficulty Levels

### Objective
Use DVWA's adjustable difficulty to understand how defenses work (and how to bypass them).

### Steps

**Step 1: Setup DVWA**
1. Navigate to `http://localhost/setup.php`
2. Click "Create / Reset Database"
3. Login: `admin` / `password`
4. Go to "DVWA Security" → Set to "Low"

**Step 2: Walk through each vulnerability at LOW difficulty**

For each module (Brute Force, Command Injection, CSRF, File Inclusion, File Upload, SQL Injection, XSS), do:

1. Submit a normal input → document the request
2. Submit a malicious input → document the behavior
3. Read the source code (click "View Source") → understand WHY it worked

**Step 3: Increase to MEDIUM and re-exploit**

| Module | Low Bypass | Medium Defense | Medium Bypass |
|--------|------------|----------------|---------------|
| SQLi | `' OR 1=1-- -` | Input escaping | Try different encodings |
| XSS Reflected | `<script>alert(1)</script>` | Script tag filtered | `<img onerror=alert(1) src=x>` |
| Command Injection | `; id` | `;` blacklisted | `| id` or `|| id` |
| File Inclusion | `../../etc/passwd` | Path checks | `....//....//etc/passwd` |

**Step 4: Try HIGH difficulty**

At HIGH, defenses are stronger. Document what defense was added and research bypass techniques. This is where you build real skill.

### ✅ Success Criteria
- [ ] You completed all modules at LOW difficulty
- [ ] You bypassed at least 3 modules at MEDIUM difficulty
- [ ] You read the source code for each difficulty level and understood the defense
- [ ] You attempted at least 2 modules at HIGH difficulty

---

## 🧪 Lab 7: Build Your Own HTTP Analysis Script

### Objective
Write a Python script that analyzes HTTP responses for security weaknesses. This builds your automation mindset.

```python
#!/usr/bin/env python3
"""
HTTP Security Analyzer
Checks for common security misconfigurations in HTTP responses.
"""

import requests
import sys
import json
from urllib.parse import urlparse

def analyze_headers(url):
    """Analyze security headers in HTTP response."""
    try:
        resp = requests.get(url, verify=False, timeout=10, allow_redirects=True)
    except requests.exceptions.RequestException as e:
        print(f"[!] Request failed: {e}")
        return

    print(f"\n{'='*60}")
    print(f"TARGET: {url}")
    print(f"STATUS: {resp.status_code}")
    print(f"SERVER: {resp.headers.get('Server', 'Not disclosed')}")
    print(f"{'='*60}")

    # Security headers check
    security_headers = {
        'Strict-Transport-Security': 'HSTS missing — no forced HTTPS',
        'Content-Security-Policy': 'CSP missing — XSS easier to exploit',
        'X-Content-Type-Options': 'Missing — MIME sniffing possible',
        'X-Frame-Options': 'Missing — Clickjacking possible',
        'X-XSS-Protection': 'Missing — Browser XSS filter not enforced',
        'Referrer-Policy': 'Missing — Referer header leaks URLs',
        'Permissions-Policy': 'Missing — Browser features not restricted',
    }

    print("\n[*] Security Headers Analysis:")
    for header, warning in security_headers.items():
        value = resp.headers.get(header)
        if value:
            print(f"  ✅ {header}: {value}")
        else:
            print(f"  ❌ {header}: {warning}")

    # Information leakage
    print("\n[*] Information Leakage:")
    leak_headers = ['Server', 'X-Powered-By', 'X-AspNet-Version', 'X-AspNetMvc-Version']
    for header in leak_headers:
        value = resp.headers.get(header)
        if value:
            print(f"  ⚠️  {header}: {value} — Technology disclosed!")

    # Cookie analysis
    print("\n[*] Cookie Analysis:")
    for cookie in resp.cookies:
        flags = []
        if cookie.secure:
            flags.append("✅ Secure")
        else:
            flags.append("❌ NOT Secure")
        if 'httponly' in cookie._rest.keys() or cookie.has_nonstandard_attr('HttpOnly'):
            flags.append("✅ HttpOnly")
        else:
            flags.append("❌ NOT HttpOnly")
        print(f"  Cookie: {cookie.name} = {cookie.value[:20]}... | {' | '.join(flags)}")

    # CORS check
    print("\n[*] CORS Configuration:")
    resp_cors = requests.get(url, headers={'Origin': 'https://evil.com'}, verify=False, timeout=10)
    acao = resp_cors.headers.get('Access-Control-Allow-Origin')
    if acao:
        if acao == '*':
            print(f"  ❌ CORS wildcard (*) — Any origin can read responses!")
        elif 'evil.com' in acao:
            print(f"  ❌ CORS reflects arbitrary origin — Exploitable!")
        else:
            print(f"  ⚠️  CORS: {acao}")
    else:
        print(f"  ✅ No CORS headers for evil.com origin")

    return resp

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <URL>")
        sys.exit(1)
    analyze_headers(sys.argv[1])
```

### Exercise
1. Save this script and run it against Juice Shop
2. Run it against DVWA
3. Add these features:
   - Check for common API documentation endpoints
   - Check for `.git`, `.env`, `robots.txt` exposure
   - Output results to a JSON file

### ✅ Success Criteria
- [ ] Script runs and produces output
- [ ] You added at least 2 additional checks
- [ ] You ran it against both Juice Shop and DVWA
- [ ] You understand WHY each check matters

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: Login request intercepted and fully analyzed
- [ ] Lab 2: 15+ input points documented
- [ ] Lab 3: Multiple payloads tested with documented responses
- [ ] Lab 4: Server info extracted, security headers analyzed
- [ ] Lab 5: JWT decoded and attack vectors tested
- [ ] Lab 6: DVWA Low completed, Medium attempted
- [ ] Lab 7: HTTP analyzer script built and extended

---

## 🧠 If You're Stuck

1. **Can't intercept traffic**: Check Burp proxy settings. Browser must be configured to use 127.0.0.1:8080. Install Burp CA cert.
2. **Juice Shop won't start**: `docker logs juiceshop` — usually a port conflict. Try `-p 3001:3000`.
3. **DVWA database error**: Visit `/setup.php` and click "Create / Reset Database" again.
4. **Don't understand the response**: Google the exact error message. StackOverflow and PortSwigger Academy are your friends.
5. **Feel overwhelmed**: Do one lab per day. Quality over speed.
