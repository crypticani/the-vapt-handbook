# 02 — Recon & Enumeration: Tools & Real Usage

> Every tool here is something you'll use in every engagement. Master the flags — don't just copy-paste.

---

## 🔴 Nmap (Network Mapper)

```bash
# Installation
sudo apt install nmap -y

# === DISCOVERY SCANS ===

# Ping sweep — find live hosts on a network
nmap -sn 10.10.10.0/24 -oA discovery

# ARP scan (local network only — more reliable than ping)
nmap -sn -PR 192.168.1.0/24

# === PORT SCANS ===

# Quick top 1000 TCP scan
nmap -sS -T4 10.10.10.100

# Full TCP port scan
nmap -sS -p- --min-rate 5000 10.10.10.100 -oA full_tcp

# Specific ports with version detection + default scripts
nmap -sV -sC -p 22,80,443,3306,8080 10.10.10.100 -oA targeted

# UDP scan (top 20 — UDP is slow)
nmap -sU --top-ports 20 10.10.10.100 -oA udp

# === SERVICE ENUMERATION ===

# Aggressive scan (OS detection + version + scripts + traceroute)
nmap -A -p 80,443 10.10.10.100

# Banner grabbing
nmap -sV --version-intensity 5 -p 80,443 10.10.10.100

# === NSE SCRIPTS ===

# HTTP enumeration
nmap --script http-enum -p 80,443 10.10.10.100

# Vulnerability scan
nmap --script vuln -p 80,443 10.10.10.100

# SMB enumeration
nmap --script smb-enum-shares,smb-enum-users,smb-os-discovery -p 445 10.10.10.100

# SSL/TLS analysis  
nmap --script ssl-enum-ciphers,ssl-cert -p 443 10.10.10.100

# DNS enumeration
nmap --script dns-brute --script-args dns-brute.domain=target.com

# WordPress enumeration
nmap --script http-wordpress-enum -p 80,443 10.10.10.100

# === EVASION ===

# Stealth scan through firewall
nmap -sS -T2 -f --source-port 53 -D RND:5 10.10.10.100

# Scan through proxy
proxychains nmap -sT -Pn -p 80,443 10.10.10.100

# === OUTPUT PARSING ===

# Parse grepable output
cat scan.gnmap | grep "open" | cut -d' ' -f2 | sort -u  # Get IPs with open ports
cat scan.gnmap | grep "80/open"  # Find all hosts with port 80

# Convert XML to HTML report
xsltproc scan.xml -o report.html
```

---

## 🔴 Subfinder (Subdomain Discovery)

```bash
# Installation
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# Basic usage
subfinder -d target.com -o subdomains.txt

# With all sources (requires API keys in ~/.config/subfinder/provider-config.yaml)
subfinder -d target.com -all -o subdomains.txt

# Silent mode (only output subdomains)
subfinder -d target.com -silent | httpx -silent

# Multiple domains
subfinder -dL domains.txt -o all_subdomains.txt

# With specific sources
subfinder -d target.com -sources crtsh,virustotal,shodan

# Recursive subdomain enumeration
subfinder -d target.com -recursive -o recursive_subs.txt
```

---

## 🔴 httpx (HTTP Toolkit)

```bash
# Installation
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest

# Probe for live hosts from subdomain list
cat subdomains.txt | httpx -silent -status-code -title -tech-detect

# Detailed output
cat subdomains.txt | httpx -silent \
  -status-code \
  -title \
  -tech-detect \
  -web-server \
  -content-length \
  -follow-redirects \
  -o live_hosts.txt

# Filter by status code
cat subdomains.txt | httpx -silent -mc 200,301,302

# Screenshot all live hosts
cat subdomains.txt | httpx -silent -screenshot -screenshot-timeout 10

# JSON output for parsing
cat subdomains.txt | httpx -silent -json -o results.json
```

---

## 🔴 Amass (Attack Surface Mapping)

```bash
# Installation
go install -v github.com/owasp-amass/amass/v4/...@master

# Passive enumeration (no direct contact)
amass enum -passive -d target.com -o amass_passive.txt

# Active enumeration  
amass enum -active -d target.com -o amass_active.txt

# Brute force included
amass enum -brute -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Intelligence gathering (relationships, AS numbers)
amass intel -d target.com

# With multiple data sources
amass enum -d target.com -active -brute -ip -src -o amass_full.txt
```

---

## 🔴 ffuf (Fuzz Faster U Fool)

