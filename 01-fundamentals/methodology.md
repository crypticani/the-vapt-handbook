# 01 — Fundamentals: Methodology

> This is your step-by-step workflow for the initial phase of any engagement. Before you scan, before you exploit — you do THIS.

---

## 🔴 Phase 0: Pre-Engagement

### Engagement Checklist

```
□ Written authorization received (scope document / Rules of Engagement)
□ Scope clearly defined:
  □ In-scope domains / IPs
  □ Out-of-scope domains / IPs
  □ Testing window (dates, hours)
  □ Allowed activities (DoS? Social engineering? Physical?)
  □ Notification requirements
  □ Emergency contacts
□ Testing environment ready:
  □ VPN connected (if required)
  □ Burp Suite configured with scope
  □ Note-taking system ready
  □ Screenshots naming convention established
  □ Clock synchronized (important for log correlation)
```

### Environment Setup Workflow

```bash
# 1. Create project directory structure
mkdir -p pentest/{scope,recon,vulns,exploits,evidence,report}

# 2. Document scope
cat > pentest/scope/scope.md << 'EOF'
# Scope Document
## Client: [CLIENT NAME]
## Date: [DATE]
## Tester: [YOUR NAME]

## In-Scope
- example.com and all subdomains
- 10.0.0.0/24
- API: api.example.com

## Out-of-Scope
- production database (10.0.0.50)
- third-party integrations
- DoS/DDoS testing

## Rules of Engagement
- Testing hours: 09:00 - 18:00 UTC
- Maximum scan rate: 100 req/sec
- No account lockout testing without notification
- Emergency contact: security@example.com
EOF

# 3. Set up Burp scope
# Target → Scope → Add → Paste domains

# 4. Prepare wordlists
ls /usr/share/seclists/Discovery/Web-Content/
# common.txt, raft-large-words.txt, directory-list-2.3-medium.txt
```

---

## 🔴 Phase 1: Passive Information Gathering

### Step 1: Open Source Intelligence (OSINT)

```bash
# 1. Technology identification
whatweb target.com
# or use Wappalyzer browser extension

# 2. DNS records
dig target.com ANY
dig target.com MX
dig target.com NS
dig target.com TXT    # SPF, DKIM, DMARC records — reveal email infrastructure

# 3. WHOIS information
whois target.com       # Registration details, name servers, admin contacts

# 4. Historical data
# https://web.archive.org/web/*/target.com
# Look for old pages, removed functionality, exposed credentials

# 5. Google dorking
site:target.com filetype:pdf
site:target.com filetype:sql
site:target.com inurl:admin
site:target.com intitle:"index of"
site:target.com ext:env OR ext:yml OR ext:config
site:target.com "password" OR "api_key" OR "secret"

# 6. GitHub/GitLab search
# Search for: target.com, @target.com
# Look for: API keys, credentials, internal URLs, configuration files

# 7. Certificate Transparency logs
# https://crt.sh/?q=%25.target.com
# Reveals subdomains through SSL certificate registrations
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u
```

### Step 2: Document Findings

```markdown
## Passive Recon Findings

### Technology Stack
- Web Server: nginx/1.18.0
- Framework: Express.js (Node.js)
- CDN: Cloudflare
- CMS: None detected

### DNS Records
- A: 104.26.10.78
- MX: mx.target.com
- NS: ns1.cloudflare.com

### Subdomains Found
- admin.target.com
- api.target.com
- staging.target.com (!!!)
- dev.target.com (!!!)

### Interesting Google Results
- Exposed admin panel: admin.target.com/dashboard
- SQL backup file: target.com/backup/db.sql.bak
```

---

## 🔴 Phase 2: Active Reconnaissance

### Step 1: Network Scanning

```bash
# Quick port scan
nmap -sV -sC -T4 target.com -oA pentest/recon/nmap_initial

# Full port scan (all 65535 ports)
nmap -sV -p- -T4 target.com -oA pentest/recon/nmap_full

# UDP scan (top 100 ports)
nmap -sU --top-ports 100 -T4 target.com -oA pentest/recon/nmap_udp
```

### Step 2: Web Application Analysis

```bash
# 1. Crawl the application
# Use Burp's built-in spider or:
gospider -s https://target.com -d 2 -o pentest/recon/spider

# 2. Directory brute forcing
gobuster dir -u https://target.com -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -t 50 -o pentest/recon/gobuster.txt -x php,asp,aspx,jsp,html,js,json,txt,bak,old

# 3. Check robots.txt and sitemap
curl https://target.com/robots.txt
curl https://target.com/sitemap.xml

# 4. Check for exposed files
curl -s -o /dev/null -w "%{http_code}" https://target.com/.git/HEAD
curl -s -o /dev/null -w "%{http_code}" https://target.com/.env
curl -s -o /dev/null -w "%{http_code}" https://target.com/wp-config.php.bak
curl -s -o /dev/null -w "%{http_code}" https://target.com/web.config
curl -s -o /dev/null -w "%{http_code}" https://target.com/server-status
```

### Step 3: Identify All Entry Points

```markdown
## Entry Points Matrix

| # | Endpoint              | Method | Auth Required | Parameters        | Notes            |
|---|----------------------|--------|---------------|-------------------|------------------|
| 1 | /login               | POST   | No            | user, pass        | Auth endpoint    |
| 2 | /api/v1/users/{id}   | GET    | Yes (JWT)     | id (path)         | Test IDOR        |
| 3 | /search              | GET    | No            | q, page, sort     | Test injection   |
| 4 | /upload              | POST   | Yes           | file (multipart)  | Test RCE         |
| 5 | /api/v1/webhook      | POST   | Yes           | url (JSON body)   | Test SSRF        |
```

