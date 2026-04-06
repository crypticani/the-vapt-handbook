# 02 — Recon & Enumeration: Payloads

> Recon payloads are not attack payloads — they're probes designed to discover, enumerate, and fingerprint. They help you understand what you're up against before you attack.

> 📋 **What You Will Do In This Section**
> - [ ] Attempt DNS zone transfers and brute force subdomains
> - [ ] Check a target for 25+ sensitive file paths
> - [ ] Run Google dork queries to find exposed information
> - [ ] Probe HTTP headers for WAF bypass and information leakage
> - [ ] Use Shodan dorks to discover exposed services

---

## 🔴 DNS Enumeration Payloads

> 💡 **Why This Matters**
> DNS is the directory of the internet. A misconfigured DNS server (especially one allowing zone transfers) gives you the complete map of a target's infrastructure in one query — every subdomain, every IP, every service. Zone transfer is the single highest-value passive recon finding.

### Zone Transfer
```bash
# Attempt zone transfer against all nameservers
for ns in $(dig target.com NS +short); do
  echo "=== Trying $ns ==="
  dig axfr @$ns target.com
done
```

#### 🧪 Try It Now — Zone Transfer Test

```bash
# Test against a known vulnerable DNS server (zonetransfer.me is designed for practicing)
echo "=== Zone Transfer: zonetransfer.me ==="
dig axfr @nsztm1.digi.ninja zonetransfer.me 2>/dev/null | head -25
```

> ✅ **Expected Output**
> ```
> zonetransfer.me.    7200  IN  SOA nsztm1.digi.ninja. ...
> zonetransfer.me.    300   IN  A   5.196.105.14
> _acme-challenge.zonetransfer.me. 301 IN TXT "..."
> email.zonetransfer.me.  7200  IN  A   74.125.206.26
> info.zonetransfer.me.   7200  IN  A   74.125.206.26
> office.zonetransfer.me. 7200  IN  A   40.101.26.242
> ...
> ```
> **This is a jackpot!** A successful zone transfer reveals ALL DNS records — every subdomain, mail server, and internal service. On a real target, this would be a critical finding.

> 🔧 **If Stuck**
> - "Connection refused" → The DNS server doesn't allow zone transfers (most don't). This is expected — it's still worth trying.
> - `dig: command not found` → `sudo apt install dnsutils -y`

### DNS Record Brute Force

```bash
# Common subdomains to check (use as wordlist or test manually)
www mail ftp vpn remote admin portal webmail 
staging dev test qa uat prod pre-prod
api api-v1 api-v2 api-gateway graphql
jenkins gitlab github ci cd deploy build
grafana prometheus kibana elastic monitoring
db database mysql postgres redis cache
cdn static assets media images
sso auth login identity oauth
docs wiki help support
shop store cart checkout payment
beta canary internal intranet
```

---

## 🔴 Web Discovery Payloads

> 💡 **Why This Matters**
> Sensitive files are deployed accidentally all the time. A `.env` file contains database credentials. A `.git/HEAD` file lets you download the entire source code. A `swagger.json` documents every API endpoint. These payloads systematically check for common mistakes.

### Sensitive File Paths

```
# Version Control
.git/HEAD
.git/config
.svn/entries
.svn/wc.db
.hg/store/00manifest.i
.bzr/README

# Environment & Config
.env
.env.local
.env.production
.env.development
.env.backup
.env.old
config.yml
config.yaml
config.json
app.config
web.config
database.yml
settings.py
application.properties

# Backup Files
backup.sql
backup.zip
backup.tar.gz
db.sql
dump.sql
database.sql.bak
site_backup.zip
www.zip

# Debug & Admin
debug
trace
actuator
actuator/env
actuator/health
actuator/configprops
_debugbar
phpinfo.php
info.php
server-info
server-status
elmah.axd
elmah

# API Documentation
swagger.json
swagger-ui.html
api-docs
openapi.json
openapi.yaml
graphql
graphiql
altair
playground
__schema

# WordPress
wp-config.php
wp-config.php.bak
wp-config.php.old
wp-config.php.save
wp-config.php~
xmlrpc.php
wp-json/wp/v2/users

# Package Files
package.json
composer.json
Gemfile
requirements.txt
Pipfile
go.mod

# CI/CD
.gitlab-ci.yml
Jenkinsfile
.circleci/config.yml
.github/workflows
Dockerfile
docker-compose.yml

# Cloud
.aws/credentials
.aws/config
firebase.json
.firebaserc
serverless.yml
```

#### 🧪 Try It Now — Scan Juice Shop for Sensitive Files

