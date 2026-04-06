# 02 — Recon & Enumeration: Methodology

> This is your step-by-step recon methodology. Follow this for every engagement.

> 📋 **What You Will Do In This Section**
> - [ ] Define scope and boundaries before scanning
> - [ ] Execute a complete 4-phase recon workflow
> - [ ] Build a passive recon report with technology stack and OSINT findings
> - [ ] Validate subdomains and scan for live services
> - [ ] Create a prioritized attack surface map

---

## 🔴 The Recon Framework

> 💡 **Why This Matters**
> Without a structured methodology, you'll miss things. This framework ensures you check every data source, scan every port, and map every entry point — in a consistent, repeatable order. Real pentesters follow this for EVERY engagement, not just the first time.

```
┌─────────────────────────────────────────────────┐
│               RECONNAISSANCE                     │
│                                                  │
│  Phase 1: SCOPE DEFINITION                      │
│  ├── Define targets                              │
│  ├── Set boundaries                              │
│  └── Choose approach                             │
│                                                  │
│  Phase 2: PASSIVE RECON                         │
│  ├── Domain intelligence                         │
│  ├── OSINT gathering                             │
│  ├── Technology detection                        │
│  └── Historical analysis                         │
│                                                  │
│  Phase 3: ACTIVE RECON                          │
│  ├── Subdomain enumeration                       │
│  ├── Port scanning                               │
│  ├── Service identification                      │
│  └── Web content discovery                       │
│                                                  │
│  Phase 4: ATTACK SURFACE MAPPING                │
│  ├── Entry points catalog                        │
│  ├── Technology → CVE mapping                    │
│  ├── Priority ranking                            │
│  └── Attack plan creation                        │
└─────────────────────────────────────────────────┘
```

---

## 🔴 Phase 1: Scope Definition (15 minutes)

> 💡 **Why This Matters**
> Scope is your legal protection. Testing something outside scope can end your career and land you in court. Define it clearly BEFORE running a single command. This also prevents wasted effort — there's no point scanning a CDN you can't bypass.

### Inputs
- Target domain(s) or IP range(s)
- Rules of Engagement document
- Any exclusions or restrictions

### Process

```markdown
## Scope Document

### Target: example.com

### What I'm testing:
- [ ] Main domain: example.com
- [ ] All subdomains: *.example.com
- [ ] IP range: 104.26.10.0/24
- [ ] API: api.example.com

### What I'm NOT testing:
- Third-party services (Stripe, Intercom, etc.)
- Partner/vendor domains
- Specific IPs: 104.26.10.5 (shared hosting — other clients)

### Approach:
- [ ] Black box (no credentials, no insider info)
- [ ] Gray box (limited credentials, some documentation)
- [ ] White box (full access, source code, documentation)
```

#### 🧪 Try It Now — Scope Your Juice Shop Engagement

```markdown
## My Scope Document

### Target: Juice Shop Lab
- URL: http://localhost:3000
- Scope: All endpoints on localhost:3000
- Authorization: Self-hosted lab (full authorization)
- Approach: Black box (no insider knowledge)
- Exclusions: None (it's our lab)
- Time: Unlimited
```

> ✅ **Expected Output**
> A completed scope document. On real engagements, this would be signed by the client and include legal language.

---

## 🔴 Phase 2: Passive Reconnaissance (1-2 hours)

### Step 2.1: Domain Intelligence

```bash
# DNS Records (complete)
dig target.com A AAAA MX NS TXT SOA CNAME +noall +answer

# WHOIS
whois target.com | grep -iE "registrant|admin|tech|email|created|updated|expires|name.server"

# Reverse WHOIS (find other domains owned by same entity)
# Use: https://viewdns.info/reversewhois/

# ASN lookup (find all IP ranges owned by the org)
# Use: https://bgp.he.net/
whois -h whois.radb.net -- '-i origin AS12345'
```

#### 🧪 Try It Now — Domain Intelligence

```bash
TARGET="hackerone.com"
echo "=== WHOIS Summary ==="
whois $TARGET 2>/dev/null | grep -iE "registrant|created|updated|expires|name.server" | head -10

echo ""
echo "=== DNS Summary ==="
echo "A:  $(dig +short $TARGET A | head -2 | tr '\n' ', ')"
echo "MX: $(dig +short $TARGET MX | head -2 | tr '\n' ', ')"
echo "NS: $(dig +short $TARGET NS | head -2 | tr '\n' ', ')"
```

