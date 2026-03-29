# 02 — Recon & Enumeration: Labs

> These labs build your recon muscle memory. You'll practice the complete reconnaissance workflow against intentionally vulnerable targets.

---

## 🧪 Lab 1: Full Passive Recon Against a Real Domain

### Objective
Perform complete passive reconnaissance without touching the target directly. Use a public bug bounty program (e.g., `*.hackerone.com` or a program from HackerOne's directory).

### Steps

**Step 1: DNS Enumeration**
```bash
TARGET="hackerone.com"

# All DNS record types
for type in A AAAA MX NS TXT CNAME SOA; do
  echo "=== $type ==="
  dig $TARGET $type +short
done

# Zone transfer attempt
dig axfr @$(dig $TARGET NS +short | head -1) $TARGET
```

**Step 2: Certificate Transparency**
```bash
# Pull subdomains from crt.sh
curl -s "https://crt.sh/?q=%25.${TARGET}&output=json" | \
  jq -r '.[].name_value' | \
  sed 's/\*\.//g' | \
  sort -u > crtsh_subs.txt

echo "Found $(wc -l < crtsh_subs.txt) subdomains"
cat crtsh_subs.txt | head -20
```

**Step 3: Google Dorking**
```
Manually search in Google:
1. site:hackerone.com filetype:pdf
2. site:hackerone.com inurl:admin
3. site:hackerone.com intitle:"index of"
4. site:*.hackerone.com -www
5. site:github.com "hackerone.com" password
```

**Step 4: Wayback Machine**
```bash
echo $TARGET | waybackurls | head -50 > wayback.txt
cat wayback.txt | grep -iE '\.env|\.bak|\.sql|\.zip|\.log' 
cat wayback.txt | grep '?' | sort -u | head -20
```

**Step 5: Document Findings**
Create a markdown report with:
- Technology stack identified
- Subdomains found
- Interesting historical URLs
- Potential attack vectors

### ✅ Success Criteria
- [ ] 50+ subdomains discovered through passive means
- [ ] DNS records fully documented
- [ ] At least 3 Google dork results analyzed
- [ ] Historical URLs reviewed for sensitive files

---

## 🧪 Lab 2: Active Subdomain Enumeration & Resolution

### Objective
Combine multiple tools to build a comprehensive subdomain list, then identify live hosts.

### Steps

**Step 1: Multi-source subdomain enumeration**
```bash
TARGET="target.com"  # Use a HackerOne program target
mkdir -p recon/$TARGET

# Source 1: subfinder
subfinder -d $TARGET -silent > recon/$TARGET/subs_subfinder.txt

# Source 2: crt.sh
curl -s "https://crt.sh/?q=%25.${TARGET}&output=json" | \
  jq -r '.[].name_value' 2>/dev/null | \
  sed 's/\*\.//g' | sort -u > recon/$TARGET/subs_crtsh.txt

# Source 3: waybackurls (extract subdomains from URLs)
echo $TARGET | waybackurls 2>/dev/null | \
  unfurl domains 2>/dev/null | sort -u > recon/$TARGET/subs_wayback.txt

# Combine
cat recon/$TARGET/subs_*.txt | sort -u > recon/$TARGET/all_subs.txt
echo "Total unique subdomains: $(wc -l < recon/$TARGET/all_subs.txt)"
```

**Step 2: DNS resolution**
```bash
# Resolve all subdomains to IPs
cat recon/$TARGET/all_subs.txt | while read sub; do
  ip=$(dig +short $sub 2>/dev/null | grep -oE '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | head -1)
  if [ -n "$ip" ]; then
    echo "$sub,$ip"
  fi
done > recon/$TARGET/resolved.csv

echo "Resolved: $(wc -l < recon/$TARGET/resolved.csv) subdomains"
```

**Step 3: Live host probing**
```bash
cat recon/$TARGET/all_subs.txt | httpx -silent \
  -status-code \
  -title \
  -tech-detect \
  -web-server \
  -content-length \
  -follow-redirects \
  -o recon/$TARGET/live_hosts.txt

echo "Live hosts: $(wc -l < recon/$TARGET/live_hosts.txt)"
```

**Step 4: Categorize findings**
```bash
# Find interesting subdomains
cat recon/$TARGET/live_hosts.txt | grep -iE "staging|dev|test|admin|internal|api|jenkins|gitlab|grafana|kibana"
```

### ✅ Success Criteria
- [ ] Used at least 3 different enumeration sources
- [ ] Successfully resolved subdomains to IPs
- [ ] Identified live hosts with their technologies
- [ ] Found at least 1 "interesting" subdomain (staging, dev, admin)

---

## 🧪 Lab 3: Nmap Deep Dive

### Objective
Master Nmap scanning techniques against a local lab machine.

### Setup
```bash
# Use a vulnerable machine from VulnHub or HackTheBox
# Or use Metasploitable:
docker run -d -p 2222:22 -p 8080:80 -p 3306:3306 --name metasploitable tleemcjr/metasploitable2
# Target: localhost or the Docker bridge IP
```

### Steps

**Step 1: Discovery scan**
```bash
# Quick top 1000 ports
nmap -sS -T4 localhost -oA nmap_quick
# Document: How many ports open? What services detected?
```

**Step 2: Full port scan**
```bash
# ALL 65535 ports
nmap -sS -p- --min-rate 5000 localhost -oA nmap_full
# Compare: Did the full scan find ports the quick scan missed?
```

**Step 3: Service enumeration**
```bash
# Deep version detection on all open ports
PORTS=$(grep "^[0-9]" nmap_full.nmap | cut -d'/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -sV -sC -p $PORTS localhost -oA nmap_services
```

**Step 4: Vulnerability scanning**
```bash
nmap --script vuln -p $PORTS localhost -oA nmap_vulns
```

**Step 5: UDP scan**
```bash
sudo nmap -sU --top-ports 50 localhost -oA nmap_udp
```

**Step 6: Parse and analyze**
```bash
# Extract open ports
grep "^[0-9]" nmap_services.nmap

# Search for specific services
grep -i "http\|ftp\|ssh\|mysql\|postgres\|smb" nmap_services.nmap

# Check for vulnerable versions
# For each service version found, search:
# searchsploit <service> <version>
```

### ✅ Success Criteria
- [ ] Full port scan completed (all 65535 ports)
- [ ] All open services identified with versions
- [ ] At least 1 vulnerability identified through NSE scripts
- [ ] UDP scan completed and analyzed
- [ ] Output saved in all formats (-oA)

---

## 🧪 Lab 4: Directory & File Enumeration

### Objective
Master directory brute forcing with different tools and techniques.

### Steps

**Step 1: Basic enumeration with ffuf**
```bash
# Against Juice Shop
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302,401,403 \
  -o ffuf_common.json -of json

# Count results
cat ffuf_common.json | jq '.results | length'
```

**Step 2: Extended wordlist with extensions**
```bash
ffuf -u http://localhost:3000/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -e .js,.json,.bak,.old,.txt \
  -mc 200,301,302 \
  -t 100 \
  -o ffuf_extended.json -of json
```

**Step 3: API endpoint discovery**
```bash
# Juice Shop API enumeration
ffuf -u http://localhost:3000/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt \
  -mc 200,301,302,401,403 \
  -t 50

ffuf -u http://localhost:3000/rest/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt \
  -mc 200,301,302,401,403 \
  -t 50
```

**Step 4: Compare tool output**
```bash
# Run gobuster on the same target
gobuster dir -u http://localhost:3000 \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 50 \
  -o gobuster_output.txt

# Compare results: Did one tool find something the other missed?
```

**Step 5: Check for hidden files**
```bash
# Check for common sensitive files
for file in .git/HEAD .env .htaccess robots.txt sitemap.xml \
  web.config server-status .DS_Store .svn/entries \
  backup.zip database.sql wp-config.php.bak; do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/$file)
  if [ "$code" != "404" ]; then
    echo "[!] $file → HTTP $code"
  fi
done
```

### ✅ Success Criteria
- [ ] Used at least 3 different wordlists
- [ ] Discovered API endpoints
- [ ] Compared results between ffuf and gobuster
- [ ] Checked for sensitive file exposure
- [ ] Found at least 5 non-obvious paths

---

## 🧪 Lab 5: Build a Recon Automation Script

### Objective
Write a bash script that automates your complete recon workflow.

### Steps

**Step 1: Create the script**
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

**Step 2: Test the script**
```bash
chmod +x recon_lab.sh
./recon_lab.sh hackerone.com
```

**Step 3: Enhance the script**

Add these features:
1. Port scanning of discovered IPs
2. Screenshot capture of live hosts
3. Nuclei scanning
4. HTML report generation
5. Slack/Telegram notification on completion

### ✅ Success Criteria
- [ ] Script runs end-to-end without errors
- [ ] Output is organized and readable
- [ ] You added at least 2 enhancements
- [ ] Results are consistent with manual recon

---

## 🧪 Lab 6: Technology Fingerprinting Deep Dive

### Objective
Identify the complete technology stack of a web application using multiple techniques.

### Steps

**Step 1: HTTP header analysis**
```bash
# Juice Shop
curl -sI http://localhost:3000 | tee headers.txt

# What technology indicators do you see?
# Check: Server, X-Powered-By, Set-Cookie, Content-Type
```

**Step 2: whatweb analysis**
```bash
whatweb -v http://localhost:3000
```

**Step 3: JavaScript analysis (in browser)**
```javascript
// Open browser console on the target:
// Check for frameworks
typeof React !== 'undefined'       // React
typeof angular !== 'undefined'     // Angular  
typeof Vue !== 'undefined'         // Vue
typeof jQuery !== 'undefined'      // jQuery

// Check jQuery version (if present)
jQuery.fn.jquery

// Check for state dumps
window.__NEXT_DATA__
window.__NUXT__
window.__REDUX_STATE__

// List all loaded scripts
performance.getEntriesByType('resource')
  .filter(r => r.name.endsWith('.js'))
  .forEach(r => console.log(r.name))
```

**Step 4: Error-based fingerprinting**
```bash
# Force 404 error
curl http://localhost:3000/nonexistent_page_xyz

# Force 500 error  
curl "http://localhost:3000/rest/products/search?q='"

# The error format reveals the framework
# Express.js: "Cannot GET /path"
# Django: Yellow debug page with traceback
# Rails: "Routing Error - No route matches"
# Spring: Whitelabel Error Page
```

### ✅ Success Criteria
- [ ] Complete technology stack documented
- [ ] Multiple fingerprinting methods used
- [ ] Error pages analyzed for information leakage
- [ ] Findings cross-referenced between tools

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: Full passive recon documented
- [ ] Lab 2: Multi-source subdomain enumeration completed
- [ ] Lab 3: Nmap deep dive with all scan types
- [ ] Lab 4: Directory enumeration with multiple tools
- [ ] Lab 5: Recon automation script built and tested
- [ ] Lab 6: Technology stack fully fingerprinted

---

## 🧠 If You're Stuck

1. **subfinder/amass not finding anything**: Check if API keys are configured in config files. Many sources require API keys.
2. **httpx timing out**: Add `--timeout 5` flag. Some hosts are slow.
3. **ffuf too noisy**: Use `-fc 403,404` to filter common non-interesting codes, or `-fs <size>` to filter by response size.
4. **Nmap scan taking forever**: Use `--min-rate 5000` with `-sS`. Avoid `-A` on full port scans.
5. **No results from Google dorking**: Try different operators, be more creative with search terms.
