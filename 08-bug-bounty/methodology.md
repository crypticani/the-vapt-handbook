# 08 — Bug Bounty: Methodology

> Your operational playbook for bug bounty hunting.

> 📋 **What You Will Do In This Section**
> - [ ] Follow the 6-step Bug Bounty Kill Chain
> - [ ] Use vulnerability-specific hunting flows (IDOR, XSS, SSRF, Auth Bypass)
> - [ ] Focus on high-value bug types that pay the most
> - [ ] Complete the pre-hunting checklist before every session

---

## 🔴 The Bug Bounty Kill Chain

> 💡 **Why This Matters**
> This is your operational playbook for every hunting session. Without a kill chain, you'll spend 4 hours scanning and 0 hours testing. The kill chain ensures you spend 70% of your time on exploitation — where the money is.

```
1. TARGET SELECTION (15 min)
   ├── Choose program on HackerOne/Bugcrowd
   ├── Analyze scope, bounty table, response time
   └── Read program policy COMPLETELY

2. RECONNAISSANCE (1-2 hours)
   ├── Subdomain enumeration (subfinder, crt.sh, amass)
   ├── Technology fingerprinting (whatweb, httpx)
   ├── Content discovery (ffuf, waybackurls)
   ├── JS analysis for endpoints and secrets
   └── Attack surface mapping (top 5 priority targets)

3. VULNERABILITY HUNTING (2-4 hours)
   ├── Pick ONE vulnerability class to focus on
   ├── Test highest-priority endpoints first
   ├── Use Burp for manual testing
   ├── Automate repetitive checks (nuclei, ffuf)
   └── Chain vulnerabilities for higher impact

4. VALIDATION (30 min)
   ├── Confirm the bug is real (reproduce 3 times)
   ├── Determine full impact
   ├── Test reproduction steps from scratch
   └── Take screenshots and save requests

5. REPORTING (30-60 min)
   ├── Write clear, professional report
   ├── Include exact steps to reproduce
   ├── Attach evidence (screenshots, requests)
   ├── Suggest remediation
   └── Submit and wait patiently

6. FOLLOW-UP
   ├── Respond to triage questions promptly
   ├── Provide additional PoC if requested
   ├── Don't argue about severity
   └── Learn from the interaction
```

---

## 🔴 Vulnerability-Specific Hunting Flows

### IDOR Hunting Flow

> 💡 **Why This Matters**
> IDOR is the most consistently rewarded vulnerability class. This systematic flow catches IDORs that automated scanners cannot detect — because they require authenticated context switching.

```
1. Create 2 accounts (Account A and Account B)
2. As Account A, perform EVERY authenticated action
   └── Capture ALL requests in Burp
3. For EACH request with an ID parameter:
   ├── Note the ID value and endpoint
   ├── Replay with Account B's authentication token
   ├── Check: Can B access A's resources?
   └── Test: GET, POST, PUT, DELETE on the same endpoint
4. Try predictable IDs: sequential, timestamp-based, hashed
5. Try parameter pollution: ?id=1&id=2
6. Document EVERY successful access with exact reproduction steps
```

### XSS Hunting Flow

```
1. Input "UNIQUE_CANARY_XYZ" in EVERY parameter
2. Search ALL responses for your canary (including JSON, headers)
3. For each reflection point:
   ├── Identify the context (HTML body, attribute, JS, URL)
   ├── Try the simplest payload for that context
   ├── If filtered, try encoding/alternative payloads
   └── Test in multiple browsers
4. For stored XSS:
   └── Check where your input appears to OTHER users
5. For DOM XSS:
   └── Review JS source for document.location, innerHTML, eval
```

### SSRF Hunting Flow

```
1. Identify any parameter that accepts URLs:
   ├── webhooks, callbacks, image URLs
   ├── import functions, PDF generators
   ├── link previews, URL shorteners
   └── proxy/redirect parameters
2. Set up interactsh (out-of-band detection)
3. Replace URL with your interactsh URL
4. If callback received → SSRF confirmed
5. Escalate:
   ├── Internal: 127.0.0.1, 10.0.0.0/8, 172.16.0.0/12
   ├── Cloud metadata: 169.254.169.254
   ├── Protocols: file://, gopher://, dict://
   └── Demonstrate impact: credential theft, internal access
```