> ✅ **Expected Output**
> ```
> === WHOIS Summary ===
> Creation Date: 2011-...
> Updated Date: 2024-...
> Name Server: ns1.p16.dynect.net
>
> === DNS Summary ===
> A:  104.16.100.52, 104.16.99.52,
> MX: 1 aspmx.l.google.com., 5 alt1.aspmx.l.google.com.,
> NS: ns1.p16.dynect.net., ns2.p16.dynect.net.,
> ```

> 🔧 **If Stuck**
> - `whois: command not found` → `sudo apt install whois -y`
> - WHOIS returns "REDACTED" → Many domains use WHOIS privacy now. Check https://viewdns.info for historical WHOIS.

### Step 2.2: Subdomain Discovery (Passive Only)

```bash
# Certificate Transparency
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u > subs_crtsh.txt

# subfinder (uses multiple APIs)
subfinder -d target.com -silent > subs_subfinder.txt

# SecurityTrails (API)
# https://securitytrails.com/domain/target.com/dns

# VirusTotal
# https://www.virustotal.com/gui/domain/target.com/relations

# Combine all
cat subs_*.txt | sort -u > all_subs_passive.txt
```

### Step 2.3: Technology Detection

```bash
# Quick fingerprint
whatweb -q target.com
curl -sI target.com | grep -iE "^(server|x-powered|set-cookie|x-frame)"

# Browser: Install Wappalyzer extension and visit target
```

### Step 2.4: OSINT Deep Dive

```bash
# Google dorking (dedicate 30 minutes to this)
# Run each query and document interesting results:
# site:target.com filetype:sql|bak|log|env|config
# site:target.com inurl:admin|login|dashboard|portal
# site:target.com intitle:"index of"
# site:*.target.com -www
# "target.com" site:github.com password|token|secret
# "target.com" site:pastebin.com

# Wayback Machine
echo target.com | waybackurls > wayback.txt
cat wayback.txt | grep -iE '\.(bak|sql|zip|env|log|config|xml|json)$'
cat wayback.txt | grep '?' | sort -u > params.txt

# Shodan
shodan host $(dig +short target.com | head -1)
```

### Step 2.5: GitHub/Source Leak Hunt

```bash
# Search on GitHub:
# "target.com" password
# "target.com" api_key secret
# "@target.com" (employee accounts)
# org:target-company filename:.env
# org:target-company filename:wp-config

# Check for exposed .git
curl -s https://target.com/.git/HEAD
# If "ref: refs/heads/main" → .git exposed, use git-dumper

# Check for exposed .env
curl -s https://target.com/.env
```

### Deliverable: Passive Recon Report

```markdown
## Passive Recon Summary

### Technology Stack
- Server: Nginx 1.18
- Framework: Node.js + Express
- Frontend: React
- CDN: Cloudflare
- Email: Google Workspace (via SPF)

### Subdomains (Passive)
Total: 47 unique subdomains
Interesting:
- staging.target.com
- api-old.target.com
- jenkins.target.com

### Potential Secrets Found
- GitHub: API key found in commit abc123 by @employee
- Wayback: /config/database.yml exposed (now 404)
- Pastebin: Employee credentials posted 6 months ago

### Attack Vectors Identified
1. staging.target.com → likely weaker security
2. jenkins.target.com → default credentials?
3. api-old.target.com → deprecated API, missing patches?
4. DMARC policy = none → email spoofing possible
```

> ✅ **Expected Output**
> A completed report like above. On real engagements, this report goes into your final deliverable and shows the client how much information is publicly available about them.

---

## 🔴 Phase 3: Active Reconnaissance (2-4 hours)

> 💡 **Why This Matters**
> Active recon directly interacts with the target — port scanning, brute forcing directories, probing services. It's more revealing than passive recon but leaves traces in logs. Execute this phase after passive recon has narrowed your focus.

### Step 3.1: Subdomain Validation & Expansion

```bash
# Active brute force
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -t 50 > subs_brute.txt

# Combine with passive results
cat all_subs_passive.txt subs_brute.txt | sort -u > all_subs.txt

# Resolve and find live hosts
cat all_subs.txt | httpx -silent -status-code -title -tech-detect -follow-redirects > live_hosts.txt
```

### Step 3.2: Port Scanning

```bash
# Extract unique IPs from resolved subdomains
cat all_subs.txt | while read sub; do
  dig +short $sub | grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'
done | sort -u > target_ips.txt

# Quick scan (top 1000 ports)
nmap -sS -T4 -iL target_ips.txt -oA nmap_quick

# Full port scan on interesting hosts
nmap -sS -p- --min-rate 5000 <interesting_IP> -oA nmap_full_<hostname>

# Deep service scan on all open ports
nmap -sV -sC -p <open_ports> <interesting_IP> -oA nmap_deep_<hostname>

# UDP scan
sudo nmap -sU --top-ports 50 -T4 <interesting_IP> -oA nmap_udp_<hostname>
```

