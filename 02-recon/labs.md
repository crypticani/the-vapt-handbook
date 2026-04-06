# 02 — Recon & Enumeration: Labs

> These labs build your recon muscle memory. You'll practice the complete reconnaissance workflow against intentionally vulnerable targets.

> 📋 **What You Will Do In This Section**
> - [ ] Perform a full passive recon against a real public domain
> - [ ] Combine multiple subdomain enumeration tools and find live hosts
> - [ ] Master Nmap scanning with all scan types (TCP, UDP, NSE scripts)
> - [ ] Discover hidden directories, API endpoints, and sensitive files
> - [ ] Build a recon automation script you can reuse on engagements
> - [ ] Fingerprint a complete technology stack using multiple methods

---

## 🧪 Lab 1: Full Passive Recon Against a Real Domain

### Objective
Perform complete passive reconnaissance without touching the target directly. Use a public bug bounty program (e.g., `*.hackerone.com`).

> 💡 **Why This Matters**
> Passive recon is undetectable. You're using public data sources that anyone can access. On a real engagement, this phase gives you a comprehensive understanding of the target before they even know you're looking. This is also the ONLY recon you can legally do on targets you don't have authorization to test.

### Steps

**Step 1: DNS Enumeration**
```bash
TARGET="hackerone.com"

# All DNS record types
echo "=== DNS Enumeration: $TARGET ==="
for type in A AAAA MX NS TXT CNAME SOA; do
  echo "--- $type ---"
  dig $TARGET $type +short
  echo ""
done

# Zone transfer attempt
echo "--- Zone Transfer Attempt ---"
dig axfr @$(dig $TARGET NS +short | head -1) $TARGET 2>&1 | head -5
```

> ✅ **Expected Output**
> ```
> === DNS Enumeration: hackerone.com ===
> --- A ---
> 104.16.100.52
> 104.16.99.52
>
> --- MX ---
> 1 aspmx.l.google.com.
> 5 alt1.aspmx.l.google.com.
> 10 alt2.aspmx.l.google.com.
>
> --- NS ---
> ns1.p16.dynect.net.
> ns2.p16.dynect.net.
>
> --- TXT ---
> "v=spf1 ... ~all"
> "google-site-verification=..."
>
> --- Zone Transfer Attempt ---
> ; Transfer failed.
> ```
> **Intelligence gathered**:
> - Two A records → likely Cloudflare (104.16.x.x range)
> - MX → Google Workspace (Gmail)
> - Zone transfer failed → well-configured DNS

> 🔧 **If Stuck**
> - `dig: command not found` → `sudo apt install dnsutils -y`
> - No output → Check internet: `ping 8.8.8.8`. Try specifying resolver: `dig @8.8.8.8 hackerone.com A +short`

**Step 2: Certificate Transparency**
```bash
# Pull subdomains from crt.sh
curl -s "https://crt.sh/?q=%25.${TARGET}&output=json" | \
  jq -r '.[].name_value' | \
  sed 's/\*\.//g' | \
  sort -u > /tmp/crtsh_subs.txt

echo "Found $(wc -l < /tmp/crtsh_subs.txt) subdomains"
echo "--- First 20 ---"
head -20 /tmp/crtsh_subs.txt
```

> ✅ **Expected Output**
> ```
> Found 25+ subdomains
> --- First 20 ---
> api.hackerone.com
> docs.hackerone.com
> gslink.hackerone.com
> hackerone.com
> mta-sts.hackerone.com
> support.hackerone.com
> www.hackerone.com
> ...
> ```

> 🔧 **If Stuck**
> - `jq: command not found` → `sudo apt install jq -y`
> - Empty result → crt.sh might be rate-limiting. Wait 30 seconds and retry.
> - JSON parse error → Try visiting `https://crt.sh/?q=%25.hackerone.com` in your browser to verify the service is up.

**Step 3: Google Dorking**

