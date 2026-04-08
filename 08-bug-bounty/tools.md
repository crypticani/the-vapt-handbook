# 08 — Bug Bounty: Tools

> Your bug bounty toolkit — from recon to exploitation.

> 📋 **What You Will Do In This Section**
> - [ ] Install the complete bug bounty tool stack
> - [ ] Run subdomain enumeration with subfinder, amass, and crt.sh
> - [ ] Scan for vulnerabilities with nuclei and Dalfox
> - [ ] Set up out-of-band testing with interactsh
> - [ ] Configure monitoring and notifications with notify

---

## 🔴 Recon Tools

> 💡 **Why This Matters**
> Recon tools automate the discovery of attack surface — subdomains, URLs, technologies, and endpoints. The ProjectDiscovery suite (subfinder + httpx + nuclei) is the gold standard for bug bounty recon. Install these first.

```bash
# Subdomain enumeration
subfinder -d target.com -all -silent           # Fast, multiple sources
amass enum -passive -d target.com               # Comprehensive, slower
assetfinder --subs-only target.com              # Quick alternative

# Live host detection
httpx -silent -status-code -title -tech-detect < subs.txt

# URL gathering
waybackurls target.com                          # Archive.org URLs
gau target.com                                  # GetAllUrls (multiple sources)
katana -u https://target.com                    # Fast web crawler

# JavaScript endpoint extraction
cat urls.txt | grep '\.js$' | while read url; do
  curl -s "$url" | grep -oP '(\/api\/[a-zA-Z0-9/_-]+)'
done

# Or use LinkFinder:
python3 linkfinder.py -i https://target.com -d -o cli
```

#### 🧪 Try It Now — Recon Tool Verification

```bash
echo "=== Bug Bounty Recon Tools ==="
for tool in subfinder httpx nuclei ffuf waybackurls gau katana; do
  which $tool &>/dev/null && echo "  ✅ $tool" || echo "  ❌ $tool"
done
echo ""
echo "Install all (Go required):"
echo "  go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest"
echo "  go install github.com/projectdiscovery/httpx/cmd/httpx@latest"
echo "  go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest"
echo "  go install github.com/ffuf/ffuf/v2@latest"
```

---

## 🔴 Scanning Tools

```bash
# Nuclei — Template-based vulnerability scanner
nuclei -u https://target.com -severity critical,high
nuclei -l live_hosts.txt -t cves/ -severity critical
nuclei -l live_hosts.txt -as    # Auto-select templates based on tech

# Nikto — Web server scanner
nikto -h https://target.com

# Dalfox — XSS scanner with Burp integration
dalfox url "https://target.com/search?q=test" -b YOUR_XSS_HUNTER

# ParamSpider — Parameter discovery from web archives
python3 paramspider.py -d target.com
```

---

## 🔴 Exploitation Tools

> 💡 **Why This Matters**
> After discovering potential vulnerabilities through recon, these tools help verify and exploit them. Burp Suite is your primary manual testing tool. interactsh detects blind/out-of-band vulnerabilities that no scanner can find.

```bash
# Burp Suite — Primary tool for manual testing
# Start: java -jar burpsuite_community.jar

# SQLMap — SQL injection automation
sqlmap -u "https://target.com/search?q=test" --batch

# ffuf — Fuzz everything (directories, parameters, headers)
ffuf -u https://target.com/FUZZ -w common.txt

# jwt_tool — JWT analysis and attacks
python3 jwt_tool.py TOKEN

# interactsh — Out-of-band testing (detect SSRF, blind XSS, blind SQLi)
go install github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client
# Gives you unique URLs — use in SSRF/XSS payloads to detect callbacks
```

---

## 🔴 Monitoring Tools

```bash
# notify — Send alerts to Telegram/Slack/Discord
go install github.com/projectdiscovery/notify/cmd/notify@latest

# Configure ~/.config/notify/provider-config.yaml:
# telegram:
#   - id: your_chat_id
#     telegram_api_key: your_bot_token

# Full pipeline with notifications:
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
□ Amass (comprehensive subdomain enum)
□ interactsh (out-of-band testing)
□ jwt_tool (JWT attacks)
□ Dalfox (XSS specialization)
□ gau (URL gathering)
□ katana (web crawling)
□ notify (alerting)

BROWSER EXTENSIONS:
□ Wappalyzer (tech detection)
□ Cookie Editor
□ HackBar
□ FoxyProxy (proxy switching)
```

---

## 🧠 If You're Stuck

1. **Go tools not installing**: Install Go first: `sudo apt install golang-go`. Set `export PATH=$PATH:~/go/bin`
2. **nuclei templates outdated**: Update with `nuclei -update-templates`
3. **Burp not intercepting**: Check proxy settings (127.0.0.1:8080) and browser proxy configuration
4. **interactsh URL not getting callbacks**: Ensure the target can make outbound requests. Try on a SSRF-vulnerable test app first.
5. **Too many tools, don't know which to use**: Start with just subfinder + httpx + Burp. Add tools as you need them.
