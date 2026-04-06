# 02 — Recon & Enumeration: Tools & Real Usage

> Every tool here is something you'll use in every engagement. Master the flags — don't just copy-paste.

> 📋 **What You Will Do In This Section**
> - [ ] Install and verify all core recon tools
> - [ ] Run Nmap with different scan types and interpret results
> - [ ] Use subfinder + httpx for subdomain discovery pipeline
> - [ ] Master ffuf for directory, parameter, and virtual host fuzzing
> - [ ] Run Nuclei for automated vulnerability detection
> - [ ] Build a complete recon automation pipeline

---

## 🔴 Nmap (Network Mapper)

> 💡 **Why This Matters**
> Nmap is the foundation of every pentest. It tells you what ports are open, what services are running, and what versions are installed — the minimum information needed to start attacking. You'll use Nmap on literally every engagement.

```bash
# Installation
sudo apt install nmap -y
```

### Discovery & Port Scanning

```bash
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
```

#### 🧪 Try It Now — Scan Your Lab

```bash
# Service detection on your lab targets
nmap -sV -sC -p 80,3000 localhost 2>/dev/null
```

> ✅ **Expected Output**
> ```
> PORT     STATE SERVICE VERSION
> 80/tcp   open  http    Apache httpd 2.4.x ((Debian))
> |_http-title: DVWA
> 3000/tcp open  http    Node.js Express framework
> |_http-title: OWASP Juice Shop
> ```

### NSE Scripts

```bash
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
```

#### 🧪 Try It Now — NSE Against Juice Shop

```bash
nmap --script http-enum -p 3000 localhost 2>/dev/null | grep -A 20 "http-enum"
```

> ✅ **Expected Output**
> ```
> | http-enum:
> |   /ftp/: Potentially interesting directory
> |   /robots.txt: Robots file
> |_  /assets/: Potentially interesting directory
> ```

> 🔧 **If Stuck**
> - `nmap: command not found` → `sudo apt install nmap -y`
> - "requires root privileges" → Use `sudo` for SYN scans, or use `-sT` for non-root

### Evasion & Output

```bash
# === EVASION ===
# Stealth scan through firewall
nmap -sS -T2 -f --source-port 53 -D RND:5 10.10.10.100

# Scan through proxy
proxychains nmap -sT -Pn -p 80,443 10.10.10.100

# === OUTPUT PARSING ===
# Parse grepable output
cat scan.gnmap | grep "open" | cut -d' ' -f2 | sort -u  # IPs with open ports
cat scan.gnmap | grep "80/open"  # Hosts with port 80

# Convert XML to HTML report
xsltproc scan.xml -o report.html
```

---

## 🔴 Subfinder (Subdomain Discovery)

> 💡 **Why This Matters**
> Subfinder queries 40+ data sources (crt.sh, VirusTotal, Shodan, etc.) to find subdomains passively. One command replaces hours of manual searching. The subdomain no one knows about is often the one with the weakest security.

```bash
# Installation
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# Basic usage
subfinder -d target.com -o subdomains.txt

# With all sources (requires API keys in ~/.config/subfinder/provider-config.yaml)
subfinder -d target.com -all -o subdomains.txt

# Silent mode (only output subdomains — great for piping)
subfinder -d target.com -silent | httpx -silent

# Multiple domains
subfinder -dL domains.txt -o all_subdomains.txt

# With specific sources
subfinder -d target.com -sources crtsh,virustotal,shodan

# Recursive subdomain enumeration
subfinder -d target.com -recursive -o recursive_subs.txt
```

#### 🧪 Try It Now

```bash
subfinder -d hackerone.com -silent 2>/dev/null | head -10
echo "---"
echo "Total: $(subfinder -d hackerone.com -silent 2>/dev/null | wc -l) subdomains"
```

> ✅ **Expected Output**
> ```
> api.hackerone.com
> docs.hackerone.com
> www.hackerone.com
> support.hackerone.com
> ...
> ---
> Total: 15+ subdomains
> ```

