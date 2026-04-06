# 02 — Recon & Enumeration: Core Concepts & Attacker Thinking

> Recon is where pentests are won or lost. 80% of your time should be here. The more you know about the target, the fewer exploits you need.

> 📋 **What You Will Do In This Section**
> - [ ] Understand the full recon kill chain (passive → active → mapping)
> - [ ] Perform DNS enumeration and extract intelligence from records
> - [ ] Discover subdomains through certificate transparency
> - [ ] Use Google dorking to find exposed files, admin panels, and leaked secrets
> - [ ] Run active subdomain enumeration and identify live hosts
> - [ ] Scan ports with Nmap and fingerprint services
> - [ ] Map the complete attack surface of a target into a prioritized document

---

## 🔴 The Recon Mindset

### Why Recon Matters

```
Bad pentester:  Finds target → Runs scanner → Reports whatever scanner finds
Good pentester: Finds target → Maps EVERYTHING → Identifies the ONE weakness that matters
```

> 💡 **Why This Matters**
> The goal of recon isn't to collect data — it's to build a **mental model** of the target's infrastructure, technology stack, and attack surface. Every piece of information narrows your search. A 10-minute recon phase that discovers a staging server with default credentials is worth more than 10 hours of blind SQLi testing.

### The Recon Kill Chain

```
PASSIVE RECON (no direct contact with target)
  │
  ├─→ Domain intelligence (WHOIS, DNS, certificates)
  ├─→ Technology fingerprinting (Wappalyzer, BuiltWith)
  ├─→ OSINT (Google dorks, GitHub leaks, social media)
  ├─→ Historical data (Wayback Machine, cached pages)
  └─→ Subdomain discovery (certificate transparency, passive DNS)
  
ACTIVE RECON (direct contact — may be logged)
  │
  ├─→ Port scanning (Nmap)
  ├─→ Service enumeration (version detection)
  ├─→ Directory/file brute forcing (gobuster, ffuf)
  ├─→ Web application crawling (Burp Spider)
  ├─→ Virtual host discovery
  └─→ Technology-specific enumeration (WordPress, APIs)
  
ATTACK SURFACE MAPPING (combining everything)
  │
  ├─→ Entry points matrix
  ├─→ Technology → CVE mapping
  ├─→ Trust relationship mapping
  └─→ Priority target identification
```

---

## 🔴 Passive Reconnaissance

> 💡 **Why This Matters**
> Passive recon leaves NO trace in the target's logs. You're using public data sources — DNS records, certificates, search engines, archived pages. The target has no idea you're looking at them. This is legal against ANY target and is always your first step.

### DNS Intelligence

DNS tells you everything about how an organization's infrastructure is built.

```bash
# Full DNS enumeration
dig target.com ANY +noall +answer

# Individual record types
dig target.com A          # IPv4 addresses
dig target.com AAAA       # IPv6 addresses  
dig target.com MX         # Mail servers → reveals email infrastructure
dig target.com NS         # Name servers → who manages DNS
dig target.com TXT        # SPF, DKIM, DMARC → email security posture
dig target.com CNAME      # Aliases → reveals CDN, hosting
dig target.com SOA        # Start of Authority → admin email

# Reverse DNS lookup
dig -x 104.26.10.78

# Zone transfer (if misconfigured — jackpot)
dig axfr @ns1.target.com target.com
# If this works, you get ALL DNS records — complete infrastructure map

# DNS over HTTPS (bypass DNS logging)
curl -s "https://1.1.1.1/dns-query?name=target.com&type=A" -H "Accept: application/dns-json" | jq
```

#### 🧪 Try It Now — DNS Enumeration Against a Public Target

```bash
# Let's enumerate a public target (hackerone.com is fair game — it's a bug bounty platform)
TARGET="hackerone.com"

echo "=== DNS Records for $TARGET ==="
for type in A MX NS TXT; do
  echo "--- $type ---"
  dig $TARGET $type +short
done

# Try a zone transfer (will almost certainly fail on a well-configured target)
echo "--- Zone Transfer ---"
dig axfr @$(dig $TARGET NS +short | head -1) $TARGET 2>&1 | head -5
```