```bash
echo "=== Sensitive File Scan: Juice Shop ==="
for file in .git/HEAD .env .env.bak robots.txt package.json swagger.json \
  api-docs graphql .well-known/security.txt sitemap.xml \
  config.json app.config Dockerfile docker-compose.yml \
  server-status debug trace actuator actuator/env \
  wp-config.php .DS_Store backup.sql backup.zip; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/$file")
  size=$(curl -s "http://localhost:3000/$file" | wc -c)
  if [ "$code" != "404" ] && [ "$code" != "000" ]; then
    echo "  ⚠️  $file → HTTP $code ($size bytes)"
  fi
done
```

> ✅ **Expected Output**
> ```
>   ⚠️  robots.txt → HTTP 200 (28 bytes)
>   ⚠️  package.json → HTTP 200 (large)
>   ⚠️  ftp → HTTP 200 (11000+ bytes)
> ```
> **Findings:**
> - `robots.txt` → Reveals disallowed paths (hidden content)
> - `package.json` → Reveals all dependencies and their versions (search for CVEs!)
> - `/ftp` → Open directory with downloadable files (information disclosure)

### Interesting HTTP Methods
```http
OPTIONS / HTTP/1.1
TRACE / HTTP/1.1
PUT / HTTP/1.1
DELETE / HTTP/1.1
PATCH / HTTP/1.1
PROPFIND / HTTP/1.1
MOVE / HTTP/1.1
COPY / HTTP/1.1
```

#### 🧪 Try It Now — Test HTTP Methods

```bash
echo "=== HTTP Method Testing: Juice Shop ==="
for method in GET POST PUT DELETE PATCH OPTIONS TRACE; do
  code=$(curl -s -o /dev/null -w "%{http_code}" -X $method http://localhost:3000/)
  echo "  $method → $code"
done
```

> ✅ **Expected Output**
> ```
>   GET → 200
>   POST → 200
>   PUT → 200
>   DELETE → 200
>   PATCH → 200
>   OPTIONS → 204
>   TRACE → 200
> ```
> Multiple methods returning `200` on the same endpoint means the application isn't restricting methods properly. `OPTIONS` returning `204` reveals CORS preflight support.

### Virtual Host Discovery
```
# Headers to use for vhost discovery
Host: admin.target.com
Host: internal.target.com
Host: dev.target.com
Host: staging.target.com
Host: api.target.com
Host: localhost
Host: 127.0.0.1
Host: target.com.evil.com
```

---

## 🔴 Google Dork Payloads

> 💡 **Why This Matters**
> Google has indexed millions of pages — including pages that companies have since deleted or forgotten about. Google dorks use advanced search operators to find exposed files, admin panels, test environments, and leaked credentials without ever touching the target.

```
# Complete dork collection for a target

# Exposed files
site:target.com filetype:sql
site:target.com filetype:bak
site:target.com filetype:log
site:target.com filetype:env
site:target.com filetype:xml
site:target.com filetype:json
site:target.com filetype:yml
site:target.com filetype:conf
site:target.com filetype:cfg
site:target.com filetype:ini
site:target.com filetype:txt "password"
site:target.com filetype:csv
site:target.com filetype:xls

# Admin and login panels
site:target.com inurl:admin
site:target.com inurl:login
site:target.com inurl:signin
site:target.com inurl:dashboard
site:target.com inurl:portal
site:target.com inurl:panel
site:target.com inurl:manage
site:target.com inurl:console

# Information disclosure
site:target.com intitle:"Index of"
site:target.com intitle:"Apache2 Ubuntu Default Page"
site:target.com intitle:"phpMyAdmin"
site:target.com "Directory Listing For /"
site:target.com "Not Found" "debug" 
site:target.com inurl:server-status
site:target.com inurl:server-info

# API endpoints
site:target.com inurl:api
site:target.com inurl:graphql
site:target.com inurl:swagger
site:target.com inurl:rest
site:target.com inurl:v1 OR inurl:v2

# Subdomains
site:*.target.com -www
site:*.target.com -www -mail

# Code & secrets
site:github.com "target.com" password
site:github.com "target.com" api_key
site:github.com "target.com" secret
site:github.com "target.com" token
site:github.com "target.com" AWS_ACCESS_KEY
site:github.com "@target.com"
site:pastebin.com "target.com"
site:trello.com "target.com"

# Cloud storage
site:s3.amazonaws.com "target"
site:blob.core.windows.net "target"
site:storage.googleapis.com "target"
```

---

## 🔴 Shodan Dorks

> 💡 **Why This Matters**
> Shodan has already port-scanned the entire internet. Instead of scanning your target (which is slow and leaves logs), query Shodan for instant intelligence — open ports, services, TLS certificates, exposed dashboards, and default credentials.

