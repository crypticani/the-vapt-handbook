# 08 — Bug Bounty: Tools

> Your bug bounty toolkit — from recon to exploitation.

---

## 🔴 Recon Tools

```bash
# Subdomain enumeration
subfinder -d target.com -all -silent
amass enum -passive -d target.com
assetfinder --subs-only target.com

# Live host detection
httpx -silent -status-code -title -tech-detect < subs.txt

# URL gathering
waybackurls target.com
gau target.com           # GetAllUrls — combines multiple sources
katana -u https://target.com  # Fast web crawler

# JavaScript analysis
# Extract endpoints from JS files:
cat urls.txt | grep '\.js$' | while read url; do
  curl -s "$url" | grep -oP '(\/api\/[a-zA-Z0-9/_-]+)'
done

# Or use: LinkFinder
python3 linkfinder.py -i https://target.com -d -o cli
```

## 🔴 Scanning Tools

```bash
# Nuclei
nuclei -u https://target.com -severity critical,high
nuclei -l live_hosts.txt -t cves/ -severity critical

# Nikto
nikto -h https://target.com

# Dalfox (XSS scanner)
dalfox url "https://target.com/search?q=test" -b YOUR_XSS_HUNTER

# ParamSpider (parameter discovery from web archives)
python3 paramspider.py -d target.com
```

## 🔴 Exploitation Tools

```bash
# Burp Suite — Primary tool for manual testing
# SQLMap — SQL injection automation
# ffuf — Fuzzing everything
# jwt_tool — JWT attacks
# SSRFmap — SSRF exploitation
# Commix — Command injection

# Collaborator alternatives (for OOB testing)
# Burp Collaborator (Pro)
# interactsh: go install github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client
# Gives you a unique URL — use it to detect SSRF, blind XSS, blind SQLi
```

## 🔴 Monitoring Tools

```bash
# Subdomain monitoring
# sublert — monitors new subdomains
# notify — sends alerts to Telegram/Slack/Discord

# Install notify
go install github.com/projectdiscovery/notify/cmd/notify@latest

# Configure ~/.config/notify/provider-config.yaml:
# telegram:
#   - id: your_chat_id
#     telegram_api_key: your_bot_token

# Pipeline with notifications
subfinder -d target.com -silent | httpx -silent | nuclei -severity critical,high | notify -provider telegram
```

---

## 📋 Bug Bounty Tool Stack

```
ESSENTIAL (install first):
□ Burp Suite Community Edition
□ subfinder + httpx + nuclei (ProjectDiscovery suite)
□ ffuf
□ waybackurls
□ SQLMap
□ Python3 + requests

RECOMMENDED:
□ Amass
□ interactsh
□ jwt_tool
□ Dalfox
□ gau
□ katana
□ notify

BROWSER EXTENSIONS:
□ Wappalyzer (tech detection)
□ Cookie Editor
□ HackBar
□ FoxyProxy (proxy switching)
```