> ✅ **Expected Output**
> ```
> === DNS Records for hackerone.com ===
> --- A ---
> 104.16.100.52
> 104.16.99.52
> --- MX ---
> 1 aspmx.l.google.com.
> 5 alt1.aspmx.l.google.com.
> ...
> --- NS ---
> ns1.p16.dynect.net.
> ns2.p16.dynect.net.
> ...
> --- TXT ---
> "v=spf1 ... ~all"
> ...
> --- Zone Transfer ---
> ; Transfer failed.
> ```
> **What you learned:**
> - `A` records → Two IPs, probably behind Cloudflare
> - `MX` records → Google Workspace (aspmx.l.google.com) → they use Gmail for email
> - `TXT` records → SPF record reveals authorized email senders
> - Zone transfer failed → DNS is properly configured (this would be a huge finding if it worked)

> 🔧 **If Stuck**
> - `dig: command not found` → Install: `sudo apt install dnsutils -y`
> - No output → Check internet connectivity. Try `dig google.com A +short`
> - Timeout → DNS might be blocking queries. Try a different resolver: `dig @8.8.8.8 hackerone.com A +short`

### 🧠 Attacker Thinking: Reading DNS Records

```
MX: mx.target.com → They run their own mail server (not GSuite/O365)
                     → Test for open relay, SMTP enumeration

TXT: "v=spf1 include:_spf.google.com ~all"
     → They use Google Workspace → phishing vector identified
     
TXT: "v=DMARC1; p=none"  
     → DMARC set to "none" → email spoofing is possible!

CNAME: www.target.com → d1234.cloudfront.net
       → They use CloudFront CDN → try to bypass CDN to find origin IP
       
CNAME: staging.target.com → target-staging.herokuapp.com
       → If Heroku app is deleted → subdomain takeover!
```

### Certificate Transparency

Every SSL certificate is logged publicly. This reveals subdomains the target may not want you to know about.

```bash
# Query crt.sh
curl -s "https://crt.sh/?q=%25.target.com&output=json" | \
  jq -r '.[].name_value' | \
  sed 's/\*\.//g' | \
  sort -u > subdomains.txt

# Alternative sources
# https://censys.io
# https://shodan.io
# https://securitytrails.com

# Combine with DNS resolution
cat subdomains.txt | while read sub; do
  ip=$(dig +short $sub 2>/dev/null | head -1)
  if [ -n "$ip" ]; then
    echo "$sub → $ip"
  fi
done
```

#### 🧪 Try It Now — Find Subdomains via Certificate Transparency

```bash
# Discover subdomains of a real bug bounty target
TARGET="hackerone.com"
curl -s "https://crt.sh/?q=%25.${TARGET}&output=json" | \
  jq -r '.[].name_value' 2>/dev/null | \
  sed 's/\*\.//g' | \
  sort -u > /tmp/crtsh_subs.txt

echo "Found $(wc -l < /tmp/crtsh_subs.txt) unique subdomains"
echo "--- Top 20 subdomains ---"
head -20 /tmp/crtsh_subs.txt
```

> ✅ **Expected Output**
> ```
> Found 30+ unique subdomains
> --- Top 20 subdomains ---
> api.hackerone.com
> docs.hackerone.com
> gslink.hackerone.com
> hackerone.com
> mta-sts.hackerone.com
> www.hackerone.com
> ...
> ```
> Each of these subdomains is a potential attack surface. `api.hackerone.com` is the API, `docs.hackerone.com` hosts documentation — and any dev/staging subdomains are high-priority targets.