Manually search in Google (not automatable — Google blocks automated queries):
```
1. site:hackerone.com filetype:pdf
2. site:hackerone.com inurl:admin
3. site:*.hackerone.com -www
4. site:github.com "hackerone.com" password
5. site:hackerone.com intitle:"index of"
```

> ✅ **Expected Output**
> - Query 1: PDF files (policies, reports) hosted on hackerone.com
> - Query 2: Any admin-related pages indexed by Google
> - Query 3: Non-www subdomains (reveals hidden services)
> - Query 4: GitHub code referencing hackerone.com with passwords
> - Query 5: Exposed directory listings (unlikely on a security company, but always check)

**Step 4: Wayback Machine**
```bash
# Using the Wayback Machine CDX API
curl -s "http://web.archive.org/cdx/search/cdx?url=*.${TARGET}/*&output=text&fl=original&collapse=urlkey" 2>/dev/null | head -30 > /tmp/wayback.txt

echo "Found $(wc -l < /tmp/wayback.txt) historical URLs"

# Look for interesting files in history
cat /tmp/wayback.txt | grep -iE '\.(env|bak|sql|zip|log|config|xml|json)$' | head -10
echo "---"

# Look for URLs with parameters (potential injection points)
cat /tmp/wayback.txt | grep '?' | sort -u | head -10
```

> ✅ **Expected Output**
> ```
> Found 30+ historical URLs
> (various URLs from web.archive.org)
> ```
> Even if files are now deleted (404), the Wayback Machine might have cached copies.

> 🔧 **If Stuck**
> - Wayback API returns nothing → Target might not have been archived. Try with a more popular domain.
> - `waybackurls` tool not installed → Use the direct CDX API as shown above, or install: `go install github.com/tomnomnom/waybackurls@latest`

**Step 5: Document Findings**

Create a report summarizing:
```markdown
## Passive Recon Report: hackerone.com

### Technology Stack
- CDN: Cloudflare (104.16.x.x)
- Email: Google Workspace (MX → aspmx.l.google.com)
- DNS: DynECT (enterprise DNS)

### Subdomains Found (passive)
- Total: XX unique
- Key: api, docs, support, www, mta-sts

### Historical URLs
- Total: XX from Wayback Machine
- Interesting: [list any .env, .bak, etc.]

### Attack Vectors Identified
1. [List potential vectors based on findings]
```

### ✅ Success Criteria
- [ ] 20+ subdomains discovered through passive means
- [ ] DNS records fully documented (A, MX, NS, TXT)
- [ ] At least 3 Google dork results analyzed
- [ ] Historical URLs reviewed for sensitive files
- [ ] Passive recon report document created

---

## 🧪 Lab 2: Active Subdomain Enumeration & Resolution

### Objective
Combine multiple tools to build a comprehensive subdomain list, then identify live hosts.

> 💡 **Why This Matters**
> No single tool finds everything. Subfinder uses APIs, crt.sh uses certificate logs, and DNS brute forcing finds subdomains no public source knows about. Combining multiple sources dramatically increases coverage — the subdomain a scanner misses could be the unpatched staging server with admin:admin credentials.

### Steps

**Step 1: Multi-source subdomain enumeration**
```bash
TARGET="hackerone.com"  # Use any HackerOne program target
mkdir -p /tmp/recon/$TARGET

# Source 1: subfinder
echo "[*] Running subfinder..."
subfinder -d $TARGET -silent > /tmp/recon/$TARGET/subs_subfinder.txt 2>/dev/null
echo "  subfinder: $(wc -l < /tmp/recon/$TARGET/subs_subfinder.txt) subdomains"

# Source 2: crt.sh
echo "[*] Querying crt.sh..."
curl -s "https://crt.sh/?q=%25.${TARGET}&output=json" | \
  jq -r '.[].name_value' 2>/dev/null | \
  sed 's/\*\.//g' | sort -u > /tmp/recon/$TARGET/subs_crtsh.txt
echo "  crt.sh: $(wc -l < /tmp/recon/$TARGET/subs_crtsh.txt) subdomains"

# Combine and deduplicate
cat /tmp/recon/$TARGET/subs_*.txt | sort -u > /tmp/recon/$TARGET/all_subs.txt
echo "[+] Total unique subdomains: $(wc -l < /tmp/recon/$TARGET/all_subs.txt)"
```

