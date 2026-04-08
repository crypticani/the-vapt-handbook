# 08 — Bug Bounty: Labs

> These labs put your entire VAPT skillset together in a bug bounty context.

> 📋 **What You Will Do In This Section**
> - [ ] Execute a full recon pipeline against a real bug bounty target
> - [ ] Hunt for IDOR vulnerabilities using the two-account method
> - [ ] Write a professional bug bounty report
> - [ ] Build an automated monitoring system
> - [ ] Submit your first report to a real program

---

## 🧪 Lab 1: Full Recon Pipeline

### Objective
Build and execute a complete recon pipeline against a bug bounty target scope.

> 💡 **Why This Matters**
> Recon is step 1 of every bug bounty hunt. A thorough recon pipeline discovers 10x more attack surface than manual browsing. The hunter who finds the hidden staging server or forgotten API endpoint is the one who gets paid.

### Steps

```bash
# 1. Pick a target from HackerOne
# Go to: https://hackerone.com/bug-bounty-programs
# Choose one with wide scope (*.target.com)
# READ THE POLICY COMPLETELY before starting

# 2. Run your recon pipeline
# Save the bb_recon.sh script from notes.md, then:
chmod +x bb_recon.sh
./bb_recon.sh target.com

# 3. Analyze results
echo "=== Recon Analysis ==="
echo "Total subdomains: $(wc -l < bb_target.com_*/subs/all.txt)"
echo "Live hosts: $(wc -l < bb_target.com_*/tech/live.txt)"
echo "Interesting: $(wc -l < bb_target.com_*/tech/interesting.txt)"
echo "URLs with params: $(wc -l < bb_target.com_*/urls/params.txt)"

# 4. Create an attack surface map
# Document top 5 priority targets:
```

```markdown
## Attack Surface Map: target.com

| # | Subdomain | Technology | Priority | Attack Vector |
|---|-----------|-----------|---------|--------------|
| 1 | api.target.com | Node.js | HIGH | IDOR, auth bypass |
| 2 | staging.target.com | PHP/Laravel | HIGH | Dev config exposed |
| 3 | admin.target.com | React | HIGH | Auth bypass, priv esc |
| 4 | docs.target.com | Static | LOW | XSS in search |
| 5 | webhook.target.com | Python | MEDIUM | SSRF |
```

> 🔧 **If Stuck**
> - No subdomains found → Try cert transparency: `curl -s "https://crt.sh/?q=%25.TARGET&output=json" | jq -r '.[].name_value' | sort -u`
> - httpx too slow → Add rate limiting, focus on interesting subdomains first
> - nuclei finds nothing → Run with `-as` flag for automatic template selection

### ✅ Success Criteria
- [ ] 50+ subdomains discovered
- [ ] Live hosts identified with technologies
- [ ] Attack surface map created
- [ ] Top 5 priority targets documented

---

## 🧪 Lab 2: IDOR Hunting

### Objective
Find IDOR (Insecure Direct Object Reference) vulnerabilities using the two-account method.

> 💡 **Why This Matters**
> IDOR is one of the most common and highest-paying vulnerability classes. Average IDOR bounty: $1,000-5,000. The two-account method is systematic and catches IDORs that automated scanners miss entirely.

### Steps

```bash
# === Practice on Juice Shop ===

# 1. Create 2 accounts
# Account A: attacker@test.com
# Account B: victim@test.com

# 2. As Account A, capture the auth token
# Login → Get JWT from response

# 3. As Account A, perform authenticated actions:
#    - View basket
#    - View orders
#    - View profile
#    Note every endpoint with a numeric ID

# 4. Test IDOR on basket endpoint:
curl http://localhost:3000/rest/basket/1 \
  -H "Authorization: Bearer ACCOUNT_B_TOKEN"
# Can Account B see Account A's basket?

# 5. Test IDOR on user profile:
curl http://localhost:3000/api/Users/1 \
  -H "Authorization: Bearer ACCOUNT_B_TOKEN"
# Can Account B see another user's profile?
```

> ✅ **Expected Output**
> ```json
> {
>   "data": {
>     "id": 1,
>     "username": "admin",
>     "email": "admin@juice-sh.op"
>   }
> }
> ```
> If you receive another user's data with YOUR token → IDOR confirmed!

```bash
# 6. Test EVERY numeric ID in EVERY endpoint:
for id in $(seq 1 20); do
  echo "=== Testing ID: $id ==="
  curl -s http://localhost:3000/rest/basket/$id \
    -H "Authorization: Bearer YOUR_TOKEN" | jq '.data.id' 2>/dev/null
done
```

> 🔧 **If Stuck**
> - Juice Shop not running → `docker run -d -p 3000:3000 bkimminich/juice-shop`
> - Can't get JWT → Login via API: `curl -X POST http://localhost:3000/rest/user/login -H "Content-Type: application/json" -d '{"email":"admin@juice-sh.op","password":"admin123"}'`
> - IDOR not working → Try different HTTP methods (PUT, DELETE, PATCH) on the same endpoint

### ✅ Success Criteria
- [ ] Two accounts created
- [ ] All authenticated endpoints mapped
- [ ] At least 1 IDOR found
- [ ] Finding documented with steps to reproduce

---

## 🧪 Lab 3: Write a Bug Bounty Report

### Objective
Write a professional report for a vulnerability you've found (use Juice Shop findings).

> 💡 **Why This Matters**
> The same bug submitted with a clear report vs. a vague report can mean the difference between $5,000 and $500. Programs triage faster and pay more for well-structured reports.