> 🔧 **If Stuck**
> - `jq: command not found` → Install: `sudo apt install jq -y`
> - crt.sh returns empty → The site might be rate-limiting. Wait 30 seconds and retry, or visit `https://crt.sh/?q=%25.hackerone.com` in your browser.
> - JSON parse error → crt.sh sometimes returns HTML instead of JSON. Add `2>/dev/null` to suppress errors.

### Google Dorking (Advanced)

> 💡 **Why This Matters**
> Google has already crawled and indexed parts of your target that the target itself may have forgotten about. Exposed admin panels, backup files, leaked credentials, and debug pages are all findable through creative search queries — without touching the target directly.

```bash
# Find exposed admin panels
site:target.com inurl:admin OR inurl:login OR inurl:dashboard

# Find exposed files
site:target.com filetype:sql OR filetype:bak OR filetype:log OR filetype:env

# Find exposed configuration
site:target.com filetype:xml OR filetype:conf OR filetype:cfg

# Find exposed API documentation  
site:target.com inurl:swagger OR inurl:api-docs OR inurl:graphql

# Find error pages (reveal technology)
site:target.com "error" OR "exception" OR "stack trace" OR "debug"

# Find login pages
site:target.com inurl:"/login" OR inurl:"/signin" OR inurl:"/auth"

# Find exposed directories
site:target.com intitle:"Index of" OR intitle:"Directory listing"

# Find S3 buckets
site:s3.amazonaws.com "target"
site:target.com.s3.amazonaws.com

# Find credentials in code
site:github.com "target.com" password OR api_key OR secret OR token

# Find subdomains Google knows about
site:*.target.com -www

# Find cached/removed pages
cache:target.com/admin
```

#### 🧪 Try It Now — Google Dork Practice

Open Google in your browser and search for each of these (replace with a real bug bounty target):

```
1. site:hackerone.com filetype:pdf
2. site:*.hackerone.com -www
3. site:github.com "hackerone.com" api_key
```

> ✅ **Expected Output**
> - Query 1: PDF files hosted on the domain (policies, reports, whitepapers)
> - Query 2: Non-www subdomains Google has indexed
> - Query 3: Code on GitHub that references the domain with potential API keys

### GitHub & Source Code Leaks

```bash
# Search GitHub for secrets
# Manual: https://github.com/search?q=target.com+password&type=code

# Automated with trufflehog
trufflehog github --org=target-org

# Automated with gitleaks  
gitleaks detect --source=https://github.com/target-org/repo

# Search patterns:
# "target.com" + password
# "target.com" + api_key
# "target.com" + secret
# "target.com" + AWS_ACCESS_KEY
# "@target.com" (employee accounts)
# "target.com" + jdbc OR connection_string

# Check for .git exposure on the web
curl -s https://target.com/.git/HEAD
# If you get: ref: refs/heads/main → Git repo is exposed!
# Use git-dumper to download:
pip3 install git-dumper
git-dumper https://target.com/.git/ ./dumped_repo
```

#### 🧪 Try It Now — Check Juice Shop for Git Exposure

```bash
# Check if Juice Shop exposes version control
echo "--- .git exposure ---"
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/.git/HEAD
echo ""

echo "--- .env exposure ---"
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/.env
echo ""

echo "--- robots.txt ---"
curl -s http://localhost:3000/robots.txt

echo "--- ftp directory ---"
curl -s http://localhost:3000/ftp/ | head -20
```

> ✅ **Expected Output**
> ```
> --- .git exposure ---
> 404
> --- .env exposure ---
> 404
> --- robots.txt ---
> User-agent: *
> Disallow: /ftp
> --- ftp directory ---
> (HTML listing of files in the /ftp directory)
> ```
> **Key findings:**
> - `.git` and `.env` are not exposed (404) — good security
> - `robots.txt` reveals `/ftp` — a directory the site doesn't want search engines to index
> - `/ftp` is actually accessible and contains files! This is information disclosure.

### Wayback Machine