> ✅ **Expected Output**
> ```
> [*] Running subfinder...
>   subfinder: 15+ subdomains
> [*] Querying crt.sh...
>   crt.sh: 25+ subdomains
> [+] Total unique subdomains: 30+
> ```
> Notice how combining sources found more than either alone!

> 🔧 **If Stuck**
> - `subfinder: command not found` → Install: `go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest`
> - subfinder finds 0 → It needs API keys for many sources. Add them to `~/.config/subfinder/provider-config.yaml`. Even without keys, it still queries free sources.
> - Go not installed → `sudo apt install golang-go -y`

**Step 2: DNS resolution**
```bash
# Resolve all subdomains to IPs
echo "[*] Resolving subdomains..."
cat /tmp/recon/$TARGET/all_subs.txt | while read sub; do
  ip=$(dig +short $sub 2>/dev/null | grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | head -1)
  if [ -n "$ip" ]; then
    echo "$sub → $ip"
  fi
done | tee /tmp/recon/$TARGET/resolved.txt

echo "[+] Resolved: $(wc -l < /tmp/recon/$TARGET/resolved.txt) subdomains"
```

> ✅ **Expected Output**
> ```
> [*] Resolving subdomains...
> api.hackerone.com → 104.16.100.52
> docs.hackerone.com → 104.16.99.52
> www.hackerone.com → 104.16.100.52
> ...
> [+] Resolved: 10+ subdomains
> ```
> Notice multiple subdomains resolving to the same IP — they're behind the same infrastructure (likely Cloudflare).

**Step 3: Live host probing**
```bash
# Probe for live web servers
echo "[*] Probing for live hosts..."
cat /tmp/recon/$TARGET/all_subs.txt | httpx -silent \
  -status-code \
  -title \
  -tech-detect \
  -follow-redirects 2>/dev/null | tee /tmp/recon/$TARGET/live_hosts.txt

echo "[+] Live hosts: $(wc -l < /tmp/recon/$TARGET/live_hosts.txt)"
```

> ✅ **Expected Output**
> ```
> https://hackerone.com [200] [HackerOne | ...] [Cloudflare,React]
> https://api.hackerone.com [200] [HackerOne API] [Cloudflare]
> https://docs.hackerone.com [200] [HackerOne Docs] [Cloudflare]
> [+] Live hosts: 5+
> ```

> 🔧 **If Stuck**
> - `httpx: command not found` → Install: `go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest`
> - httpx hangs → Add `--timeout 5` to limit connection timeout
> - All hosts return the same response → They might be behind a CDN returning a default page. This is normal for Cloudflare-protected sites.

**Step 4: Categorize findings**
```bash
# Find interesting subdomains
echo "=== Interesting Subdomains ==="
cat /tmp/recon/$TARGET/live_hosts.txt | grep -iE "staging|dev|test|admin|internal|api|jenkins|gitlab|grafana|kibana" || echo "  No high-value subdomains found (target is well-maintained)"
```

### ✅ Success Criteria
- [ ] Used at least 2 different enumeration sources
- [ ] Successfully resolved subdomains to IPs
- [ ] Identified live hosts with their technologies
- [ ] Categorized findings by priority

---

## 🧪 Lab 3: Nmap Deep Dive

### Objective
Master Nmap scanning techniques against your local lab targets.

> 💡 **Why This Matters**
> Nmap is the most-used tool in pentesting. Every engagement starts with port scanning. Understanding the difference between scan types, knowing when to use scripts, and reading output fluently saves massive time. A full port scan finds the one weird service on port 9999 that no one remembered to secure.

