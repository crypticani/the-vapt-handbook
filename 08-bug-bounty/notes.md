# 08 — Bug Bounty: Core Concepts & Attacker Thinking

> Bug bounty is where penetration testing meets entrepreneurship. You find real vulnerabilities in production systems and get paid for it.

> 📋 **What You Will Do In This Section**
> - [ ] Develop the bug bounty mindset — find what others miss
> - [ ] Evaluate and select bug bounty programs strategically
> - [ ] Build and execute a full recon pipeline (subdomain → live hosts → tech detection)
> - [ ] Write professional-quality bug bounty reports
> - [ ] Understand common severity ratings and bounty ranges
> - [ ] Set up continuous monitoring for new attack surface

---

## 🔴 The Bug Bounty Mindset

> 💡 **Why This Matters**
> Bug bounty is a meritocracy — you get paid based on the quality and impact of vulnerabilities you find. The top 1% of hunters earn $100K+/year. The difference between earning $0 and earning $50K is methodology + persistence + report quality. This section teaches all three.

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
1. PATIENCE: You'll spend weeks before your first bounty
2. CREATIVITY: The obvious bugs are already found — think laterally
3. PERSISTENCE: Check the same target weekly — new features = new bugs
4. SPECIALIZATION: Be the BEST at 2-3 vulnerability classes
5. WRITING: 50% of bounty success is report quality
```

---

## 🔴 Target Selection Strategy

> 💡 **Why This Matters**
> Spending 10 hours on the wrong program = $0. Spending 10 hours on the right program = $1,000+. Strategic target selection is THE most underrated skill in bug bounty.

### Where To Start

```
TIER 1 — START HERE (Lower competition, faster triage):
├── VDPs (Vulnerability Disclosure Programs) — No bounty but builds reputation
├── New programs on HackerOne/Bugcrowd — Less competition
├── Programs with wide scope (*.target.com) — More attack surface
└── Programs with no live hackers — Check leaderboard activity

TIER 2 — INTERMEDIATE:
├── Established programs with fast response time
├── Programs that accept mobile/API/hardware
└── Companies with large attack surfaces

TIER 3 — ADVANCED (High competition, high payouts):
├── Big tech (Google, Facebook, Microsoft, Apple)
├── Financial services
└── Cryptocurrency platforms
```

#### 🧪 Try It Now — Program Evaluation

```bash
echo "=== Bug Bounty Program Evaluation Template ==="
echo ""
echo "Program: ________________"
echo ""
echo "Scope:"
echo "  [ ] Wide scope (*.target.com)?"
echo "  [ ] API endpoints in scope?"
echo "  [ ] Mobile apps in scope?"
echo ""
echo "Payout:"
echo "  Critical: $______ - $______"
echo "  High:     $______ - $______"
echo "  Medium:   $______ - $______"
echo ""
echo "Response Time:"
echo "  [ ] Average response < 7 days?"
echo "  [ ] Average bounty time < 30 days?"
echo ""
echo "Red Flags:"
echo "  [ ] No response > 30 days → SKIP"
echo "  [ ] Very restrictive scope → SKIP"
echo "  [ ] Only accepts critical → HARD to get paid"
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

### Stage 1: Asset Discovery

> 💡 **Why This Matters**
> The more attack surface you discover, the more places you can look for bugs. The best bug bounty hunters have 10x more subdomains in their lists than average hunters — and that's where they find bugs others miss.

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
cat $OUTDIR/subs/all.txt | httpx -silent -status-code -title -tech-detect > $OUTDIR/tech/live.txt
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

> ✅ **Expected Output**
> ```
> [1/6] Subdomain enumeration...
>   [+] 247 unique subdomains
> [2/6] Resolving live hosts...
>   [+] 89 live hosts
> [3/6] Gathering URLs...
>   [+] 1,247 URLs with parameters
> [4/6] Technology detection...
>   [+] 12 interesting hosts
> [5/6] Quick vulnerability scan...
>   [+] 3 high/critical findings
> [6/6] Summary saved to bb_target.com_20240115/
> ```

> 🔧 **If Stuck**
> - subfinder not installed → `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest`
> - No subdomains found → Try `curl -s "https://crt.sh/?q=%25.TARGET&output=json" | jq -r '.[].name_value'` manually
> - httpx too slow → Add `-threads 50 -rate-limit 100` flags

### Stage 2: Deep Dive (Per Target)

