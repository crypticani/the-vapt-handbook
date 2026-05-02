# 01 — Fundamentals: Methodology

Use this before touching a target. Keep it open during live testing.

## Quick Reference

HTTP attack checklist:
- [ ] Capture request in Burp or `curl -v`
- [ ] Change method, path, headers, cookies, and body
- [ ] Replay with one change at a time
- [ ] Compare status, length, headers, and body
- [ ] Save working evidence

Response analysis checklist:
- [ ] Status code changed
- [ ] Response length changed
- [ ] Error leaked stack, SQL, path, user, or framework
- [ ] Auth state changed
- [ ] New data, role, token, or redirect appeared

Input tracing checklist:
- [ ] Identify every parameter, header, cookie, and JSON field
- [ ] Find where input is reflected, stored, queried, or executed
- [ ] Test the same input in browser, Burp Repeater, and `curl`
- [ ] Map input to likely bug class: SQLi, XSS, IDOR, SSRF, CMDi, upload

## Quick Mode

- Confirm scope before traffic.
- Verify tools before testing.
- Capture normal behavior first.
- Test high-impact inputs first: login, search, upload, IDs, redirects, APIs.
- Document command, request, response, and conclusion.

---

## 🔴 Pre-Engagement Workflow

```
┌─────────────────────────────────────────────────────┐
│              PRE-ENGAGEMENT METHODOLOGY              │
│                                                      │
│  PHASE 1: AUTHORIZATION (Before you touch anything)  │
│     ├── Written scope received                       │
│     ├── Rules of engagement defined                  │
│     ├── Emergency contacts documented                │
│     ├── Testing window confirmed                     │
│     └── Legal authorization signed                   │
│                                                      │
│  PHASE 2: ENVIRONMENT SETUP (30 min)                │
│     ├── Tools installed and verified                 │
│     ├── Burp Suite configured with proxy             │
│     ├── Wordlists downloaded                         │
│     ├── Note-taking system ready                     │
│     └── VPN/clean IP configured (if needed)          │
│                                                      │
│  PHASE 3: TARGET PROFILING (1 hour)                 │
│     ├── Technology stack identified                  │
│     ├── All entry points mapped                      │
│     ├── Authentication model understood              │
│     ├── Business logic documented                    │
│     └── Baseline behavior captured                   │
│                                                      │
│  PHASE 4: TEST PLANNING (30 min)                    │
│     ├── Priority list of endpoints to test           │
│     ├── Vulnerability classes to focus on             │
│     ├── Time allocation per area                     │
│     └── Testing approach documented                  │
│                                                      │
│  PHASE 5: EXECUTE → Sections 02-05                  │
│  PHASE 6: REPORT  → Section 08                      │
└─────────────────────────────────────────────────────┘
```

---

## 🔴 Phase 1: Authorization

### Authorization Document Template

```markdown
# Penetration Test Authorization

## Client Information
- Company: ___________
- Contact: ___________
- Email: ___________
- Phone: ___________

## Scope
### In Scope:
- [ ] *.target.com (all subdomains)
- [ ] api.target.com (REST API)
- [ ] mobile app (iOS / Android)
- [ ] IP range: 10.0.0.0/24

### Out of Scope:
- [ ] payments.target.com
- [ ] Third-party services (Stripe, AWS console)
- [ ] Physical security
- [ ] Social engineering

## Rules of Engagement
- Testing window: Mon-Fri, 9am-6pm EST
- Denial of Service: NOT ALLOWED
- Data modification: ALLOWED (test accounts only)
- Social engineering: NOT ALLOWED
- Automated scanning: ALLOWED (rate limited to 100 req/sec)

## Emergency Contact
- Name: ___________
- Phone: ___________
- Email: ___________
- Escalation: "If system goes down, call immediately"

## Authorization
I, [Name], authorize [Pentester] to perform penetration testing
on the systems described above during the specified testing window.

Signature: ___________
Date: ___________
```