### Steps

**Step 1: Discovery scan (quick)**
```bash
# Quick top 1000 ports against your lab
echo "=== Quick Scan ==="
nmap -sS -T4 localhost -oA /tmp/nmap_quick 2>/dev/null | grep "open"
```

> ✅ **Expected Output**
> ```
> 80/tcp   open  http
> 3000/tcp open  ppp
> ```
> Two ports open — DVWA on 80, Juice Shop on 3000.

> 🔧 **If Stuck**
> - `requires root privileges` → Use `sudo nmap -sS ...` or switch to `-sT` (connect scan, no sudo needed)
> - No ports found → Docker containers might not be running. Check: `docker ps`

**Step 2: Full port scan**
```bash
# ALL 65535 ports
echo "=== Full Port Scan ==="
sudo nmap -sS -p- --min-rate 10000 localhost -oA /tmp/nmap_full 2>/dev/null | grep "open"
```

> ✅ **Expected Output**
> ```
> 80/tcp   open  http
> 3000/tcp open  ppp
> ```
> On a real target, the full scan often reveals additional ports (8080, 8443, 9090, etc.) that the quick scan missed!

**Step 3: Service enumeration**
```bash
# Deep version detection on discovered ports
echo "=== Service Detection ==="
nmap -sV -sC -p 80,3000 localhost -oA /tmp/nmap_services 2>/dev/null
cat /tmp/nmap_services.nmap | grep -A 3 "open"
```

> ✅ **Expected Output**
> ```
> 80/tcp   open  http    Apache httpd 2.4.x ((Debian))
> |_http-server-header: Apache/2.4.x (Debian)
> |_http-title: DVWA
> 3000/tcp open  http    Node.js Express framework
> |_http-title: OWASP Juice Shop
> ```
> Now you know the exact software versions — Apache on DVWA, Express on Juice Shop. Search these versions for CVEs!

**Step 4: Vulnerability scanning with NSE**
```bash
echo "=== Vulnerability Scripts ==="
nmap --script vuln -p 80,3000 localhost -oA /tmp/nmap_vulns 2>/dev/null | grep -E "VULNERABLE|http-" | head -15
```

> ✅ **Expected Output**
> ```
> |   VULNERABLE:
> |     Slowloris DOS attack
> |       State: LIKELY VULNERABLE
> | http-enum:
> |   /ftp/: Potentially interesting directory
> |   /robots.txt: Robots file
> ```
> NSE found that Juice Shop's `/ftp` directory is accessible and that the server may be vulnerable to Slowloris DoS.

**Step 5: UDP scan**
```bash
echo "=== UDP Scan (top 20 ports) ==="
sudo nmap -sU --top-ports 20 -T4 localhost -oA /tmp/nmap_udp 2>/dev/null | grep "open"
```

> ✅ **Expected Output**
> On local Docker targets, UDP ports are usually closed. On a real server, you might find DNS (53), SNMP (161), or TFTP (69).

> 🔧 **If Stuck**
> - UDP scan takes forever → This is normal. UDP scanning is inherently slow because there's no connection acknowledgment. Use `--top-ports 20` instead of a full scan.
> - All ports "open|filtered" → This is UDP's default state when no response is received. Focus on ports that show as definitively "open".

**Step 6: Parse and search**
```bash
# Extract key findings
echo "=== Summary ==="
echo "Open TCP ports:"
grep "open" /tmp/nmap_services.nmap | grep "tcp"

echo ""
echo "Technologies found:"
grep -E "http-title|server-header|http-generator" /tmp/nmap_services.nmap
```

### ✅ Success Criteria
- [ ] Full port scan completed (all 65535 ports)
- [ ] All open services identified with versions (Apache, Express)
- [ ] At least 1 vulnerability identified through NSE scripts
- [ ] UDP scan completed
- [ ] Output saved in all formats (`-oA`)

---

## 🧪 Lab 4: Directory & File Enumeration