#### 🧪 Try It Now — Active Recon on Juice Shop

```bash
echo "=== Port Scan ==="
nmap -sV -p 3000 localhost 2>/dev/null | grep "open"

echo ""
echo "=== Directory Discovery ==="
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302 -t 20 -s 2>/dev/null | head -10

echo ""
echo "=== Sensitive Files ==="
for path in .git/HEAD .env robots.txt sitemap.xml .well-known/security.txt package.json; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/$path")
  [ "$code" != "404" ] && echo "  ⚠️  $path → $code"
done
```

> ✅ **Expected Output**
> ```
> === Port Scan ===
> 3000/tcp open  http    Node.js Express framework
>
> === Directory Discovery ===
> assets
> ftp
> profile
> robots.txt
>
> === Sensitive Files ===
>   ⚠️  robots.txt → 200
>   ⚠️  package.json → 200
> ```

### Step 3.3: Web Content Discovery

```bash
# For each live web host:
for url in $(cat live_hosts.txt | awk '{print $1}'); do
  host=$(echo $url | sed 's|https\?://||' | sed 's|/.*||')
  echo "[*] Scanning $url"
  
  # Directory brute force
  ffuf -u "$url/FUZZ" \
    -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
    -mc 200,301,302,401,403 \
    -t 50 -rate 100 \
    -o "ffuf_${host}.json" -of json 2>/dev/null
    
  # Check for sensitive files
  for path in .git/HEAD .env robots.txt sitemap.xml .well-known/security.txt; do
    code=$(curl -s -o /dev/null -w "%{http_code}" "$url/$path")
    if [ "$code" != "404" ] && [ "$code" != "000" ]; then
      echo "  [!] $path → HTTP $code"
    fi
  done
done
```

### Step 3.4: API Discovery

```bash
# Check for API documentation
for url in $(cat live_hosts.txt | awk '{print $1}'); do
  for endpoint in /swagger.json /api-docs /swagger-ui.html /openapi.json /graphql /.well-known/openapi.json /v1/api-docs /v2/api-docs; do
    code=$(curl -s -o /dev/null -w "%{http_code}" "$url$endpoint")
    if [ "$code" == "200" ]; then
      echo "[!] API docs found: $url$endpoint"
    fi
  done
done

# GraphQL introspection
curl -s -X POST "$url/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name fields { name } } } }"}' | python3 -m json.tool
```

---

## 🔴 Phase 4: Attack Surface Mapping (30 minutes)

> 💡 **Why This Matters**
> This is your master document — the output of ALL your recon work, organized for exploitation. Without it, you'll forget findings, miss connections, and waste time retesting things. With it, you have a clear, prioritized plan of attack.

### Build The Master Document

```markdown
## Attack Surface Map

### Date: [DATE]
### Target: target.com
### Tester: [YOUR NAME]

---

### Infrastructure Overview
| Component | Details | Risk Level |
|-----------|---------|------------|
| Primary web | Nginx 1.18 + Express | Medium (known CVEs) |
| API server | Node.js REST API | High (multiple endpoints) |
| Database | MongoDB (error-based detection) | High (NoSQL injection) |
| CDN | Cloudflare | Bypass needed |
| CI/CD | Jenkins 2.x (exposed) | Critical |

### Subdomain Analysis
| Subdomain | Status | Technology | Priority | Notes |
|-----------|--------|------------|----------|-------|
| www.target.com | 200 | React SPA | Medium | Main app |
| api.target.com | 200 | Express API | HIGH | 15 endpoints found |
| staging.target.com | 200 | Same stack | CRITICAL | Debug mode, test creds? |
| admin.target.com | 403 | Unknown | HIGH | Admin panel exists |
| dev.target.com | NXDOMAIN | CNAME→Heroku | CRITICAL | Subdomain takeover! |
| jenkins.target.com | 200 | Jenkins 2.289 | CRITICAL | Default creds? |

### Entry Points (Prioritized)
| # | URL | Method | Auth? | Input | Risk | Attack Vector |
|---|-----|--------|-------|-------|------|---------------|
| 1 | /api/login | POST | No | JSON body | High | SQLi, brute force |
| 2 | /api/users/{id} | GET | JWT | Path param | High | IDOR |
| 3 | /api/upload | POST | JWT | Multipart | Critical | RCE via file upload |
| 4 | /api/search | GET | No | Query string | High | SQLi, XSS |
| 5 | /api/webhook | POST | JWT | JSON (URL) | High | SSRF |

### Quick Wins To Try First
1. Jenkins default credentials (admin:admin)
2. Subdomain takeover on dev.target.com
3. Staging environment — test credentials, debug mode
4. API IDOR on /api/users/{id}
5. SQLi on /api/search?q=

### Tech → CVE Opportunities
| Technology | Version | CVE | Impact |
|------------|---------|-----|--------|
| Nginx | 1.18.0 | CVE-2021-23017 | DNS resolver DoS |
| Jenkins | 2.289 | CVE-2021-XXXXX | RCE |
| Express | Unknown | Check for prototype pollution | DoS/RCE |

### Recommended Testing Order
1. 🔴 staging.target.com → Full app test (weakest target)
2. 🔴 jenkins.target.com → Default creds → RCE
3. 🔴 dev.target.com → Subdomain takeover
4. 🟡 api.target.com → IDOR, injection testing
5. 🟡 admin.target.com → 403 bypass
6. 🟢 www.target.com → Standard web app test
```

