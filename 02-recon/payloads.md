# 02 - Recon & Enumeration: Payloads

Recon payloads should help discover content, not exploit it.

Keep this file focused on wordlists, path patterns, and fuzzing patterns.

---

## Wordlists

Common locations:

```text
/usr/share/wordlists/dirb/common.txt
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
```

Install SecLists if missing:

```bash
sudo apt install seclists -y
```

---

## Directory Fuzzing Patterns

Basic web paths:

```text
admin
login
dashboard
portal
api
api/v1
api/v2
uploads
files
backup
backups
dev
test
staging
old
tmp
private
```

Run with `ffuf`:

```bash
ffuf -u http://TARGET/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -mc all -fc 404
```

Run with `gobuster`:

```bash
gobuster dir -u http://TARGET/ \
  -w /usr/share/wordlists/dirb/common.txt
```

---

## File Extension Patterns

Use extensions when the technology suggests them.

```text
php
aspx
asp
jsp
txt
bak
old
zip
tar.gz
sql
conf
config
json
xml
```

`ffuf`:

```bash
ffuf -u http://TARGET/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .php,.txt,.bak,.old,.zip,.conf,.json \
  -mc all -fc 404
```

`gobuster`:

```bash
gobuster dir -u http://TARGET/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak,old,zip,conf,json
```

---

## High-Value Web Checks

Use these as a small manual checklist or custom wordlist.

```text
robots.txt
sitemap.xml
.git/HEAD
.git/config
.env
.env.local
.env.production
backup.zip
backup.tar.gz
backup.sql
db.sql
dump.sql
config.php
config.json
web.config
server-status
phpinfo.php
info.php
swagger.json
swagger-ui.html
openapi.json
api-docs
graphql
graphiql
actuator
actuator/env
actuator/health
```

Quick check:

```bash
URL="http://TARGET"
for path in robots.txt sitemap.xml .git/HEAD .env backup.zip swagger.json openapi.json api-docs graphql actuator/health; do
  curl -s -o /dev/null -w "%{http_code} %{size_download} $path\n" "$URL/$path"
done
```

---

## Vhost Fuzzing Patterns

Common vhost names:

```text
www
admin
portal
vpn
mail
webmail
api
dev
test
staging
uat
beta
internal
intranet
jenkins
gitlab
grafana
monitoring
support
docs
```

`ffuf`:

```bash
ffuf -u http://TARGET_IP/ \
  -H "Host: FUZZ.target.local" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs DEFAULT_SIZE
```

`gobuster`:

```bash
gobuster vhost -u http://target.local/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain
```

---

## Parameter Fuzzing Patterns

Common parameter names:

```text
id
user
uid
page
file
path
url
redirect
next
return
search
q
query
debug
token
key
```

Example:

```bash
ffuf -u "http://TARGET/page.php?FUZZ=test" \
  -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc all -fs DEFAULT_SIZE
```