Use written authorization as your boundary. If it is not in scope, do not test it.

#### 🧪 Try It Now — Fill Out Authorization for Juice Shop

```markdown
# Your Practice Authorization (fill this in)

## Target: OWASP Juice Shop
- URL: http://localhost:3000
- Scope: All endpoints (self-hosted lab)
- Authorization: Self-authorized (local lab)
- Rules: All techniques permitted, no restrictions
- Testing window: Anytime
- Emergency contact: N/A (local Docker container)
```

> ✅ **Expected Output**
> A completed document that you can reference. This builds the habit — on a real engagement, you'll do this automatically.

### Quick Mode

- Scope answers what you can touch.
- Rules answer how hard you can test.
- Contacts answer who to call if something breaks.
- Testing window answers when traffic is allowed.

---

## 🔴 Phase 2: Environment Setup Verification

### Quick Mode

- Run the tool check.
- Fix missing tools before testing.
- Confirm lab or target returns an HTTP response.
- Keep Burp, terminal, browser, and notes open.

### Tool Verification Script

```bash
#!/bin/bash
# verify_tools.sh — Run this before every engagement

echo "=== VAPT Tool Verification ==="
echo ""

# Core tools
echo "[1/8] Burp Suite..."
which burpsuite > /dev/null 2>&1 && echo "  ✅ Burp Suite installed" || echo "  ❌ Burp Suite not found"

echo "[2/8] Nmap..."
nmap --version 2>/dev/null | head -1 && echo "  ✅" || echo "  ❌ nmap not found → sudo apt install nmap"

echo "[3/8] ffuf..."
ffuf -V 2>/dev/null | head -1 && echo "  ✅" || echo "  ❌ ffuf not found → sudo apt install ffuf"

echo "[4/8] SQLMap..."
sqlmap --version 2>/dev/null && echo "  ✅" || echo "  ❌ sqlmap not found → sudo apt install sqlmap"

echo "[5/8] curl..."
curl --version 2>/dev/null | head -1 && echo "  ✅" || echo "  ❌ curl not found → sudo apt install curl"

echo "[6/8] Python3..."
python3 --version 2>/dev/null && echo "  ✅" || echo "  ❌ python3 not found"

echo "[7/8] Wordlists..."
[ -d "/usr/share/seclists" ] && echo "  ✅ SecLists found at /usr/share/seclists" || echo "  ❌ SecLists not found → sudo apt install seclists"
[ -f "/usr/share/wordlists/rockyou.txt" ] && echo "  ✅ rockyou.txt found" || echo "  ⚠️  rockyou.txt not found (may need to decompress or download)"

echo "[8/8] Docker..."
docker --version 2>/dev/null && echo "  ✅" || echo "  ❌ Docker not found"

echo ""
echo "=== Lab Targets ==="
echo "Juice Shop: $(curl -s -o /dev/null -w '%{http_code}' http://localhost:3000 2>/dev/null || echo 'DOWN')"
echo "DVWA:       $(curl -s -o /dev/null -w '%{http_code}' http://localhost 2>/dev/null || echo 'DOWN')"
```

> ✅ **Expected Output**
> ```
> === VAPT Tool Verification ===
>
> [1/8] Burp Suite...
>   ✅ Burp Suite installed
> [2/8] Nmap...
>   Nmap version 7.92 ( https://nmap.org )
>   ✅
> ...
> === Lab Targets ===
> Juice Shop: 200
> DVWA:       302
> ```
> All items should show ✅. Fix any ❌ items before proceeding.

> 🔧 **If Stuck**
> - Multiple ❌ items → You're not on Kali Linux. Install each tool individually with `sudo apt install TOOL -y`
> - Docker not running → `sudo systemctl start docker`
> - Lab targets DOWN → `docker start juiceshop dvwa`

---

## 🔴 Phase 3: Target Profiling

### Quick Mode

- Headers identify stack.
- Status codes identify control points.
- Response sizes identify anomalies.
- Entry points become your test queue.