---

## 🔴 Phase 3: Vulnerability Identification

### Systematic Testing Approach

For EACH entry point, test in this order:

```
1. Authentication
   □ Is auth required? Try without credentials
   □ Can you use another user's token?
   □ Can you forge/tamper the token?

2. Authorization
   □ Can you access other users' resources? (IDOR)
   □ Can you access admin functions?
   □ Try different HTTP methods

3. Injection
   □ SQL injection (', ", ;, --, UNION, SLEEP)
   □ XSS (<script>, <img>, event handlers)
   □ Command injection (;, |, &&, ``)
   □ Template injection ({{7*7}}, ${7*7})
   □ Path traversal (../, ....// , %2e%2e%2f)

4. Business Logic
   □ Can you change prices, quantities?
   □ Can you skip steps in a workflow?
   □ Race conditions (send same request simultaneously)?
   □ Negative values (negative quantity = credit?)

5. Information Disclosure
   □ Verbose error messages?
   □ Stack traces?
   □ Internal IPs in responses?
   □ Sensitive data in comments/JS?
```

### Testing Priority (Where To Focus Time)

```
HIGH PRIORITY (test first):
├── Authentication endpoints → Account takeover potential
├── File upload functionality → RCE potential
├── Search/query parameters → SQLi/XSS
├── API endpoints with user IDs → IDOR
└── URL/webhook parameters → SSRF

MEDIUM PRIORITY:
├── User profile/settings → Stored XSS, mass assignment
├── Password reset flow → Account takeover chain
├── Export/download functions → IDOR, path traversal
└── Admin interfaces → Privilege escalation

LOWER PRIORITY (but still check):
├── Static pages → Reflected XSS via params
├── Contact/feedback forms → Email injection, stored XSS
└── Third-party integrations → Trust assumption flaws
```

---

## 🔴 Phase 4: Documentation & Evidence

### Evidence Collection Standard

```bash
# For every finding:
# 1. Screenshot of the request in Burp
# 2. Screenshot of the response showing impact
# 3. Copy the exact request (curl format)
# 4. Save the request/response pair from Burp

# Naming convention:
# VULN-001_sqli_login_endpoint_request.png
# VULN-001_sqli_login_endpoint_response.png
# VULN-001_sqli_login_endpoint.txt  (curl command)

# Document in this format:
cat > pentest/vulns/VULN-001.md << 'EOF'
## VULN-001: SQL Injection in Login Endpoint

### Severity: Critical (CVSS 9.8)
### Location: POST /api/v1/login
### Parameter: username

### Description
The username parameter in the login endpoint is vulnerable to SQL injection.
User-supplied input is concatenated directly into a SQL query without sanitization.

### Steps to Reproduce
1. Navigate to https://target.com/login
2. Enter the following in the username field: `admin' OR 1=1-- -`
3. Enter any password
4. Click Login
5. Authentication is bypassed and admin access is granted

### Request
```
POST /api/v1/login HTTP/1.1
Host: target.com
Content-Type: application/json

{"username":"admin' OR 1=1-- -","password":"anything"}
```

### Response
```
HTTP/1.1 200 OK
{"status":"success","token":"eyJhbGciOiJIUzI1NiJ9...","role":"admin"}
```

### Impact
An attacker can bypass authentication entirely and gain administrative access.
This can lead to full database compromise and data exfiltration.

### Remediation
1. Use parameterized queries / prepared statements
2. Implement input validation
3. Apply principle of least privilege to database accounts
EOF
```

---

## 🔴 Methodology Flowchart

```
START
  │
  ├─→ Pre-Engagement
  │     ├─→ Get written authorization
  │     ├─→ Define scope
  │     └─→ Set up environment
  │
  ├─→ Passive Recon
  │     ├─→ OSINT (Google, WHOIS, crt.sh)
  │     ├─→ Technology fingerprinting
  │     └─→ Document findings
  │
  ├─→ Active Recon
  │     ├─→ Port scanning (Nmap)
  │     ├─→ Directory enumeration
  │     ├─→ Spider/crawl application
  │     └─→ Map all entry points
  │
  ├─→ Vulnerability Testing
  │     ├─→ For each entry point:
  │     │     ├─→ Test authentication
  │     │     ├─→ Test authorization
  │     │     ├─→ Test injection
  │     │     ├─→ Test business logic
  │     │     └─→ Test information disclosure
  │     └─→ Document all findings with evidence
  │
  ├─→ Exploitation (if in scope)
  │     ├─→ Validate each finding
  │     ├─→ Demonstrate impact
  │     └─→ Capture evidence
  │
  └─→ Reporting
        ├─→ Executive summary
        ├─→ Technical findings
        ├─→ Remediation recommendations
        └─→ Deliver & debrief
```

---

## 📋 Quick Reference Card

```
BEFORE TESTING:
✓ Authorization in hand
✓ Scope documented
✓ Environment ready

DURING TESTING:
✓ Screenshot everything
✓ Note exact timestamps
✓ Save request/response pairs
✓ Test one thing at a time
✓ Establish baseline first

AFTER TESTING:
✓ Verify all findings are documented
✓ Remove any data/accounts you created
✓ Revoke any access you gained
✓ Notify client of critical findings immediately
```