### Steps

```markdown
# Use the report template from notes.md
# Write a report for the Juice Shop IDOR:

1. Title: IDOR in Basket API Allows Viewing Other Users' Shopping Baskets
2. Summary: One paragraph — what, where, impact
3. Severity: Medium (CVSS 6.5 — user data exposure)
4. Steps to Reproduce:
   a. Create two accounts (A and B)
   b. Login as Account A, note the basket ID
   c. Login as Account B, use Account B's token
   d. Request Account A's basket with Account B's token
   e. Observe: Account A's basket data is returned
5. PoC: Include the curl command and response
6. Impact: Attacker can view any user's basket data, PII exposure
7. Remediation: Verify basket ownership matches authenticated user ID
8. References: OWASP IDOR, CWE-639
```

> 🔧 **If Stuck**
> - Not sure about severity → Use the CVSS calculator at first.org/cvss/calculator
> - Report too short → Add screenshots of Burp requests/responses
> - Not sure about remediation → Suggest: "Implement server-side access control checks"

### ✅ Success Criteria
- [ ] Report follows the template
- [ ] Steps are reproducible by someone else
- [ ] Impact is clearly stated in business terms
- [ ] Remediation is specific and actionable

---

## 🧪 Lab 4: Build a Monitoring System

### Objective
Set up automated monitoring for new attack surface on a target.

> 💡 **Why This Matters**
> Bug bounty programs add new features and subdomains constantly. The hunter who discovers a new subdomain 30 minutes after it's deployed finds bugs before anyone else. Monitoring is passive income for bug bounty.

### Steps

```bash
# 1. Create the monitoring script
cat > monitor.sh << 'EOF'
#!/bin/bash
TARGET=$1
DATE=$(date +%Y%m%d)
BASE="$HOME/bb/$TARGET"
mkdir -p $BASE/daily

# Subdomain discovery
subfinder -d $TARGET -silent | sort -u > $BASE/daily/subs_$DATE.txt

# Compare with previous
if [ -f $BASE/daily/subs_latest.txt ]; then
  comm -13 $BASE/daily/subs_latest.txt $BASE/daily/subs_$DATE.txt > $BASE/daily/new_$DATE.txt
  NEW_COUNT=$(wc -l < $BASE/daily/new_$DATE.txt)
  if [ $NEW_COUNT -gt 0 ]; then
    echo "[!] $NEW_COUNT new subdomains for $TARGET"
    cat $BASE/daily/new_$DATE.txt
    cat $BASE/daily/new_$DATE.txt | httpx -silent | tee -a $BASE/daily/new_live_$DATE.txt
  fi
fi

cp $BASE/daily/subs_$DATE.txt $BASE/daily/subs_latest.txt
EOF
chmod +x monitor.sh

# 2. Set up cron job
echo "Add to crontab (crontab -e):"
echo "0 8 * * * /path/to/monitor.sh target.com"

# 3. Let it run for a week
# Review $HOME/bb/target.com/daily/ for new discoveries
```

### ✅ Success Criteria
- [ ] Monitoring script created and tested
- [ ] Cron job configured (or manual daily run)
- [ ] At least 3 days of monitoring data collected

---

## 🧪 Lab 5: First Real Bug Bounty Submission

### Objective
Submit your first bug bounty report to a real program.

> 💡 **Why This Matters**
> Everything before this was practice. This lab puts it all together on a real target with real consequences and real payouts. Start with a VDP (no bounty) to build confidence and reputation.

### Steps

```
1. Sign up on HackerOne (hackerone.com)
2. Pick a VDP (Vulnerability Disclosure Program) — no bounty pressure
3. Read the ENTIRE program policy
4. Run your recon pipeline
5. Spend 4+ hours testing:
   ├── Focus on ONE vulnerability type
   ├── Test manually with Burp
   ├── Check mobile API endpoints
   └── Look for IDOR on every authenticated endpoint
6. If you find a bug:
   ├── Verify it's real (reproduce 3 times)
   ├── Write a professional report
   ├── Submit and wait patiently
7. If you don't find a bug:
   ├── Document what you tested
   ├── Move to next target
   └── Come back to this target after new features are deployed
```

> 🔧 **If Stuck**
> - Can't find bugs → Focus on less popular programs with wide scope
> - Unsure if it's a real bug → Check Hacktivity for similar reports
> - Afraid to submit → Better to submit and learn from triage than never submit
> - Report rejected → Read the triage feedback carefully. Improve and resubmit.

### ✅ Success Criteria
- [ ] HackerOne account created
- [ ] Program selected and policy read
- [ ] Recon completed
- [ ] At least 4 hours of testing
- [ ] Report submitted (or detailed test documentation completed)

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: Full recon pipeline executed
- [ ] Lab 2: IDOR hunting with two-account method
- [ ] Lab 3: Professional report written
- [ ] Lab 4: Monitoring system operational
- [ ] Lab 5: First real program attempted

---

## 🧠 If You're Stuck

1. **Recon finds nothing**: Try a different target. Programs with wide scope (*.target.com) are best.
2. **Can't find IDORs**: Not every app has them. Try XSS or SSRF instead.
3. **Report writing difficult**: Read 10 accepted reports on HackerOne Hacktivity for inspiration.
4. **Monitoring detects nothing new**: That's normal. Some domains rarely change. Monitor 5+ targets.
5. **First submission nerve-wracking**: Start with VDPs. No money pressure, but real triage experience.