### Objective
Master directory brute forcing with different tools and techniques.

> 💡 **Why This Matters**
> Developers deploy files they think are hidden — backup configs (`.env.bak`), debug endpoints (`/debug`), admin panels (`/admin`), and old API versions (`/api/v1`). Directory brute forcing systematically tries thousands of common paths. A single discovered file can contain database credentials or expose an entire admin interface.

### Steps

**Step 1: Basic enumeration with ffuf**
```bash
# Against Juice Shop
echo "=== ffuf common.txt ==="
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,401,403 \
  -t 20 -s 2>/dev/null | head -15
```

> ✅ **Expected Output**
> ```
> assets
> ftp
> profile
> robots.txt
> socket.io
> ```
> Key discovery: `/ftp` — an accessible file directory!

> 🔧 **If Stuck**
> - `ffuf: command not found` → `sudo apt install ffuf -y` or `go install github.com/ffuf/ffuf/v2@latest`
> - SecLists not found → `sudo apt install seclists -y`
> - Too many results flooding the screen → Add `-fs SIZE` to filter by response size, e.g., `-fs 6788` to hide default pages

**Step 2: Extended wordlist with extensions**
```bash
echo "=== ffuf with extensions ==="
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -e .js,.json,.bak,.old,.txt,.xml \
  -mc 200,301,302 \
  -t 20 -s 2>/dev/null | head -20
```

> ✅ **Expected Output**
> Additional files found with extensions — `.json` config files, `.js` application files, `.txt` documentation.

**Step 3: API endpoint discovery**
```bash
# Juice Shop has both /api/ and /rest/ endpoints
echo "=== /api/ endpoints ==="
ffuf -u http://localhost:3000/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,401,403,500 \
  -t 20 -s 2>/dev/null

echo ""
echo "=== /rest/ endpoints ==="
ffuf -u http://localhost:3000/rest/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,401,403,500 \
  -t 20 -s 2>/dev/null
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
> You've mapped the complete API! Each endpoint is now a target for IDOR, injection, and auth bypass testing.

**Step 4: Compare tools — gobuster**
```bash
# Same target, different tool
echo "=== gobuster ==="
gobuster dir -u http://localhost:3000 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 20 -q 2>/dev/null | head -15
```

> ✅ **Expected Output**
> Similar results to ffuf. Compare — did one tool find something the other missed? This is why professionals use multiple tools.

**Step 5: Check for sensitive files**
```bash
echo "=== Sensitive File Check ==="
for file in .git/HEAD .env .htaccess robots.txt sitemap.xml \
  web.config server-status .DS_Store backup.zip database.sql \
  package.json swagger.json api-docs; do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/$file)
  if [ "$code" != "404" ]; then
    echo "  ⚠️  $file → HTTP $code"
  else
    echo "  ✅ $file → 404"
  fi
done
```

> ✅ **Expected Output**
> ```
>   ✅ .git/HEAD → 404
>   ✅ .env → 404
>   ⚠️  robots.txt → 200
>   ✅ sitemap.xml → 404
>   ⚠️  package.json → 200
>   ✅ swagger.json → 404
>   ...
> ```
> `robots.txt` (200) reveals hidden paths. `package.json` (200) reveals dependencies — check for known vulnerable packages!

### ✅ Success Criteria
- [ ] Used both ffuf and gobuster (compared results)
- [ ] Discovered API endpoints (`/api/Users`, `/rest/basket`, etc.)
- [ ] Checked for sensitive file exposure (at least 10 paths)
- [ ] Found at least 5 non-obvious paths
- [ ] Documented findings with status codes

---

## 🧪 Lab 5: Build a Recon Automation Script

### Objective
Write a bash script that automates your complete recon workflow.

> 💡 **Why This Matters**
> Manual recon is thorough but slow. On a bug bounty program with a wide scope (`*.target.com`), manually checking each subdomain is impossible. Automation lets you run recon against hundreds of targets while you sleep, and focus your manual skills on the interesting findings.

### Step 1: Create the script

```bash
#!/bin/bash
# recon_lab.sh — Automated recon against a target
# Usage: ./recon_lab.sh <domain>