### Step 1: Technology Fingerprinting

```bash
# Run this against your target to determine the tech stack
TARGET="http://localhost:3000"

echo "=== Technology Fingerprint: $TARGET ==="

# Response headers
echo -e "\n--- Response Headers ---"
curl -sI "$TARGET" | grep -iE "server|x-powered|x-aspnet|x-frame|content-security|strict-transport"

# Technology detection
echo -e "\n--- Technology Detection ---"
whatweb "$TARGET" 2>/dev/null || echo "(whatweb not installed — install with: sudo apt install whatweb)"

# JavaScript frameworks (from HTML source)
echo -e "\n--- Frontend Frameworks ---"
curl -s "$TARGET" | grep -oP '(angular|react|vue|jquery|backbone|ember)[^"]*\.js' | sort -u | head -10
```

> ✅ **Expected Output (Juice Shop)**
> ```
> === Technology Fingerprint: http://localhost:3000 ===
>
> --- Response Headers ---
> X-Powered-By: Express
> X-Frame-Options: SAMEORIGIN
> X-Content-Type-Options: nosniff
>
> --- Technology Detection ---
> http://localhost:3000 [200 OK] Express, HTML5, Script
>
> --- Frontend Frameworks ---
> (Angular framework detected)
> ```
> **What you now know**: Backend is Express (Node.js), frontend is Angular. Focus on Node.js-specific vulnerabilities (prototype pollution, NoSQL injection) and Angular template injection.

### Step 2: Capture Baseline Behavior

```bash
# Document "normal" responses for key endpoints
TARGET="http://localhost:3000"

echo "=== Baseline: Normal Responses ==="
for endpoint in "/" "/rest/products/search?q=test" "/rest/user/login" "/api/Products/"; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$TARGET$endpoint")
  SIZE=$(curl -s "$TARGET$endpoint" | wc -c)
  echo "  $endpoint → Status: $STATUS, Size: $SIZE bytes"
done
```

> ✅ **Expected Output**
> ```
> === Baseline: Normal Responses ===
>   / → Status: 200, Size: 6788 bytes
>   /rest/products/search?q=test → Status: 200, Size: 12 bytes
>   /rest/user/login → Status: 500, Size: 267 bytes
>   /api/Products/ → Status: 200, Size: 26941 bytes
> ```
> **Save these numbers.** When you inject payloads, any response that differs from these baseline sizes is potentially interesting.

> 💡 **Why This Matters**
> Without a baseline, you can't detect anomalies. "The response was 500 bytes" means nothing. "The response was 500 bytes, but the baseline is 200 bytes" means the payload changed something — and that's where you focus.

### Step 3: Map the Application

```bash
# Automated endpoint discovery
TARGET="http://localhost:3000"

echo "=== Endpoint Discovery ==="

# Method 1: Directory fuzzing
echo -e "\n--- Directory Fuzzing (top hits) ---"
ffuf -u "$TARGET/FUZZ" \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302 -t 20 -s 2>/dev/null | head -15

# Method 2: Check common sensitive files
echo -e "\n--- Sensitive Files Check ---"
for file in robots.txt .env .git/HEAD sitemap.xml crossdomain.xml security.txt; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$TARGET/$file")
  [ "$STATUS" != "404" ] && echo "  ⚠️  $file → $STATUS" || echo "  ✅ $file → 404 (not found)"
done
```

> ✅ **Expected Output**
> ```
> === Endpoint Discovery ===
>
> --- Directory Fuzzing (top hits) ---
> assets
> ftp
> profile
> robots.txt
>
> --- Sensitive Files Check ---
>   ⚠️  robots.txt → 200
>   ✅ .env → 404 (not found)
>   ✅ .git/HEAD → 404 (not found)
>   ...
> ```
> **Action items**: Check `/ftp` (might expose files) and `robots.txt` (reveals hidden paths).

---

## 🔴 Phase 4: Test Planning

### Testing Priority Matrix

After profiling, create a prioritized test plan:

```markdown
# Test Plan: Juice Shop (http://localhost:3000)

## Tech Stack
- Backend: Express (Node.js)
- Frontend: Angular
- Database: SQLite (revealed by error messages)
- Auth: JWT (RS256)

## Priority Endpoints (highest risk first)

| Priority | Endpoint | Test For | Why |
|----------|----------|----------|-----|
| 1 | /rest/user/login | SQLi, brute force | Auth bypass = full compromise |
| 2 | /rest/products/search?q= | SQLi (confirmed!) | Data extraction |
| 3 | /api/Feedbacks | Stored XSS, injection | User input stored and rendered |
| 4 | /rest/basket/{id} | IDOR | Access other users' data |
| 5 | /ftp | Sensitive file exposure | Discovered via fuzzing |
| 6 | /api/Users | Mass assignment, IDOR | User creation/enumeration |
| 7 | /profile | XSS, file upload | User-controlled content |

## Time Allocation
- Authentication testing: 1 hour
- Injection testing: 2 hours
- Access control (IDOR): 1 hour
- Business logic: 30 min
- Documentation: 30 min ongoing
```

> ✅ **Expected Output**
> A completed test plan document that guides your testing. You know exactly what to test, in what order, and why.

> 💡 **Why This Matters**
> Without a plan, you'll spend hours testing low-value endpoints while missing the login SQLi that gives instant admin access. Priority = impact × likelihood. Auth endpoints are always #1.

---

## 📋 Methodology Cheatsheet

```
BEFORE TESTING:
□ Authorization document signed
□ Scope documented (in/out)
□ Tools verified (run verify_tools.sh)
□ Lab targets running
□ Note-taking system open

PROFILING (1 hour):
□ Technology stack identified (whatweb, response headers)
□ Baseline responses captured (status codes + sizes)
□ Entry points mapped (ffuf, Burp spider)
□ Sensitive files checked (robots.txt, .env, .git)
□ Authentication model understood (JWT? cookies? API keys?)

TEST PLANNING (30 min):
□ Priority endpoints listed (highest impact first)
□ Vulnerability classes assigned per endpoint
□ Time allocated per test area
□ Documentation template ready

THEN: Proceed to 02-recon/ → 03-web-exploitation/ → ...
```

---

## 🧠 If You're Stuck

1. **Don't know where to start**: Run the target profiling commands above. The output will tell you what tech is running and what to test.
2. **Feeling overwhelmed by the scope**: Pick the TOP 3 endpoints from your test plan. Focus on those first.
3. **No findings after an hour**: Step back and re-read the error messages you collected. Developers hide secrets in error responses.
4. **Tools aren't working**: Run `verify_tools.sh` and fix the ❌ items first.
5. **Not sure if something is a vulnerability**: Ask yourself: "Can I do something the developers didn't intend?" If yes, document it as a finding.

---

## 🧠 Section 01 Complete — Self-Check

Before moving to `02-recon/`, verify you can:

- [ ] Configure Burp Suite proxy and intercept traffic
- [ ] Send custom HTTP requests with curl (POST, headers, cookies)
- [ ] Decode a JWT token and identify its claims
- [ ] Run a port scan with Nmap and interpret the output
- [ ] Discover hidden directories with ffuf
- [ ] Detect SQL injection with a single quote payload
- [ ] Explain the difference between detection and exploitation payloads
- [ ] Complete a pre-engagement authorization document
- [ ] Create a prioritized test plan for a target application

---

## Next: 02-Recon

Fundamentals apply directly in recon:

- HTTP basics help you read headers, redirects, cookies, and API behavior.
- Response analysis helps you spot technology leaks, auth boundaries, and hidden paths.
- Input tracing helps you decide which discovered endpoints are worth testing first.
- Baselines help you separate real findings from scanner noise.

Go to `02-recon/` next and use this loop:

```text
Discover target -> read responses -> map inputs -> prioritize attack surface
```

Start with `02-recon/methodology.md`.