```bash
# Get historical URLs
curl -s "http://web.archive.org/cdx/search/cdx?url=*.target.com/*&output=text&fl=original&collapse=urlkey" | sort -u > wayback_urls.txt

# Filter for interesting files
cat wayback_urls.txt | grep -iE '\.(sql|bak|zip|tar|gz|env|config|xml|json|log|old|backup)$'

# Filter for parameters (potential injection points)
cat wayback_urls.txt | grep '?' | sort -u

# Tool: waybackurls
go install github.com/tomnomnom/waybackurls@latest
echo target.com | waybackurls > wayback_urls.txt
```

> 💡 **Why This Matters**
> Websites delete sensitive content but the Wayback Machine remembers. Admin panels that were removed, configuration files that were briefly exposed, and old API endpoints that still work — all findable through historical snapshots.

---

## 🔴 Active Reconnaissance

> 💡 **Why This Matters**
> Active recon involves directly interacting with the target. Unlike passive recon, the target CAN log your requests. But it reveals much more — live services, open ports, hidden directories, and real-time behavior. Always do passive first, then go active.

### Subdomain Enumeration (The Money Maker)

```bash
# Passive sources combined
subfinder -d target.com -o subfinder.txt
amass enum -passive -d target.com -o amass_passive.txt
assetfinder --subs-only target.com > assetfinder.txt

# Active brute force
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 50
# or
ffuf -u http://FUZZ.target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -mc 200,301,302,403

# Combine all sources and deduplicate
cat subfinder.txt amass_passive.txt assetfinder.txt | sort -u > all_subdomains.txt

# Resolve to find live hosts
cat all_subdomains.txt | httpx -silent -status-code -title -tech-detect -o live_hosts.txt

# Check each live host for interesting content
cat live_hosts.txt | while read line; do
  url=$(echo $line | awk '{print $1}')
  echo "=== $url ==="
  curl -sI "$url" | grep -iE "^(server|x-powered-by|x-frame|content-security|set-cookie)"
done
```

### 🧠 Attacker Thinking: Subdomain Goldmine

```
Found: staging.target.com
 → Likely has weaker security, debug mode, test accounts
 → May have the same database as production
 → Default credentials? admin:admin?

Found: api-v1.target.com (but current is api-v3)
 → Old API version → deprecated security → missing patches
 → Try old API endpoints on new server

Found: jenkins.target.com
 → CI/CD server exposed → default creds? 
 → If accessible: source code, secrets, build artifacts

Found: grafana.target.com
 → Monitoring dashboard → default admin:admin
 → Shows internal infrastructure, service names, IPs

Found: dev.target.com → 404
 → But CNAME points to Heroku → SUBDOMAIN TAKEOVER!
```

### Port Scanning With Nmap (Beyond Basics)

```bash
# Phase 1: Quick top 1000 ports
nmap -sV -sC -T4 -oA nmap_quick target.com

# Phase 2: Full TCP scan (all 65535)
nmap -sS -p- -T4 --min-rate 1000 -oA nmap_full target.com

# Phase 3: Deep scan on discovered ports
nmap -sV -sC -p 22,80,443,8080,3306 -A -oA nmap_deep target.com

# Phase 4: UDP scan (often forgotten, often gold)
nmap -sU --top-ports 50 -T4 -oA nmap_udp target.com

# Phase 5: Script scan for specific vulnerabilities
nmap --script vuln -p 80,443 target.com
```

#### 🧪 Try It Now — Nmap Scan Against Juice Shop

```bash
# Scan your local lab targets
echo "=== Quick Service Scan ==="
nmap -sV -p 80,3000 localhost 2>/dev/null

echo ""
echo "=== Full Port Scan (local fast) ==="
nmap -sS -p- --min-rate 10000 localhost -oA /tmp/nmap_lab 2>/dev/null | grep "open"
```