```
# Target-specific
hostname:target.com
ssl.cert.subject.cn:target.com
org:"Target Organization Name"

# Technology hunting
http.title:"Dashboard" hostname:target.com
http.title:"Jenkins" hostname:target.com
http.title:"Grafana" hostname:target.com
http.title:"Kibana" hostname:target.com
product:"Apache" hostname:target.com
product:"nginx" hostname:target.com

# Exposed services
port:9200 hostname:target.com          # Elasticsearch
port:27017 hostname:target.com         # MongoDB
port:6379 hostname:target.com          # Redis
port:5432 hostname:target.com          # PostgreSQL
port:11211 hostname:target.com         # Memcached
port:2375 hostname:target.com          # Docker API
port:5900 hostname:target.com          # VNC
port:3389 hostname:target.com          # RDP

# General hunting
"default password" country:US
http.html:"admin" http.html:"password" hostname:target.com
```

---

## 🔴 Nmap Script Arguments

```bash
# HTTP enumeration with custom options
nmap --script http-enum --script-args http-enum.basepath=/api/ target.com

# HTTP brute force paths
nmap --script http-brute --script-args 'http-brute.path=/login,userdb=users.txt,passdb=pass.txt' target.com

# DNS brute force with custom wordlist
nmap --script dns-brute --script-args 'dns-brute.domain=target.com,dns-brute.hostlist=subs.txt'

# SMB enumeration
nmap --script smb-enum-shares,smb-enum-users --script-args 'smbuser=guest,smbpass=' target.com

# MySQL enumeration
nmap --script mysql-enum,mysql-brute -p 3306 target.com

# SSL analysis
nmap --script ssl-enum-ciphers,ssl-cert,ssl-heartbleed -p 443 target.com
```

---

## 🔴 HTTP Header Probes

> 💡 **Why This Matters**
> Headers manipulate how the server routes and processes your request. Spoofing `X-Forwarded-For` can bypass IP-based restrictions. Manipulating the `Host` header can access internal virtual hosts. Probing `Origin` headers reveals CORS misconfiguration. These are passive-aggressive probes — they test defenses without attacking.

### Information Gathering Headers
```http
# Send these in requests to elicit different server behavior:

# WAF / IP bypass
X-Forwarded-For: 127.0.0.1
X-Real-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
True-Client-IP: 127.0.0.1
CF-Connecting-IP: 127.0.0.1
Forwarded: for=127.0.0.1

# CORS probing
Origin: https://evil.com
Origin: null
Origin: https://target.com.evil.com

# Host header manipulation
Host: admin.target.com
Host: localhost
X-Forwarded-Host: evil.com
X-Host: evil.com

# Content type switching
Content-Type: application/json
Content-Type: application/xml
Content-Type: application/x-www-form-urlencoded
Content-Type: multipart/form-data
```

#### 🧪 Try It Now — Header Probing on Juice Shop

```bash
echo "=== CORS Probe ==="
curl -sI -H "Origin: https://evil.com" http://localhost:3000/ | grep -i "access-control"

echo ""
echo "=== X-Forwarded-For ==="
curl -s -H "X-Forwarded-For: 127.0.0.1" http://localhost:3000/rest/products/search?q=test -o /dev/null -w "Status: %{http_code}\n"

echo ""
echo "=== Content-Type Switch ==="
curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/xml" \
  -d '<login><email>test</email><password>test</password></login>' \
  -o /dev/null -w "Status: %{http_code}\n"
```

> ✅ **Expected Output**
> ```
> === CORS Probe ===
> Access-Control-Allow-Origin: *
>
> === X-Forwarded-For ===
> Status: 200
>
> === Content-Type Switch ===
> Status: 500
> ```
> **Findings:**
> - CORS allows ANY origin (`*`) — cross-origin data theft possible
> - X-Forwarded-For is accepted — could bypass IP restrictions
> - XML content type causes `500` — XML parsing might be happening (XXE potential?)

---

## 📋 Recon Payload Checklist

```
For every target:
□ DNS zone transfer attempted
□ Subdomain wordlist brute force completed
□ Sensitive files checked (25+ paths)
□ Google dorks run (15+ queries)
□ GitHub searched for secrets
□ Shodan queried
□ HTTP methods tested
□ Virtual hosts enumerated
□ API documentation endpoints checked
□ WAF detection performed
□ CORS configuration probed
□ Header manipulation tested
```

---

## 🧠 If You're Stuck

1. **Zone transfer always fails**: This is expected on 99% of targets. It's still worth trying — when it works, it's a jackpot. Move on to other subdomain techniques.
2. **Google dorks return nothing**: The target might have good SEO hygiene. Try broader queries or check GitHub/Pastebin instead.
3. **Sensitive file scan finds nothing**: The presence of files depends on the technology. A Node.js app won't have `wp-config.php`. Adjust your file list to match the detected technology stack.
4. **Not sure what to do with findings**: Each finding maps to a next step — `robots.txt` → check disallowed paths. `package.json` → search for vulnerable packages. `.git/HEAD` → download the repo with git-dumper.
