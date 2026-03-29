# 08 — Bug Bounty: Methodology

> Your operational playbook for bug bounty hunting.

---

## 🔴 The Bug Bounty Kill Chain

```
1. TARGET SELECTION (15 min)
   ├── Choose program on HackerOne/Bugcrowd
   ├── Analyze scope, bounty table, response time
   └── Read program policy completely

2. RECONNAISSANCE (1-2 hours)
   ├── Subdomain enumeration (subfinder, crt.sh, amass)
   ├── Technology fingerprinting (whatweb, httpx)
   ├── Content discovery (ffuf, waybackurls)
   ├── JS analysis for endpoints and secrets
   └── Attack surface mapping

3. VULNERABILITY HUNTING (2-4 hours)
   ├── Pick ONE vulnerability class to focus on
   ├── Test highest-priority endpoints first
   ├── Use Burp for manual testing, automate repetitive checks
   └── Chain vulnerabilities for higher impact

4. VALIDATION (30 min)
   ├── Confirm the bug is real (not false positive)
   ├── Determine full impact
   ├── Test reproduction steps
   └── Take screenshots and save requests

5. REPORTING (30-60 min)
   ├── Write clear, professional report
   ├── Include exact steps to reproduce
   ├── Attach evidence (screenshots, requests)
   ├── Suggest remediation
   └── Submit and wait

6. FOLLOW-UP
   ├── Respond to triage questions promptly
   ├── Provide additional PoC if requested
   ├── Don't argue about severity
   └── Learn from the interaction
```

---

## 🔴 Vulnerability-Specific Hunting Flows

### IDOR Hunting Flow
```
1. Create 2 accounts (Account A and Account B)
2. Perform every action as Account A → log all requests
3. Replay Account A's requests using Account B's token
4. For every request with an ID parameter:
   - Change the ID to Account A's ID while authenticated as Account B
   - Check if data leaks or actions succeed
5. Look for predictable IDs (sequential, UUIDs that are timestamp-based)
6. Test: GET, POST, PUT, DELETE on every resource
```

### XSS Hunting Flow
```
1. Input "UNIQUE_CANARY_XYZ" in every parameter
2. Search the response for your canary
3. For each reflection point:
   - Identify the context (HTML, attribute, JS, URL)
   - Try the simplest payload for that context
   - If filtered, try encoding/alternative payloads
   - Test in multiple browsers
4. For stored XSS: Check where your input is displayed to OTHER users
5. For DOM XSS: Review JavaScript source for document.location, innerHTML, eval
```

### SSRF Hunting Flow
```
1. Identify any parameter that accepts URLs
   - webhooks, callbacks, image URLs, import functions, PDF generators
2. Set up Burp Collaborator or interact.sh
3. Replace URL with your Collaborator URL
4. If callback received → SSRF confirmed
5. Try internal targets: 127.0.0.1, 169.254.169.254, internal services
6. Try protocol switching: file://, gopher://, dict://
7. Demonstrate impact: metadata access, internal service access
```

### Authentication Bypass Flow
```
1. Study the login flow completely
2. Test: SQL injection in username/password
3. Test: Default credentials
4. Test: Password reset flow for flaws
5. Test: JWT manipulation (none alg, weak secret, claim tampering)
6. Test: OAuth misconfigurations (redirect_uri manipulation)
7. Test: 2FA bypass (rate limiting, backup codes, race conditions)
8. Test: Session handling (fixation, insufficient expiry)
```

---

## 🔴 High-Value Bug Types (Focus Here)

| Bug Type | Average Bounty | Why |
|----------|---------------|-----|
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
□ Program policy read completely
□ Scope noted (in-scope and out-of-scope)
□ Tool environment ready (Burp, scripts, wordlists)
□ Recon pipeline run
□ Attack surface mapped
□ Note-taking ready
□ Collaborator/interact.sh server running
□ Two accounts created on target (for IDOR testing)
□ VPN/clean IP (some programs block known scanner IPs)
```