> ✅ **Expected Output**
> ```
> === Quick Service Scan ===
> PORT     STATE SERVICE VERSION
> 80/tcp   open  http    Apache httpd 2.4.x
> 3000/tcp open  http    Node.js Express framework
>
> === Full Port Scan (local fast) ===
> 80/tcp   open  http
> 3000/tcp open  ppp
> ```
> Port 80 is DVWA (Apache), port 3000 is Juice Shop (Express). On a real target, you'd see many more services — SSH, databases, admin panels.

> 🔧 **If Stuck**
> - `You requested a scan type which requires root privileges` → Run with `sudo`: `sudo nmap -sS ...`
> - Scan takes forever → For local targets, add `--min-rate 10000`. For remote targets, `--min-rate 1000` is safer.
> - All ports filtered → You may have a firewall blocking. For local Docker targets, scan `127.0.0.1`.

### Nmap Flags You Must Know

```bash
# Scan types
-sS    # SYN scan (stealth — doesn't complete TCP handshake)
-sT    # Connect scan (full handshake — more reliable, more visible)
-sU    # UDP scan
-sV    # Version detection (probe open ports for service/version)
-sC    # Default scripts (equivalent to --script=default)
-A     # Aggressive scan (-sV -sC -O --traceroute)

# Port specification
-p 80,443,8080    # Specific ports
-p 1-1024         # Port range
-p-               # ALL 65535 ports (essential for thorough testing)
--top-ports 100   # Most common 100 ports

# Timing
-T0    # Paranoid (very slow, IDS evasion)
-T1    # Sneaky (slow, IDS evasion)  
-T2    # Polite
-T3    # Normal (default)
-T4    # Aggressive (recommended for most scans)
-T5    # Insane (may cause errors)
--min-rate 1000   # Send at least 1000 packets/sec

# Output
-oN file.txt      # Normal output
-oX file.xml      # XML output
-oG file.gnmap     # Grepable output
-oA basename       # All formats (recommended — always use this)

# Evasion
-f             # Fragment packets
-D RND:5       # Decoy scan with 5 random IPs
--source-port 53  # Source port 53 (DNS) — may bypass firewalls
-sS -T2 -f    # Low-profile scan
```

### Nmap Script Engine (NSE)

```bash
# List all available scripts
ls /usr/share/nmap/scripts/ | wc -l  # ~600 scripts

# Useful script categories
nmap --script http-enum target.com          # Enumerate web directories
nmap --script http-headers target.com       # Get HTTP headers
nmap --script http-methods target.com       # Discover allowed HTTP methods
nmap --script ssl-enum-ciphers target.com   # TLS cipher analysis
nmap --script http-sql-injection target.com # Basic SQLi detection
nmap --script http-xssed target.com         # Known XSS references
nmap --script smb-vuln* target.com          # SMB vulnerabilities
nmap --script ftp-anon target.com           # Anonymous FTP access

# Run all vuln scripts
nmap --script vuln target.com

# Run specific script categories
nmap --script "default and safe" target.com
nmap --script "http-*" target.com
```

#### 🧪 Try It Now — NSE Scripts Against Juice Shop

```bash
# Run HTTP enumeration scripts against Juice Shop
nmap --script http-enum -p 3000 localhost 2>/dev/null

# Run HTTP methods detection
nmap --script http-methods -p 3000 localhost 2>/dev/null
```

> ✅ **Expected Output**
> ```
> PORT     STATE SERVICE
> 3000/tcp open  ppp
> | http-enum:
> |   /ftp/: Potentially interesting directory
> |   /robots.txt: Robots file
> |_  /assets/: Potentially interesting directory
> ```
> NSE automatically found the `/ftp` directory — which matches what we found in `robots.txt` earlier!

---

## 🔴 Directory & File Enumeration

### Strategy

```
1. Start with a small, common wordlist (fast scan)
2. If interesting patterns found, use targeted wordlists
3. Add file extensions based on detected technology
4. Check for backup files (.bak, .old, .swp)
5. Always check for version control exposure (.git, .svn)
```