```bash
# Installation
go install github.com/ffuf/ffuf/v2@latest

# === DIRECTORY FUZZING ===

# Basic directory brute force
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -mc 200,301,302,401,403 \
  -o output.json -of json

# With extensions
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -e .php,.asp,.aspx,.jsp,.html,.js,.json,.bak,.txt,.xml,.sql,.zip,.old \
  -mc 200,301,302 \
  -t 100

# Recursive scanning
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -recursion -recursion-depth 3

# === PARAMETER FUZZING ===

# GET parameter discovery
ffuf -u 'https://target.com/api/v1/resource?FUZZ=1' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200

# POST parameter discovery (JSON)
ffuf -u https://target.com/api/v1/resource \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"FUZZ": "test"}' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200

# === VHOST FUZZING ===

# Virtual host discovery
ffuf -u https://target.com \
  -H "Host: FUZZ.target.com" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 1234  # Filter default response size

# === FUZZING WITH FILTER ===

# Filter by response size (remove noise)
ffuf -u https://target.com/FUZZ -w wordlist.txt -fs 4242

# Filter by word count
ffuf -u https://target.com/FUZZ -w wordlist.txt -fw 100

# Filter by line count
ffuf -u https://target.com/FUZZ -w wordlist.txt -fl 10

# Filter by regex
ffuf -u https://target.com/FUZZ -w wordlist.txt -fr "not found"

# === AUTHENTICATION ===

# With cookies
ffuf -u https://target.com/FUZZ -w wordlist.txt -b "session=abc123"

# With headers
ffuf -u https://target.com/FUZZ -w wordlist.txt -H "Authorization: Bearer TOKEN"

# === SPEED CONTROL ===

# Rate limiting
ffuf -u https://target.com/FUZZ -w wordlist.txt -rate 100  # 100 req/sec

# Threads
ffuf -u https://target.com/FUZZ -w wordlist.txt -t 200  # 200 threads
```

---

## 🔴 Gobuster

```bash
# Installation
go install github.com/OJ/gobuster/v3@latest

# Directory brute force
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -t 50 \
  -o gobuster.txt

# With extensions
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -x php,txt,html,bak,old,js,json \
  -t 50

# DNS subdomain brute force
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -t 50

# VHOST brute force
gobuster vhost -u https://target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -t 50

# With authentication
gobuster dir -u https://target.com \
  -w wordlist.txt \
  -c "session=abc123" \
  -t 50

# Custom status codes
gobuster dir -u https://target.com \
  -w wordlist.txt \
  -s "200,204,301,302,307,401,403" \
  -t 50

# With custom User-Agent
gobuster dir -u https://target.com \
  -w wordlist.txt \
  -a "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -t 50
```

---

## 🔴 Nuclei (Template-Based Scanner)

```bash
# Installation
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Update templates
nuclei -update-templates

# Scan a single target
nuclei -u https://target.com -o nuclei_results.txt

# Scan multiple targets
nuclei -l live_hosts.txt -o nuclei_results.txt

# Filter by severity
nuclei -u https://target.com -severity critical,high

# Filter by specific tags
nuclei -u https://target.com -tags cve,sqli,xss,rce

# Technology-specific scan
nuclei -u https://target.com -tags wordpress
nuclei -u https://target.com -tags nginx
nuclei -u https://target.com -tags jenkins

# Custom rate limiting
nuclei -u https://target.com -rate-limit 20 -bulk-size 5

# JSON output
nuclei -u https://target.com -json -o results.json
```

---

## 🔴 Shodan (Internet-Wide Scanner)

```bash
# Installation
pip3 install shodan

# Initialize with API key
shodan init YOUR_API_KEY

# Search for a target
shodan host 104.26.10.78

# Search query
shodan search "hostname:target.com" --fields ip_str,port,org,hostnames

# Find all IPs for an organization
shodan search "org:\"Target Corp\"" --fields ip_str,port,data

# Useful search queries:
# hostname:target.com           → All IPs associated with target
# ssl.cert.subject.cn:target.com → SSL certificates for target
# org:"Target Corp"             → All assets for the organization
# http.title:"Dashboard"        → Find exposed dashboards
# port:9200 country:US          → Elasticsearch instances
# "default password"            → Devices with default creds
# product:jenkins               → Jenkins instances
# product:kubernetes            → Kubernetes API servers

# Browser: https://www.shodan.io/search?query=hostname:target.com
```

---

## 🔴 waybackurls

```bash
# Installation
go install github.com/tomnomnom/waybackurls@latest

# Get all historical URLs
echo target.com | waybackurls > wayback.txt

# Filter for potentially interesting files
cat wayback.txt | grep -iE '\.(bak|sql|zip|tar|gz|env|config|xml|json|log|old|txt)$'

# Filter for URLs with parameters (injection targets)
cat wayback.txt | grep '?' | sort -u > params.txt

# Extract unique paths
cat wayback.txt | unfurl paths | sort -u > paths.txt

# Extract unique query keys
cat wayback.txt | unfurl keys | sort -u > keys.txt
```

---

## 🔴 Recon Automation Pipeline