> 🔧 **If Stuck**
> - `subfinder: command not found` → Make sure `~/go/bin` is in PATH: `export PATH=$PATH:~/go/bin`
> - Very few results → Add API keys to `~/.config/subfinder/provider-config.yaml` for more sources
> - Go not installed → `sudo apt install golang-go -y`

---

## 🔴 httpx (HTTP Toolkit)

> 💡 **Why This Matters**
> After finding subdomains, you need to know which are alive and what they're running. httpx probes each subdomain for HTTP/HTTPS, returning status codes, titles, and technology — in seconds.

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

#### 🧪 Try It Now — Pipeline: subfinder → httpx

```bash
# Full pipeline: find subdomains, then probe for live hosts
subfinder -d hackerone.com -silent 2>/dev/null | httpx -silent -status-code -title -tech-detect 2>/dev/null | head -10
```

> ✅ **Expected Output**
> ```
> https://hackerone.com [200] [HackerOne | ...] [Cloudflare,React]
> https://api.hackerone.com [200] [...] [Cloudflare]
> https://docs.hackerone.com [200] [...] [Cloudflare]
> ```

---

## 🔴 Amass (Attack Surface Mapping)

> 💡 **Why This Matters**
> Amass goes deeper than subfinder — it maps relationships between domains, finds associated IP ranges, and discovers infrastructure connections. For large targets, it's the most comprehensive tool but also the slowest.

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

> 🔧 **If Stuck**
> - Amass takes forever → Passive mode (`-passive`) is much faster. Brute force can take hours on large wordlists.
> - High memory usage → Amass is memory-intensive. Close other applications or use subfinder as an alternative.

---

## 🔴 ffuf (Fuzz Faster U Fool)

> 💡 **Why This Matters**
> ffuf is the Swiss Army knife of web fuzzing. It replaces `FUZZ` in any part of an HTTP request with wordlist entries — making it useful for directory discovery, parameter discovery, virtual host discovery, and even brute force attacks.

```bash
# Installation
go install github.com/ffuf/ffuf/v2@latest

# === DIRECTORY FUZZING ===
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -mc 200,301,302,401,403 \
  -o output.json -of json

# With extensions
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -e .php,.asp,.aspx,.jsp,.html,.js,.json,.bak,.txt,.xml,.sql,.zip,.old \
  -mc 200,301,302 -t 100

# Recursive scanning
ffuf -u https://target.com/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -recursion -recursion-depth 3

# === PARAMETER FUZZING ===
ffuf -u 'https://target.com/api/v1/resource?FUZZ=1' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200

# POST parameter discovery (JSON)
ffuf -u https://target.com/api/v1/resource \
  -X POST -H "Content-Type: application/json" \
  -d '{"FUZZ": "test"}' \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -mc 200

# === VHOST FUZZING ===
ffuf -u https://target.com \
  -H "Host: FUZZ.target.com" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 1234  # Filter default response size

# === FILTER OPTIONS ===
-fs 4242     # Filter by response size
-fw 100      # Filter by word count
-fl 10       # Filter by line count
-fr "not found"  # Filter by regex
-fc 403,404  # Filter by status code

# === AUTHENTICATION ===
-b "session=abc123"                # With cookies
-H "Authorization: Bearer TOKEN"   # With headers

# === SPEED CONTROL ===
-rate 100    # 100 req/sec
-t 200       # 200 threads
```

#### 🧪 Try It Now — Parameter Fuzzing on Juice Shop

```bash
# Discover hidden GET parameters on Juice Shop's search
ffuf -u "http://localhost:3000/rest/products/search?FUZZ=test" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200 -t 20 -s 2>/dev/null | head -10
```

> ✅ **Expected Output**
> ```
> q
> ```
> Only `q` is a valid parameter. But on a real application, parameter fuzzing might reveal hidden parameters like `debug`, `admin`, `internal`, etc.

---

## 🔴 Gobuster

```bash
# Installation
go install github.com/OJ/gobuster/v3@latest

# Directory brute force
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt \
  -t 50 -o gobuster.txt

# With extensions
gobuster dir -u https://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -x php,txt,html,bak,old,js,json -t 50

# DNS subdomain brute force
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 50

# VHOST brute force
gobuster vhost -u https://target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50

# With authentication
gobuster dir -u https://target.com -w wordlist.txt -c "session=abc123" -t 50
```