> 💡 **Why This Matters**
> Developers deploy files they think are hidden — backup configs, test scripts, debug endpoints, old versions. Directory brute forcing finds them by systematically trying thousands of common paths. A single `.env.bak` file can contain database credentials.

### Gobuster

```bash
# Directory brute force
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -t 50 \
  -o gobuster_dirs.txt

# With file extensions
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -x php,asp,aspx,jsp,html,js,json,txt,bak,old,sql,zip,tar.gz,config \
  -t 50 \
  -o gobuster_files.txt

# With authentication
gobuster dir -u https://target.com \
  -w /usr/share/wordlists/dirb/common.txt \
  -c "session=YOUR_SESSION_COOKIE" \
  -t 50

# With custom status codes
gobuster dir -u https://target.com \
  -w /usr/share/wordlists/dirb/common.txt \
  -s 200,204,301,302,307,401,403 \
  -t 50

# Virtual host / subdomain brute force
gobuster vhost -u https://target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 50
```

### ffuf (Fuzz Faster U Fool)

```bash
# Directory brute force
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -mc 200,301,302,401,403 \
  -t 50

# With extensions
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -e .php,.asp,.txt,.bak,.html \
  -mc 200,301,302

# Parameter fuzzing (GET)
ffuf -u 'https://target.com/page?FUZZ=test' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200 \
  -fs 4242  # Filter by response size (remove "normal" size responses)

# Parameter fuzzing (POST)
ffuf -u https://target.com/login \
  -X POST \
  -d '{"FUZZ":"test"}' \
  -H "Content-Type: application/json" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# Virtual host discovery
ffuf -u https://target.com \
  -H "Host: FUZZ.target.com" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 1234  # Filter out default response size

# Recursive fuzzing
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -recursion \
  -recursion-depth 2

# Multiple fuzzing points
ffuf -u https://target.com/FUZZ1/FUZZ2 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt:FUZZ1 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-files.txt:FUZZ2
```

#### 🧪 Try It Now — Fuzz Juice Shop APIs

```bash
# Discover REST API endpoints
echo "=== /api/ endpoints ==="
ffuf -u http://localhost:3000/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,401,403,500 -t 20 -s 2>/dev/null | head -10

echo ""
echo "=== /rest/ endpoints ==="
ffuf -u http://localhost:3000/rest/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,401,403,500 -t 20 -s 2>/dev/null | head -10
```

> ✅ **Expected Output**
> ```
> === /api/ endpoints ===
> Challenges
> Feedbacks
> Products
> Quantitys
> Users
>
> === /rest/ endpoints ===
> basket
> captcha
> products
> user
> ```
> You just discovered the full Juice Shop API structure! Each of these is a potential attack endpoint (IDOR on Users, injection on Feedbacks, etc.).

---

## 🔴 Technology Fingerprinting

### What To Identify

```
Web Server:     Apache, Nginx, IIS, LiteSpeed
Framework:      Express, Django, Flask, Rails, Laravel, Spring, ASP.NET
CMS:            WordPress, Drupal, Joomla
CDN/WAF:        Cloudflare, AWS WAF, Akamai, Imperva
Database:       MySQL, PostgreSQL, MSSQL, MongoDB, Redis
Language:       PHP, Python, Node.js, Java, Ruby, .NET
Frontend:       React, Angular, Vue, jQuery
API:            REST, GraphQL, gRPC, SOAP
Auth:           JWT, OAuth, SAML, Session cookies
Cloud:          AWS, GCP, Azure, Heroku, Vercel
```

### How to Fingerprint