```bash
#!/bin/bash
# recon.sh — Automated recon pipeline
# Usage: ./recon.sh target.com

DOMAIN=$1
OUTPUT="recon_${DOMAIN}"
mkdir -p $OUTPUT

echo "[*] Starting recon on $DOMAIN"

# 1. Subdomain enumeration
echo "[*] Subdomain enumeration..."
subfinder -d $DOMAIN -silent > $OUTPUT/subs_subfinder.txt
amass enum -passive -d $DOMAIN -o $OUTPUT/subs_amass.txt 2>/dev/null
curl -s "https://crt.sh/?q=%25.$DOMAIN&output=json" | jq -r '.[].name_value' 2>/dev/null | sed 's/\*\.//g' > $OUTPUT/subs_crtsh.txt

# Combine and deduplicate
cat $OUTPUT/subs_*.txt | sort -u > $OUTPUT/all_subdomains.txt
echo "[+] Found $(wc -l < $OUTPUT/all_subdomains.txt) unique subdomains"

# 2. Find live hosts
echo "[*] Probing for live hosts..."
cat $OUTPUT/all_subdomains.txt | httpx -silent -status-code -title -tech-detect -web-server > $OUTPUT/live_hosts.txt
echo "[+] Found $(wc -l < $OUTPUT/live_hosts.txt) live hosts"

# 3. Port scan live IPs
echo "[*] Port scanning..."
cat $OUTPUT/all_subdomains.txt | sort -u | while read sub; do
  dig +short $sub 2>/dev/null
done | sort -u > $OUTPUT/ips.txt

nmap -sS -T4 --top-ports 1000 -iL $OUTPUT/ips.txt -oA $OUTPUT/nmap_scan 2>/dev/null

# 4. Directory brute force on live web hosts
echo "[*] Directory reconnaissance..."
cat $OUTPUT/live_hosts.txt | awk '{print $1}' | while read url; do
  domain=$(echo $url | unfurl domains)
  ffuf -u "$url/FUZZ" \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -mc 200,301,302,401,403 \
    -t 50 -rate 100 \
    -o "$OUTPUT/ffuf_${domain}.json" -of json 2>/dev/null
done

# 5. Wayback URLs
echo "[*] Wayback Machine..."
echo $DOMAIN | waybackurls > $OUTPUT/wayback.txt 2>/dev/null

# 6. Nuclei scan
echo "[*] Running nuclei..."
nuclei -l $OUTPUT/live_hosts.txt -severity critical,high -o $OUTPUT/nuclei.txt 2>/dev/null

# 7. Generate summary
echo ""
echo "========== RECON SUMMARY =========="
echo "Subdomains found:    $(wc -l < $OUTPUT/all_subdomains.txt)"
echo "Live hosts:          $(wc -l < $OUTPUT/live_hosts.txt)"
echo "Unique IPs:          $(wc -l < $OUTPUT/ips.txt)"
echo "Wayback URLs:        $(wc -l < $OUTPUT/wayback.txt 2>/dev/null || echo 0)"
echo "Critical/High vulns: $(wc -l < $OUTPUT/nuclei.txt 2>/dev/null || echo 0)"
echo "Results saved to:    $OUTPUT/"
echo "===================================="
```

---

## 🔴 Wordlists Reference

```bash
# SecLists (the standard)
# Install: sudo apt install seclists
# Location: /usr/share/seclists/

# Common choices:
/usr/share/seclists/Discovery/Web-Content/common.txt              # 4,700 entries — fast
/usr/share/seclists/Discovery/Web-Content/raft-large-words.txt     # 119,000 entries — thorough
/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt  # 220,000 — comprehensive
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt  # Quick subdomain check
/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt # Thorough subdomain check
/usr/share/seclists/Usernames/top-usernames-shortlist.txt          # Username brute force
/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt # Password brute force

# Custom wordlists
# cewl — generate wordlist from target website
cewl https://target.com -d 2 -m 5 -w custom_wordlist.txt

# Generate from specific technology
# PHP: add .php, .php.bak, .php~, .php.old, .phps
# Java: add .jsp, .jsf, .do, .action
# .NET: add .aspx, .ashx, .asmx, .axd, .config
```

---

## 📋 Tool Selection Matrix

| Task | Primary Tool | Alternative | When To Use |
|------|-------------|-------------|-------------|
| Subdomain discovery | subfinder | amass, assetfinder | Every engagement |
| Live host detection | httpx | masscan | After subdomain enum |
| Port scanning | nmap | masscan, rustscan | Every engagement |
| Directory fuzzing | ffuf | gobuster, dirsearch | Every web target |
| Parameter fuzzing | ffuf + Burp Intruder | wfuzz | When testing inputs |
| Vuln scanning | nuclei | nikto | Quick win detection |
| Technology detection | whatweb + httpx | wappalyzer | Every web target |
| OSINT | shodan + Google | censys, spyse | Passive phase |
| Wayback URLs | waybackurls | gau | Historical discovery |
| API scanning | nuclei + Postman | Burp | API-heavy targets |