---

## 🔴 Nuclei (Template-Based Scanner)

> 💡 **Why This Matters**
> Nuclei runs 7000+ security check templates (CVEs, misconfigs, exposed panels, default credentials) against your targets. It finds "low-hanging fruit" instantly — things like exposed admin panels, known CVEs, and default credentials that would take hours to check manually.

```bash
# Installation
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Update templates (do this regularly)
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

#### 🧪 Try It Now — Nuclei Against Juice Shop

```bash
nuclei -u http://localhost:3000 -severity medium,high,critical -silent 2>/dev/null | head -10
```

> ✅ **Expected Output**
> ```
> [tech-detect:express] [http] [info] http://localhost:3000
> [robots-txt] [http] [info] http://localhost:3000/robots.txt
> [cors-misconfig] [http] [info] http://localhost:3000
> [x-powered-by-header] [http] [info] http://localhost:3000 [Express]
> ```
> Nuclei automatically detected: Express framework, robots.txt, CORS misconfiguration, and technology leak via X-Powered-By header!

> 🔧 **If Stuck**
> - `nuclei: command not found` → Install via Go or download binary: `go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest`
> - No templates → Run `nuclei -update-templates` first
> - Scan running slowly → Add `-rate-limit 50` and `-timeout 5`

---

## 🔴 Shodan (Internet-Wide Scanner)

> 💡 **Why This Matters**
> Shodan has already scanned the entire internet. Instead of scanning yourself (which is slow and generates logs), query Shodan for instant intelligence on your target's infrastructure — open ports, services, TLS certificates, and more.

```bash
# Installation
pip3 install shodan

# Initialize with API key (free tier available at shodan.io)
shodan init YOUR_API_KEY

# Search for a target
shodan host 104.26.10.78

# Search query
shodan search "hostname:target.com" --fields ip_str,port,org,hostnames

# Useful search queries:
# hostname:target.com           → All IPs associated with target
# ssl.cert.subject.cn:target.com → SSL certificates for target
# org:"Target Corp"             → All assets for the organization
# http.title:"Dashboard"        → Find exposed dashboards
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

> 💡 **Why This Matters**
> This is your "one command" recon pipeline. Run it at the start of every engagement — it combines subdomain discovery, live host detection, port scanning, directory fuzzing, and vulnerability scanning into a single script.

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

# 4. Nuclei scan
echo "[*] Running nuclei..."
nuclei -l $OUTPUT/live_hosts.txt -severity critical,high -o $OUTPUT/nuclei.txt 2>/dev/null

# 5. Summary
echo ""
echo "========== RECON SUMMARY =========="
echo "Subdomains found:    $(wc -l < $OUTPUT/all_subdomains.txt)"
echo "Live hosts:          $(wc -l < $OUTPUT/live_hosts.txt)"
echo "Unique IPs:          $(wc -l < $OUTPUT/ips.txt)"
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

# Custom wordlists from target website
cewl https://target.com -d 2 -m 5 -w custom_wordlist.txt
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

---

## 🧠 If You're Stuck With Tools

1. **Go tools won't install**: Ensure Go is installed (`sudo apt install golang-go -y`) and PATH includes `~/go/bin`: `export PATH=$PATH:~/go/bin` (add to `~/.bashrc` for persistence).
2. **subfinder/httpx returns nothing**: Check internet connectivity. Some tools need API keys for full coverage.
3. **Nmap requires root**: SYN scan (`-sS`) needs root. Use `sudo nmap` or switch to connect scan (`-sT`).
4. **ffuf too many results**: Use filter flags: `-fs SIZE` (filter size), `-fc 404,403` (filter codes), `-fw WORDS` (filter word count).
5. **Nuclei templates missing**: Run `nuclei -update-templates` before scanning.
6. **SecLists not found**: `sudo apt install seclists -y` or clone: `git clone https://github.com/danielmiessler/SecLists.git /usr/share/seclists`