#### 🧪 Try It Now — Build Your Juice Shop Attack Surface Map

```markdown
## Attack Surface Map: Juice Shop

### Date: [TODAY]
### Target: http://localhost:3000

### Technology Stack
| Layer | Technology | Source |
|-------|-----------|--------|
| Backend | Express (Node.js) | X-Powered-By header |
| Database | SQLite | Error message |
| Frontend | Angular | JavaScript analysis |
| Auth | JWT (RS256) | Login response |
| CORS | * (allow all) | Access-Control-Allow-Origin |

### Entry Points (fill in from your Lab 2 results)
| # | Endpoint | Method | Risk | Attack Vector |
|---|----------|--------|------|---------------|
| 1 | /rest/user/login | POST | High | SQLi (confirmed!) |
| 2 | /rest/products/search?q= | GET | High | SQLi (confirmed!) |
| 3 | /api/Users/ | GET/POST | High | IDOR, mass assignment |
| 4 | /api/Feedbacks/ | POST | Medium | Stored XSS |
| 5 | /rest/basket/{id} | GET | High | IDOR |
| 6 | /ftp/ | GET | Medium | File disclosure |

### Quick Wins
1. SQLi on search → dump database
2. IDOR on baskets → access other users' data
3. /ftp/ → download exposed files
4. JWT manipulation → escalate to admin
```

> ✅ **Expected Output**
> A completed attack surface map. This is your roadmap for `03-web-exploitation/`. Every exploit you'll run is targeting an entry point from this map.

---

## 📋 Phase Completion Checklist

```
PHASE 1: SCOPE
□ Scope document created
□ Boundaries clearly defined
□ Testing approach determined

PHASE 2: PASSIVE RECON
□ DNS records enumerated (all types)
□ Subdomains discovered (crt.sh + subfinder + others)
□ Technology stack identified
□ Google dorking completed (15+ queries)
□ GitHub/source code search done
□ Wayback Machine analyzed
□ WHOIS information gathered
□ Shodan searched

PHASE 3: ACTIVE RECON
□ Subdomains validated and expanded (brute force)
□ Live hosts identified (httpx)
□ Full port scan completed (TCP + UDP)
□ Service versions detected (nmap -sV)
□ Directories/files enumerated (ffuf/gobuster)
□ API documentation searched
□ Virtual hosts checked
□ Sensitive files checked (25+ paths)

PHASE 4: ATTACK SURFACE MAP
□ Infrastructure overview documented
□ All subdomains categorized by priority
□ Entry points cataloged with attack vectors
□ Quick wins identified
□ Technologies mapped to CVEs
□ Testing order prioritized
□ Ready for exploitation phase → 03-web-exploitation/
```

---

## 🧠 If You're Stuck

1. **Don't know where to start Phase 2**: Start with DNS — `dig target.com A MX NS TXT +short`. Then crt.sh for subdomains. Then Google dorks. Follow the order above.
2. **Phase 3 scan taking too long**: Prioritize. Quick port scan first (`--top-ports 100`), then full scan on interesting hosts only.
3. **Can't build the attack surface map**: Use the template above. Fill in what you know, leave gaps for what you don't. The map improves as you test.
4. **Findings seem overwhelming**: Focus on the Quick Wins section. Start with the easiest, highest-impact targets. On most engagements, 80% of findings come from 20% of the attack surface.
5. **Not finding anything on passive recon**: Some targets have excellent security posture. That's OK — it means you focus more on application-level testing in Phase 3/4.
