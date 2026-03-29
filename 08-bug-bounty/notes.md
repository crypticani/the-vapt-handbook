# 08 — Bug Bounty: Core Concepts & Attacker Thinking

> Bug bounty is where penetration testing meets entrepreneurship. You find real vulnerabilities in production systems and get paid for it.

---

## 🔴 The Bug Bounty Mindset

```
Bug bounty is NOT scanning targets with automated tools.
Bug bounty IS:
- Finding vulnerabilities others missed
- In places others didn't look
- Using techniques others don't know
- And reporting them CLEARLY
```

### What Makes a Good Bug Bounty Hunter

```
1. PATIENCE: You'll submit hundreds of reports before your first bounty
2. CREATIVITY: The obvious bugs are already found
3. PERSISTENCE: The same target that had nothing yesterday might have new features today
4. SPECIALIZATION: Be the BEST at one vulnerability class, not mediocre at all
5. WRITING: 50% of bounty success is report quality
```

---

## 🔴 Target Selection Strategy

### Where To Start

```
TIER 1 — START HERE (Lower competition, faster response):
├── VDPs (Vulnerability Disclosure Programs) — No bounty but builds reputation
├── New programs on HackerOne/Bugcrowd — Less competition
├── Programs with wide scope (*.target.com) — More attack surface
└── Programs with no live hackers — Check leaderboard activity

TIER 2 — INTERMEDIATE:
├── Established programs with fast response
├── Programs that accept mobile/API/hardware
├── Companies with large attack surfaces
└── Programs that reward creative chains

TIER 3 — ADVANCED (High competition, high payouts):
├── Big tech (Google, Facebook, Microsoft, Apple)
├── Financial services
├── Cryptocurrency platforms
└── Critical infrastructure
```

### Program Analysis

Before targeting a program, analyze it:

```markdown
## Program Evaluation: target.com

### Scope
- *.target.com → Wide scope! ✅
- api.target.com → API testing ✅
- Mobile app (iOS/Android) → Less competition ✅
- Out of scope: payments.target.com ❌

### Bounty Table
| Severity | Payout |
|----------|--------|
| Critical | $5,000 - $20,000 |
| High | $1,000 - $5,000 |
| Medium | $250 - $1,000 |
| Low | $50 - $250 |

### Response Stats
- Average response time: 2 days ✅
- Average time to bounty: 14 days ✅
- Resolved reports: 200+ ✅

### Red Flags
- No response > 30 days → Skip
- Only accepts critical → Hard to get paid
- Very restrictive scope → Limited attack surface
- "We don't pay for X" where X is common → Frustrating
```

### 🧠 Attacker Thinking: Where Others Don't Look

```
Everyone tests: Main website, login page, search box
Few people test: 
  - Mobile API endpoints (intercept with Burp)
  - Old API versions (v1, v2 when v3 is current)
  - Subdomains found through cert transparency
  - Newly deployed features (monitor for changes)
  - PDF generators, export functions
  - Webhook functionality
  - Third-party integrations
  - Error pages and debug endpoints
  - GraphQL introspection
  - WebSocket endpoints
```

---

## 🔴 Recon Methodology (Step-by-Step)

### Stage 1: Asset Discovery (1-2 hours)

```bash
#!/bin/bash
# bb_recon.sh — Bug bounty recon pipeline
TARGET=$1
OUTDIR="bb_${TARGET}_$(date +%Y%m%d)"
mkdir -p $OUTDIR/{subs,urls,tech,vulns}

echo "[1/6] Subdomain enumeration..."
subfinder -d $TARGET -silent > $OUTDIR/subs/subfinder.txt
amass enum -passive -d $TARGET -o $OUTDIR/subs/amass.txt 2>/dev/null
curl -s "https://crt.sh/?q=%25.${TARGET}&output=json" | jq -r '.[].name_value' | sort -u > $OUTDIR/subs/crtsh.txt
cat $OUTDIR/subs/*.txt | sort -u > $OUTDIR/subs/all.txt
echo "  [+] $(wc -l < $OUTDIR/subs/all.txt) unique subdomains"

echo "[2/6] Resolving live hosts..."
cat $OUTDIR/subs/all.txt | httpx -silent -status-code -title -tech-detect -follow-redirects > $OUTDIR/tech/live.txt
echo "  [+] $(wc -l < $OUTDIR/tech/live.txt) live hosts"

echo "[3/6] Gathering URLs..."
echo $TARGET | waybackurls 2>/dev/null > $OUTDIR/urls/wayback.txt
cat $OUTDIR/urls/wayback.txt | grep '?' | sort -u > $OUTDIR/urls/params.txt
echo "  [+] $(wc -l < $OUTDIR/urls/params.txt) URLs with parameters"

echo "[4/6] Technology detection..."
cat $OUTDIR/tech/live.txt | grep -iE "staging|dev|test|admin|api|internal" > $OUTDIR/tech/interesting.txt
echo "  [+] $(wc -l < $OUTDIR/tech/interesting.txt) interesting hosts"

echo "[5/6] Quick vulnerability scan..."
nuclei -l $OUTDIR/tech/live.txt -severity critical,high -silent -o $OUTDIR/vulns/nuclei.txt 2>/dev/null
echo "  [+] $(wc -l < $OUTDIR/vulns/nuclei.txt 2>/dev/null || echo 0) high/critical findings"

echo "[6/6] Summary saved to $OUTDIR/"
```