set -e

if [ -z "$1" ]; then
  echo "Usage: $0 <domain>"
  exit 1
fi

DOMAIN=$1
OUTDIR="recon_${DOMAIN}_$(date +%Y%m%d_%H%M%S)"
mkdir -p $OUTDIR/{subs,web,ports,vulns}

echo "============================================"
echo "  Recon Automation — $DOMAIN"
echo "  Output: $OUTDIR"
echo "============================================"

# Phase 1: Subdomain discovery
echo -e "\n[Phase 1] Subdomain Enumeration"
echo "  [*] Running subfinder..."
subfinder -d $DOMAIN -silent > $OUTDIR/subs/subfinder.txt 2>/dev/null
echo "  [+] subfinder: $(wc -l < $OUTDIR/subs/subfinder.txt) subdomains"

echo "  [*] Querying crt.sh..."
curl -s "https://crt.sh/?q=%25.${DOMAIN}&output=json" | \
  jq -r '.[].name_value' 2>/dev/null | \
  sed 's/\*\.//g' | sort -u > $OUTDIR/subs/crtsh.txt
echo "  [+] crt.sh: $(wc -l < $OUTDIR/subs/crtsh.txt) subdomains"

# Combine
cat $OUTDIR/subs/*.txt | sort -u > $OUTDIR/subs/all.txt
echo "  [+] Total unique: $(wc -l < $OUTDIR/subs/all.txt)"

# Phase 2: Live host detection
echo -e "\n[Phase 2] Live Host Detection"
cat $OUTDIR/subs/all.txt | httpx -silent -status-code -title -tech-detect \
  > $OUTDIR/web/live_hosts.txt 2>/dev/null
echo "  [+] Live hosts: $(wc -l < $OUTDIR/web/live_hosts.txt)"

# Phase 3: Interesting subdomains
echo -e "\n[Phase 3] Interesting Subdomains"
cat $OUTDIR/web/live_hosts.txt | \
  grep -iE "staging|dev|test|admin|internal|api|jenkins|gitlab|grafana|kibana|jira|confluence" \
  > $OUTDIR/web/interesting.txt 2>/dev/null || true
echo "  [+] Interesting: $(wc -l < $OUTDIR/web/interesting.txt)"

# Phase 4: Summary
echo -e "\n============================================"
echo "  SUMMARY"
echo "  Subdomains:  $(wc -l < $OUTDIR/subs/all.txt)"
echo "  Live hosts:  $(wc -l < $OUTDIR/web/live_hosts.txt)"
echo "  Interesting: $(wc -l < $OUTDIR/web/interesting.txt)"
echo "  Output dir:  $OUTDIR/"
echo "============================================"
```

### Step 2: Test the script
```bash
# Save the script
# (Copy the code above into recon_lab.sh)

chmod +x recon_lab.sh
./recon_lab.sh hackerone.com
```

> ✅ **Expected Output**
> ```
> ============================================
>   Recon Automation — hackerone.com
>   Output: recon_hackerone.com_20260406_120000
> ============================================
>
> [Phase 1] Subdomain Enumeration
>   [*] Running subfinder...
>   [+] subfinder: 15 subdomains
>   [*] Querying crt.sh...
>   [+] crt.sh: 25 subdomains
>   [+] Total unique: 30
>
> [Phase 2] Live Host Detection
>   [+] Live hosts: 8
>
> [Phase 3] Interesting Subdomains
>   [+] Interesting: 2
>
> ============================================
>   SUMMARY
>   Subdomains:  30
>   Live hosts:  8
>   Interesting: 2
>   Output dir:  recon_hackerone.com_20260406_120000/
> ============================================
> ```

> 🔧 **If Stuck**
> - `subfinder: command not found` → The script still works with just crt.sh. Or install subfinder first.
> - `httpx: command not found` → Install: `go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest`
> - Script exits immediately → Check `set -e` at the top. This makes it exit on ANY error. Remove it for debugging, or add `|| true` after commands that might fail.
> - No live hosts → The target might be blocking automated tools. Try adding `--timeout 10` to the httpx command.

### Step 3: Enhance the script

Add these features (pick at least 2):
1. **Port scanning**: Add `nmap -sS -T4 --top-ports 100` on discovered IPs
2. **Sensitive file check**: Loop through common files (`.git/HEAD`, `.env`, `robots.txt`) on each live host
3. **Nuclei scanning**: `nuclei -l live_hosts.txt -severity critical,high`
4. **Slack/Telegram notification**: Send summary via webhook on completion
5. **HTML report**: Generate a formatted HTML summary

### ✅ Success Criteria
- [ ] Script runs end-to-end without errors
- [ ] Output is organized in folders (subs/, web/, ports/)
- [ ] You added at least 2 enhancements
- [ ] Results are consistent with manual recon

---

## 🧪 Lab 6: Technology Fingerprinting Deep Dive

### Objective
Identify the complete technology stack of Juice Shop using multiple techniques.

> 💡 **Why This Matters**
> Knowing the tech stack narrows your attack from "try everything" to "try exploits for Express + SQLite + Angular." A Node.js app is vulnerable to different attacks (prototype pollution, NoSQL injection) than a PHP app (type juggling, deserialization). Technology fingerprinting tells you WHAT to attack.

### Steps

**Step 1: HTTP header analysis**
```bash
echo "=== Step 1: Response Headers ==="
curl -sI http://localhost:3000 | tee /tmp/headers.txt

echo ""
echo "--- Key Findings ---"
grep -i "x-powered-by" /tmp/headers.txt && echo "  → Technology leaked!"
grep -i "server:" /tmp/headers.txt || echo "  → Server header hidden (good practice)"
grep -i "set-cookie" /tmp/headers.txt || echo "  → No cookies set on homepage"
grep -i "access-control" /tmp/headers.txt && echo "  → CORS configured"
```

> ✅ **Expected Output**
> ```
> HTTP/1.1 200 OK
> X-Powered-By: Express
> Access-Control-Allow-Origin: *
> X-Content-Type-Options: nosniff
> X-Frame-Options: SAMEORIGIN
>
> --- Key Findings ---
> X-Powered-By: Express  → Technology leaked!
>   → Server header hidden (good practice)
>   → No cookies set on homepage
> Access-Control-Allow-Origin: *  → CORS configured
> ```

**Step 2: whatweb analysis**
```bash
echo "=== Step 2: whatweb ==="
whatweb -v http://localhost:3000 2>/dev/null | head -20
```

> ✅ **Expected Output**
> ```
> http://localhost:3000 [200 OK]
>   Country[RESERVED][ZZ]
>   HTML5
>   HTTPServer[Express]
>   Script
>   X-Frame-Options[SAMEORIGIN]
>   X-Powered-By[Express]
>   X-XSS-Protection[0]
> ```

> 🔧 **If Stuck**
> - `whatweb: command not found` → `sudo apt install whatweb -y`

**Step 3: JavaScript analysis (in browser)**

Open `http://localhost:3000` in your browser and press F12:
```javascript
// In the Console tab, run these:

// Check for Angular (Juice Shop uses Angular)
typeof angular !== 'undefined'    // For AngularJS
// Or look for ng-version in the page source

// Check for global variables
Object.keys(window).filter(k => k.startsWith('__'))
// Look for __ANGULAR_DEVTOOLS_GLOBAL_HOOK__, __webpack_modules__, etc.

// List all loaded JavaScript files
performance.getEntriesByType('resource')
  .filter(r => r.name.endsWith('.js'))
  .map(r => r.name.split('/').pop())
```

> ✅ **Expected Output**
> ```javascript
> // You'll see Angular-related globals and JS files like:
> // main.js, polyfills.js, runtime.js, vendor.js
> // These confirm Angular as the frontend framework
> ```

**Step 4: Error-based fingerprinting**
```bash
echo "=== Step 4: Error-Based Detection ==="

# Force 404
echo "--- 404 Error ---"
curl -s http://localhost:3000/nonexistent_xyz_page | head -3

# Force 500 (SQL error)
echo ""
echo "--- 500 Error (SQLi probe) ---"
curl -s "http://localhost:3000/rest/products/search?q='" | python3 -m json.tool 2>/dev/null | head -8
```

> ✅ **Expected Output**
> ```
> --- 404 Error ---
> <!DOCTYPE html>... (returns the Angular app — SPA handles routing)
>
> --- 500 Error (SQLi probe) ---
> {
>     "error": {
>         "message": "SQLITE_ERROR: near \"'\": syntax error",
>         "stack": "SequelizeDatabaseError: SQLITE_ERROR..."
>     }
> }
> ```
> **The error reveals everything:**
> - Database: **SQLite**
> - ORM: **Sequelize**
> - Framework: **Express** (from stack trace)
> - Error handling: **Verbose** (full stack trace exposed — bad practice!)

### Complete Tech Stack Document

```markdown
## Juice Shop Technology Stack

| Layer | Technology | Source | Attack Vectors |
|-------|-----------|--------|----------------|
| Frontend | Angular | JS analysis, page source | DOM XSS, template injection |
| Backend | Express (Node.js) | X-Powered-By header | Prototype pollution, path traversal |
| Database | SQLite | Error message | SQL injection |
| ORM | Sequelize | Stack trace | ORM-specific injection |
| Auth | JWT (RS256) | Login response | Algorithm confusion, none attack |
| CORS | * (allow all) | Access-Control-Allow-Origin | Cross-origin data theft |
```

### ✅ Success Criteria
- [ ] Complete technology stack documented (frontend, backend, database, auth)
- [ ] Multiple fingerprinting methods used (headers, whatweb, errors, JS)
- [ ] Error pages analyzed for information leakage
- [ ] Findings cross-referenced between methods
- [ ] Tech stack document created with attack vectors per technology

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: Full passive recon documented (DNS, crt.sh, Google dorks, Wayback)
- [ ] Lab 2: Multi-source subdomain enumeration completed
- [ ] Lab 3: Nmap deep dive with all scan types (TCP, UDP, NSE)
- [ ] Lab 4: Directory enumeration with multiple tools (ffuf + gobuster)
- [ ] Lab 5: Recon automation script built and tested
- [ ] Lab 6: Technology stack fully fingerprinted

---

## 🧠 If You're Stuck

1. **subfinder/amass not finding anything**: These tools need API keys for many sources. Check `~/.config/subfinder/provider-config.yaml`. Even without keys, crt.sh alone finds many subdomains.
2. **httpx timing out**: Add `--timeout 5` flag. Some hosts are slow. Also try `--retries 2`.
3. **ffuf too noisy**: Use `-fs SIZE` to filter common response sizes (run once, note the default size, then filter it). Or use `-mc 200,301` to only show valid responses.
4. **Nmap scan taking forever**: Use `--min-rate 5000` with `-sS`. Avoid `-A` on full port scans. For quick results, scan `--top-ports 100` first.
5. **No results from Google dorking**: Try different operators, be more creative with search terms. Some well-maintained sites have minimal Google-indexed exposure — that's actually a finding (good security posture).
6. **Go tools won't install**: Make sure Go is installed (`sudo apt install golang-go -y`) and `~/go/bin` is in your PATH: `export PATH=$PATH:~/go/bin`
7. **Overwhelmed by results**: Focus on the interesting subdomains first — anything with "staging", "dev", "admin", or "api" in the name. Those are the high-value targets.