### Authentication Bypass Flow

```
1. Study the login flow completely (every request/response)
2. Test SQL injection in username/password
3. Test default credentials (admin/admin, etc.)
4. Test password reset flow:
   ├── Host header poisoning
   ├── Token predictability
   ├── Token reuse after password change
   └── Rate limiting on reset attempts
5. Test JWT manipulation:
   ├── none algorithm attack
   ├── Weak secret brute force
   ├── Claim tampering (role, userId)
   └── Key confusion (RS256 → HS256)
6. Test OAuth:
   ├── redirect_uri manipulation
   ├── State parameter missing/predictable
   └── Token leakage in referrer
7. Test 2FA bypass:
   ├── Rate limiting on code attempts
   ├── Backup code usage
   └── Race conditions
```

---

## 🔴 High-Value Bug Types (Focus Here)

| Bug Type | Average Bounty | Why It Pays |
|----------|---------------|-------------|
| Account Takeover (ATO) | $5,000+ | Direct user impact |
| RCE | $10,000+ | Full system compromise |
| SSRF → Metadata | $3,000+ | Cloud credential theft |
| IDOR with PII | $1,000-5,000 | Data breach risk |
| Authentication Bypass | $2,000-10,000 | Unauthorized access |
| Stored XSS (admin) | $1,000-5,000 | Persistent attack |
| SQL Injection | $3,000-10,000 | Data extraction |
| Business Logic | $1,000-5,000 | Unique, hard to automate |

---

## 📋 Pre-Hunting Checklist

```
BEFORE every hunting session:
□ Program policy read completely
□ Scope noted (in-scope AND out-of-scope)
□ Burp Suite running with browser proxy configured
□ interactsh server running (for OOB detection)
□ Recon pipeline completed (or resumed from previous)
□ Attack surface map reviewed
□ Two accounts created on target (for IDOR testing)
□ Note-taking document ready
□ VPN/clean IP if needed
□ Timer set (don't hunt for 8 hours straight → diminishing returns)
```

---

## 🔴 Post-Hunting Review

```
AFTER every hunting session:
□ Document what endpoints were tested
□ Document what vulnerability types were attempted
□ Note any "interesting but not yet exploitable" findings
□ Update attack surface map with new discoveries
□ Write up any findings (while memory is fresh!)
□ Plan which endpoints/vuln types to test next session
```

---

## 🧠 If You're Stuck

1. **No bugs after 4+ hours**: Switch to a different vulnerability type OR a different target. Don't keep testing the same thing.
2. **Finding IDORs but no impact**: Chain with other vulnerabilities. IDOR + data exposure = bounty.
3. **Reports keep getting N/A**: Read 10 accepted reports on HackerOne Hacktivity. Study the format, impact descriptions, and severity justifications.
4. **Too many targets, can't focus**: Pick ONE program, run recon once, hunt for ONE week. Depth > breadth.
5. **Burned out**: Take a break. Read writeups instead of hunting. Come back refreshed.

---

## 🧠 Handbook Complete — Final Self-Check

You've completed all 8 modules. Verify your readiness:

- [ ] **Fundamentals**: Can explain the VAPT lifecycle and set up a lab
- [ ] **Recon**: Can run a full recon pipeline (passive + active)
- [ ] **Web Exploitation**: Can test for SQLi, XSS, SSRF, IDOR manually
- [ ] **Privilege Escalation**: Can get root from a low-privilege shell
- [ ] **Post-Exploitation**: Can harvest credentials and pivot through networks
- [ ] **Cloud Security**: Can audit AWS IAM, K8s RBAC, and CI/CD pipelines
- [ ] **CTF**: Can approach and solve CTF challenges systematically
- [ ] **Bug Bounty**: Can select targets, hunt bugs, and write professional reports

**Next steps**: Pick a TryHackMe learning path OR a bug bounty program and START HUNTING. 🎯