```bash
# Comprehensive fingerprinting
whatweb -v target.com

# HTTP headers analysis
curl -sI target.com | grep -iE "^(server|x-powered-by|x-generator|x-aspnet|set-cookie)"

# Common technology indicators:
# PHP:     Set-Cookie: PHPSESSID
# Java:    Set-Cookie: JSESSIONID
# .NET:    Set-Cookie: ASP.NET_SessionId, X-AspNet-Version
# Node.js: X-Powered-By: Express
# Django:  Set-Cookie: csrftoken
# Rails:   X-Request-Id, Set-Cookie: _session_id, X-Runtime

# WordPress detection
curl -s target.com | grep -i "wp-content\|wp-includes\|wordpress"
curl -s target.com/wp-login.php
curl -s target.com/wp-json/wp/v2/users  # User enumeration!

# Detect WAF
wafw00f target.com
# Or manually: send an obvious attack payload and see if you get blocked
curl "https://target.com/?id=1' OR 1=1-- -"
# If you get a generic block page → WAF detected
```

#### 🧪 Try It Now — Fingerprint Juice Shop Tech Stack

```bash
echo "=== Response Headers ==="
curl -sI http://localhost:3000 | grep -iE "^(server|x-powered|set-cookie|x-frame|x-content|access-control)"

echo ""
echo "=== whatweb ==="
whatweb -q http://localhost:3000 2>/dev/null

echo ""
echo "=== Error-based detection ==="
curl -s "http://localhost:3000/rest/products/search?q='" 2>/dev/null | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    err = data.get('error', {})
    print(f'Database: {\"SQLite\" if \"SQLITE\" in str(err) else \"Unknown\"}')
    print(f'ORM: {\"Sequelize\" if \"Sequelize\" in str(err) else \"Unknown\"}')
except: print('Could not parse error')
"
```

> ✅ **Expected Output**
> ```
> === Response Headers ===
> X-Powered-By: Express
> Access-Control-Allow-Origin: *
> X-Content-Type-Options: nosniff
> X-Frame-Options: SAMEORIGIN
>
> === whatweb ===
> http://localhost:3000 [200 OK] Express, HTML5, Script
>
> === Error-based detection ===
> Database: SQLite
> ORM: Sequelize
> ```
> **Complete tech stack identified:**
> - Backend: Express (Node.js)
> - Database: SQLite (via Sequelize ORM)
> - Frontend: Angular (from JavaScript analysis)
> - CORS: Wide open (`*`)

---

## 🔴 Attack Surface Mapping

### The Final Product

After recon, you should have this:

```markdown
## Attack Surface Map: target.com

### Infrastructure
| Component | Details | Vector |
|-----------|---------|--------|
| Web Server | Nginx 1.18.0 | CVE-2021-23017 (DNS resolver vuln) |
| Application | Node.js + Express | Prototype pollution, SSTI |
| Database | MongoDB (detected via error) | NoSQL injection |
| CDN | Cloudflare | Need to find origin IP |
| CI/CD | Jenkins (jenkins.target.com) | Default creds, RCE |

### Subdomains
| Subdomain | Status | Technology | Priority |
|-----------|--------|------------|----------|
| www.target.com | 200 | React + Express | Medium |
| api.target.com | 200 | Express API | HIGH |
| staging.target.com | 200 | Same as prod, debug on | CRITICAL |
| admin.target.com | 403 | Admin panel | HIGH |
| dev.target.com | NXDOMAIN | Heroku CNAME | TAKEOVER |

### Entry Points (Top Priority)
| # | Endpoint | Risk | Reason |
|---|----------|------|--------|
| 1 | POST /api/login | High | Auth endpoint — SQLi/brute force |
| 2 | GET /api/users/{id} | High | IDOR potential |
| 3 | POST /api/upload | Critical | File upload — RCE |
| 4 | POST /api/webhook | High | URL parameter — SSRF |
| 5 | GET /api/search?q= | High | Reflected input — SQLi/XSS |
```

#### 🧪 Try It Now — Build an Attack Surface Map for Juice Shop