### Stage 2: Deep Dive (Per Target)

```bash
# For each interesting subdomain:

# 1. Technology fingerprint
whatweb -v https://subdomain.target.com

# 2. Directory/file discovery
ffuf -u https://subdomain.target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -mc 200,301,302,401,403 -t 50 -rate 100

# 3. JavaScript analysis
# Download all JS files and search for endpoints, secrets, API keys
curl -s https://subdomain.target.com | grep -oP 'src="[^"]*\.js[^"]*"' | while read src; do
  url=$(echo $src | cut -d'"' -f2)
  echo "=== $url ==="
  curl -s "https://subdomain.target.com$url" | grep -oP '(\/api\/[a-zA-Z0-9/_-]+|["'"'"'][a-zA-Z0-9_]+_key["'"'"']\s*:\s*["'"'"'][^"'"'"']+["'"'"'])'
done

# 4. API endpoint testing (if API found)
curl -s https://target.com/api/swagger.json | jq '.paths | keys'
# Test each endpoint for IDOR, injection, auth bypass

# 5. Parameter discovery
ffuf -u "https://target.com/api/endpoint?FUZZ=test" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200,301 -fs 0
```

### Stage 3: Continuous Monitoring

```bash
# Monitor for new subdomains (run daily via cron)
#!/bin/bash
TARGET=$1
OLD="subs_previous.txt"
NEW="subs_current.txt"

subfinder -d $TARGET -silent > $NEW
diff <(sort $OLD) <(sort $NEW) | grep '^>' | sed 's/^> //' > new_subs.txt

if [ -s new_subs.txt ]; then
  echo "New subdomains found for $TARGET:"
  cat new_subs.txt
  # Send notification (Telegram, Slack, Discord)
  # Test new subdomains immediately:
  cat new_subs.txt | httpx -silent -status-code -title
fi

cp $NEW $OLD
```

---

## 🔴 Automation Pipelines

### Subdomain Monitor + Scanner

```bash
#!/bin/bash
# Full automation pipeline
# Run daily: crontab -e → 0 6 * * * /path/to/pipeline.sh target.com

TARGET=$1
DATE=$(date +%Y%m%d)
BASE="$HOME/bb/$TARGET"
mkdir -p $BASE/daily

# 1. New subdomain discovery
subfinder -d $TARGET -silent | sort -u > $BASE/daily/subs_$DATE.txt

# 2. Compare with previous
if [ -f $BASE/daily/subs_latest.txt ]; then
  comm -13 $BASE/daily/subs_latest.txt $BASE/daily/subs_$DATE.txt > $BASE/daily/new_$DATE.txt
  NEW_COUNT=$(wc -l < $BASE/daily/new_$DATE.txt)
  if [ $NEW_COUNT -gt 0 ]; then
    echo "[!] $NEW_COUNT new subdomains for $TARGET"
    cat $BASE/daily/new_$DATE.txt | httpx -silent | nuclei -severity critical,high -silent
  fi
fi

cp $BASE/daily/subs_$DATE.txt $BASE/daily/subs_latest.txt
```

---

## 🔴 Report Writing (THE MOST IMPORTANT SKILL)

### Report Template

```markdown
# [Vulnerability Title] — [Target]

## Summary
[One paragraph: What's the bug, where is it, what's the impact?]

## Severity
**CVSS Score**: X.X (Critical/High/Medium/Low)
**Vector**: [e.g. Network/Adjacent/Local]
**Impact**: [Confidentiality/Integrity/Availability]

## Environment
- **URL**: https://target.com/vulnerable-endpoint
- **Parameter**: `username`
- **Method**: POST
- **Testing Date**: 2024-XX-XX
- **Browser/Tool**: Burp Suite CE 2024.x

## Steps to Reproduce
1. Navigate to https://target.com/login
2. Enter `admin' OR 1=1-- -` in the username field
3. Enter any value in the password field
4. Click "Login"
5. Observe: Authentication is bypassed and admin dashboard is displayed

## Proof of Concept

### Request
```http
POST /api/login HTTP/1.1
Host: target.com
Content-Type: application/json

{"username": "admin' OR 1=1-- -", "password": "anything"}
```

### Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"status": "success", "role": "admin", "token": "eyJ..."}
```

### Evidence
[Screenshot of admin dashboard accessed without valid credentials]
[Screenshot of Burp showing the request/response]

## Impact
An unauthenticated attacker can bypass the login mechanism and gain administrative access to the application. This allows the attacker to:
1. View and modify all user data (PII exposure)
2. Modify application configuration
3. Potentially extract the entire database
4. Escalate to remote code execution via the admin panel

## Remediation
1. **Use parameterized queries** (prepared statements) for all database interactions
2. **Implement input validation** — reject input containing SQL special characters
3. **Apply the principle of least privilege** — database accounts used by the application should have minimal permissions
4. **Enable WAF rules** to detect and block SQL injection attempts

## References
- OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
- CWE-89: https://cwe.mitre.org/data/definitions/89.html
```

### Report Writing Rules

```
DO:
✅ Be clear and concise — the reader should understand in 30 seconds
✅ Provide exact steps to reproduce — another person should be able to replicate
✅ Include screenshots and request/response pairs
✅ Explain the business impact in real terms
✅ Suggest specific remediation
✅ Use professional language and formatting
✅ Include CVSS score with justification
✅ Test your reproduction steps before submitting

DON'T:
❌ Submit noisy scanner output as a report
❌ Claim critical severity for everything
❌ Write vague descriptions ("I found a vulnerability")
❌ Submit without verifying the issue exists
❌ Include unnecessary technical jargon
❌ Submit duplicates without checking
❌ Be rude or demanding in communications
❌ Threaten public disclosure before responsible timeline
```

### Common Severity Ratings

| Finding | Typical Severity | Notes |
|---------|-----------------|-------|
| RCE (Remote Code Execution) | Critical | Always critical |
| SQLi with data access | Critical/High | Depends on data sensitivity |
| Authentication bypass | Critical/High | Depends on what's accessed |
| SSRF to cloud metadata | High/Critical | If credentials leaked: critical |
| Stored XSS | Medium/High | Depends on context (admin panel: high) |
| IDOR with PII | Medium/High | Depends on data type |
| Reflected XSS | Medium | Unless it leads to account takeover |
| CSRF on sensitive action | Medium | Password change, email change |
| Open Redirect | Low/Medium | Unless chained with other bugs |
| Missing security headers | Low/Info | Unless impact demonstrated |
| Self-XSS | Out of scope | Most programs don't accept |

---

## 🔴 Common Mistakes In Bug Bounty

```
1. SUBMITTING SCANNER OUTPUT: Nuclei/Nessus results are not reports. Verify and explain.
2. OVER-REPORTING: Submitting 50 "informational" findings wastes everyone's time.
3. DUPLICATE HUNTING: Check if something is already reported before investing time.
4. IGNORING SCOPE: Testing out-of-scope assets = violation of terms, possible legal trouble.
5. NOT READING PROGRAM POLICY: Each program has specific rules. Read them.
6. POOR REPORT QUALITY: The #1 reason valid bugs don't get paid.
7. GIVING UP TOO EARLY: Most hunters don't find their first bug for weeks or months.
8. NOT SPECIALIZING: Jack of all trades = master of none. Pick 2-3 vuln types and go deep.
```

---

## 🔴 Bug Bounty Platforms

| Platform | URL | Notes |
|----------|-----|-------|
| HackerOne | hackerone.com | Largest platform, most programs |
| Bugcrowd | bugcrowd.com | Good for beginners, VRT system |
| Intigriti | intigriti.com | European focus |
| YesWeHack | yeswehack.com | European focus |
| Synack | synack.com | Invite-only, consistent work |
| Open Bug Bounty | openbugbounty.org | XSS-focused, no account needed |

---

## 📋 Bug Bounty Daily Workflow

```
MORNING (30 min):
□ Check for new programs on HackerOne/Bugcrowd
□ Run subdomain monitoring for active targets
□ Read new disclosures (hackerone.com/hacktivity)

HUNTING SESSION (2-4 hours):
□ Pick ONE target
□ Run recon pipeline (if new target)
□ Focus on ONE vulnerability type
□ Test 3-5 interesting endpoints deeply
□ Document anything suspicious

EVENING (30 min):
□ Write up any findings
□ Review notes from the day
□ Plan tomorrow's targets
□ Read one blog post about new techniques
```

---

## 🧠 If You're Stuck

1. **Can't find bugs**: Narrow your focus. Test ONE vulnerability type on ONE target for a week.
2. **Reports getting closed as N/A**: Improve report quality. Read accepted reports on HackerOne Hacktivity.
3. **Duplicates**: Focus on new features, recently acquired companies, or less popular programs.
4. **Don't know what to test**: Pick one OWASP Top 10 category and become an expert.
5. **Discouraged**: Every successful hunter has had months of nothing. Persistence wins.
