# 02 — Recon & Enumeration: Payloads

> Recon payloads are not attack payloads — they're probes designed to discover, enumerate, and fingerprint. They help you understand what you're up against before you attack.

---

## 🔴 DNS Enumeration Payloads

### Zone Transfer
```bash
# Attempt zone transfer against all nameservers
for ns in $(dig target.com NS +short); do
  echo "=== Trying $ns ==="
  dig axfr @$ns target.com
done
```

### DNS Record Brute Force
```bash
# Common subdomains to check
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

### Information Gathering Headers
```http
# Send these in requests to elicit different server behavior:

# WAF bypass
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
```