```bash
echo "================================================================"
echo "  ATTACK SURFACE MAP: Juice Shop (http://localhost:3000)"
echo "================================================================"

echo ""
echo "--- Technology Stack ---"
echo "Backend:  Express (Node.js)"
echo "Database: SQLite via Sequelize"
echo "Frontend: Angular"
echo "Auth:     JWT (RS256)"
echo "CORS:     * (wide open)"

echo ""
echo "--- Key Endpoints ---"
for endpoint in "/rest/user/login" "/rest/products/search?q=test" "/api/Users/" "/api/Products/" "/api/Feedbacks/" "/ftp/" "/rest/basket/1"; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000$endpoint")
  SIZE=$(curl -s "http://localhost:3000$endpoint" | wc -c)
  echo "  $endpoint → $STATUS ($SIZE bytes)"
done
```

> ✅ **Expected Output**
> ```
> --- Key Endpoints ---
>   /rest/user/login → 500 (267 bytes)
>   /rest/products/search?q=test → 200 (12 bytes)
>   /api/Users/ → 200 (large — user list!)
>   /api/Products/ → 200 (large — products)
>   /api/Feedbacks/ → 200 (feedback data)
>   /ftp/ → 200 (file listing!)
>   /rest/basket/1 → 401 (needs auth — IDOR target)
> ```

---

## 🔴 Common Mistakes

1. **Not doing enough recon**: Jumping to exploitation with incomplete information
2. **Only scanning top 1000 ports**: Services often run on high ports (8080, 8443, 9090)
3. **Ignoring UDP**: SNMP (161), DNS (53), TFTP (69) are often open and vulnerable
4. **Missing subdomains**: The main domain is usually the most secure — subdomains are where the gold is
5. **Not checking historical data**: Wayback Machine reveals deleted secrets, old endpoints, removed auth
6. **Forgetting virtual hosts**: Multiple sites on one IP — only found through vhost enumeration
7. **Not fingerprinting the WAF**: Sending payloads through a WAF without knowing it's there wastes time

---

## 📋 Recon Checklist

```
PASSIVE:
□ DNS records (ALL types)
□ Certificate transparency (crt.sh)
□ Google dorking (files, admin panels, leaks)
□ GitHub/GitLab search
□ Wayback Machine
□ WHOIS information
□ Technology fingerprinting

ACTIVE:
□ Subdomain enumeration
□ Full port scan (TCP + UDP)
□ Service version detection
□ Directory/file brute forcing
□ Virtual host enumeration
□ Web application crawling
□ API documentation discovery

MAPPING:
□ Attack surface matrix created
□ Technologies mapped to CVEs
□ Entry points prioritized
□ Trust boundaries identified
□ Quick wins identified
```

---

## 🧠 If You're Stuck

1. **No subdomains found**: Try different wordlists, check certificate transparency, check DNS brute force. Some targets have very few subdomains — that's OK, focus on the main app.
2. **Everything behind a CDN**: Look for origin IP via historical DNS (SecurityTrails), email headers (send an email that triggers a reply), or misconfigured subdomains that resolve directly.
3. **WAF blocking scans**: Slow down (`-T2` in Nmap, `-rate 10` in ffuf), use custom User-Agents, try from different IPs. Or focus on application-level testing where the WAF passes requests through.
4. **No interesting ports**: Focus on web application testing — the attack surface is in the application logic, not the infrastructure. Modern apps run everything through port 80/443.
5. **Don't know where to start**: Start with the technology stack → search for known CVEs → then test custom functionality.

---

## 🧠 Section 02 Complete — Self-Check

Before moving to `03-web-exploitation/`, verify you can:

- [ ] Enumerate DNS records and extract intelligence from MX, TXT, and CNAME entries
- [ ] Find subdomains through certificate transparency (crt.sh)
- [ ] Run at least 5 useful Google dork queries for a target
- [ ] Perform a full Nmap port scan and interpret the service versions
- [ ] Discover hidden directories and API endpoints with ffuf
- [ ] Fingerprint a web application's technology stack from response headers and errors
- [ ] Build a prioritized attack surface map document