```bash
# For each interesting subdomain:

# 1. Technology fingerprint
whatweb -v https://subdomain.target.com

# 2. Directory/file discovery
ffuf -u https://subdomain.target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -mc 200,301,302,401,403 -t 50 -rate 100

# 3. API endpoint testing
curl -s https://target.com/api/swagger.json | jq '.paths | keys'

# 4. Parameter discovery
ffuf -u "https://target.com/api/endpoint?FUZZ=test" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200,301 -fs 0
```

### Stage 3: Continuous Monitoring

```bash
#!/bin/bash
# monitor.sh — Run daily via cron
TARGET=$1
OLD="subs_previous.txt"
NEW="subs_current.txt"

subfinder -d $TARGET -silent > $NEW
diff <(sort $OLD) <(sort $NEW) | grep '^>' | sed 's/^> //' > new_subs.txt

if [ -s new_subs.txt ]; then
  echo "New subdomains for $TARGET:"
  cat new_subs.txt
  cat new_subs.txt | httpx -silent -status-code -title
fi
cp $NEW $OLD
```

---

## 🔴 Report Writing (THE MOST IMPORTANT SKILL)

> 💡 **Why This Matters**
> A valid P1 bug with a bad report might get paid $500. The same bug with a great report gets $5,000. Report quality determines whether your finding is triaged in 2 hours or 2 weeks. Invest 50% of your time in the report.

### Report Template

```markdown
# [Vulnerability Title] — [Target]

## Summary
[One paragraph: What's the bug, where is it, what's the impact?]

## Severity
**CVSS Score**: X.X (Critical/High/Medium/Low)

## Steps to Reproduce
1. Navigate to https://target.com/endpoint
2. [Exact click-level steps]
3. [Observe the vulnerability]

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
{"status": "success", "role": "admin", "token": "eyJ..."}
```

## Impact
[Business-level description of what an attacker can do]

## Remediation
1. [Specific technical fix]
2. [Preventive measure]
```

### Report Writing Rules

```
DO:
✅ Be clear and concise — reader should understand in 30 seconds
✅ Provide exact steps to reproduce — anyone should replicate
✅ Include screenshots and request/response pairs
✅ Explain business impact in real terms
✅ Suggest specific remediation
✅ Test your reproduction steps before submitting

DON'T:
❌ Submit noisy scanner output as a report
❌ Claim critical severity for everything
❌ Write vague descriptions ("I found a vulnerability")
❌ Submit without verifying the issue exists
❌ Threaten public disclosure before responsible timeline
```

### Common Severity Ratings

| Finding | Typical Severity | Average Bounty |
|---------|-----------------|---------------|
| RCE | Critical | $10,000+ |
| SQLi with data access | Critical/High | $3,000-10,000 |
| Authentication bypass | Critical/High | $2,000-10,000 |
| SSRF → cloud metadata | High/Critical | $3,000+ |
| Stored XSS (admin) | Medium/High | $1,000-5,000 |
| IDOR with PII | Medium/High | $1,000-5,000 |
| Reflected XSS | Medium | $200-1,000 |
| CSRF on sensitive action | Medium | $200-1,000 |
| Open Redirect | Low/Medium | $50-500 |

---

## 🔴 Bug Bounty Platforms

| Platform | URL | Notes |
|----------|-----|-------|
| HackerOne | hackerone.com | Largest platform |
| Bugcrowd | bugcrowd.com | Good for beginners |
| Intigriti | intigriti.com | European focus |
| YesWeHack | yeswehack.com | European focus |
| Synack | synack.com | Invite-only, consistent work |
| Open Bug Bounty | openbugbounty.org | XSS-focused, easy start |

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
□ Test 3-5 endpoints deeply
□ Document anything suspicious

EVENING (30 min):
□ Write up any findings
□ Review notes from the day
□ Plan tomorrow's targets
□ Read one blog post about new techniques
```

---

## 🧠 If You're Stuck

1. **Can't find bugs**: Narrow your focus. Test ONE vuln type on ONE target for a week.
2. **Reports getting closed as N/A**: Improve report quality. Read accepted reports on HackerOne Hacktivity.
3. **Duplicates**: Focus on new features, recently acquired companies, or less popular programs.
4. **Don't know what to test**: Pick one OWASP Top 10 category and become an expert.
5. **Discouraged**: Every successful hunter has had months of nothing. Persistence wins.

---

## 🧠 Section 08 Complete — Self-Check

Before closing the handbook, verify you can:

- [ ] Evaluate a bug bounty program (scope, payouts, response time)
- [ ] Run a full recon pipeline (subdomain → live → tech → scan)
- [ ] Test for IDOR, XSS, SQLi, and SSRF manually
- [ ] Write a professional report that anyone can reproduce
- [ ] Set up continuous monitoring for a target
- [ ] Name the top 5 bug types by average bounty value
