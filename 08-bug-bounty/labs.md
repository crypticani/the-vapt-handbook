# 08 — Bug Bounty: Labs

> These labs put your entire VAPT skillset together in a bug bounty context.

---

## 🧪 Lab 1: Full Recon Pipeline

### Objective
Build and execute a complete recon pipeline against a bug bounty target.

### Steps
```bash
# 1. Pick a target from HackerOne
# Go to: https://hackerone.com/bug-bounty-programs
# Choose one with wide scope (*.target.com)

# 2. Run your recon pipeline
# Use the bb_recon.sh script from notes.md
./bb_recon.sh target.com

# 3. Analyze results
# How many subdomains? How many live? Any interesting ones?
# What technologies are in use?
# Any quick wins from nuclei?

# 4. Create an attack surface map
# Document top 5 priority targets with attack vectors
```

### ✅ Success Criteria
- [ ] 50+ subdomains discovered
- [ ] Live hosts identified with technologies
- [ ] Attack surface map created
- [ ] Top 5 priority targets documented

---

## 🧪 Lab 2: IDOR Hunting

### Objective
Find IDOR vulnerabilities in a web application.

### Steps
```bash
# 1. Create 2 accounts in Juice Shop (or target app)
# Account A: attacker@test.com
# Account B: victim@test.com

# 2. As Account A, perform every authenticated action:
# - View profile
# - View basket
# - View orders
# - Add/modify items

# 3. For each request with an ID:
# - Note the ID value
# - Switch to Account B's token
# - Request Account A's resource using Account B's auth

# 4. Example IDOR test on Juice Shop:
curl http://localhost:3000/rest/basket/1 \
  -H "Authorization: Bearer ACCOUNT_B_TOKEN"
# If you can see Account A's basket → IDOR!

# 5. Test every numeric ID and UUID in every endpoint
```

### ✅ Success Criteria
- [ ] Two accounts created
- [ ] All authenticated endpoints mapped
- [ ] At least 1 IDOR found
- [ ] Finding documented with steps to reproduce

---

## 🧪 Lab 3: Write a Bug Bounty Report

### Objective
Write a professional-quality bug bounty report for a vulnerability you've found.

### Steps
```markdown
# Use the report template from notes.md
# Write a report for the Juice Shop SQL injection:

1. Title: SQL Injection in Login Endpoint Allows Authentication Bypass
2. Summary: One paragraph describing the bug
3. Severity: Critical (CVSS 9.8)
4. Steps to Reproduce: Exact steps, anyone can follow
5. PoC: Include the request/response
6. Impact: Business-level description
7. Remediation: Specific technical fixes
8. References: OWASP, CWE links
```

### ✅ Success Criteria
- [ ] Report follows the template
- [ ] Steps are reproducible by someone else
- [ ] Impact is clearly stated in business terms
- [ ] Remediation is specific and actionable

---

## 🧪 Lab 4: Build a Monitoring System

### Objective
Set up automated monitoring for new attack surface on a target.

### Steps
```bash
# 1. Create a monitoring script
# - Daily subdomain check
# - Alert on new subdomains
# - Auto-scan new subdomains

# 2. Set up cron job
# crontab -e
# 0 8 * * * /path/to/monitor.sh target.com

# 3. Configure notifications (Telegram bot)
# - Create bot via @BotFather
# - Get chat_id
# - Configure notify tool

# 4. Let it run for a week
# Review alerts and investigate any new subdomains
```

### ✅ Success Criteria
- [ ] Monitoring script runs daily
- [ ] Notifications configured and working
- [ ] At least 1 week of monitoring data

---

## 🧪 Lab 5: Compete in a Bug Bounty Program

### Objective
Submit your first bug bounty report to a real program.

### Steps
```
1. Sign up on HackerOne
2. Pick a VDP (Vulnerability Disclosure Program) — no bounty pressure
3. Run your recon pipeline
4. Spend 4+ hours testing
5. If you find a bug → Write a professional report → Submit
6. If not → Document what you tested and move to next target
```

### ✅ Success Criteria
- [ ] HackerOne account created
- [ ] Program selected and policy read
- [ ] Recon completed
- [ ] At least 4 hours of testing
- [ ] Report submitted (or test documentation completed)

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: Full recon pipeline executed
- [ ] Lab 2: IDOR hunting practiced
- [ ] Lab 3: Professional report written
- [ ] Lab 4: Monitoring system operational
- [ ] Lab 5: First real program attempted
