# 02 - Recon & Enumeration: Methodology

Goal: move from an approved target to a usable attack surface map.

Pressure checklist:

- [ ] fast scan
- [ ] full scan
- [ ] service enum
- [ ] web enum
- [ ] attack surface notes

Core flow:

1. Run fast scan
2. Run detailed scan
3. Identify services
4. Enumerate each service
5. Build attack surface

Use this workflow only against systems you are authorized to test.

---

## Pressure Workflow

When time is limited, chain the workflow and make decisions from output.

```bash
TARGET=10.10.10.10

# 1. Fast scan: find obvious ports
nmap -T4 -F "$TARGET" -oA scans/fast

# 2. If open ports exist, run full TCP scan
nmap -p- -T4 "$TARGET" -oA scans/full

# 3. Extract open ports and run service detection
PORTS=$(grep -oP '\d+/open' scans/full.gnmap | cut -d/ -f1 | paste -sd, -)
nmap -sC -sV -p "$PORTS" "$TARGET" -oA scans/detailed
```

Decision rule:
- No open ports in fast scan -> still run full scan if the host is in scope.
- Open web port -> start web enum immediately while full scan runs.
- Unknown service -> banner grab and run targeted Nmap scripts.

---

## 1. Target Discovery

Start by confirming what is alive. Do this before heavy scanning so you do not waste time on dead hosts.

```bash
# Single host
ping -c 2 10.10.10.10

# Network sweep
nmap -sn 10.10.10.0/24 -oA scans/discovery

# Local network ARP discovery
sudo nmap -sn -PR 192.168.1.0/24 -oA scans/arp-discovery
```

Reasoning:
- `-sn` performs host discovery without port scanning.
- ARP discovery is reliable on local networks because firewalls often block ICMP but still respond to ARP.
- Save output with `-oA` so you have normal, grepable, and XML formats.

Create a live-host list:

```bash
grep "Up" scans/discovery.gnmap | cut -d " " -f2 > live-hosts.txt
```

---

## 2. Run Fast Scan

The fast scan finds obvious entry points quickly. It answers: "What should I inspect first?"

```bash
# Fast built-in scan
nmap -T4 -F 10.10.10.10 -oA scans/fast-F

# Fast TCP scan against one host
sudo nmap -sS -T4 --top-ports 1000 10.10.10.10 -oA scans/fast

# Fast scan against multiple hosts
sudo nmap -sS -T4 --top-ports 1000 -iL live-hosts.txt -oA scans/fast-all
```

Reasoning:
- `-sS` is a SYN scan and is fast when run as root.
- `-T4` is suitable for labs and stable networks.
- Top ports quickly reveal common services such as SSH, HTTP, SMB, RDP, and databases.
- `-F` scans fewer ports than the default scan. Use it for first contact, not final coverage.

If the target blocks ping:

```bash
sudo nmap -Pn -sS -T4 --top-ports 1000 10.10.10.10 -oA scans/fast-pn
```

Reasoning:
- `-Pn` treats the host as alive. Use it when a host is in scope but does not respond to host discovery.

---

## 3. Scan Strategy

Use scanning in layers. Each scan answers a different question.

### Time-Based Strategy

Quick recon for CTF or time-limited testing:
- Run `nmap -T4 -F`.
- Run `nmap -sC -sV` on open ports.
- If HTTP is open, run `whatweb`, `ffuf`, and login/API checks.
- Run full scan in the background if time allows.

Deep recon for real engagements:
- Confirm live hosts.
- Run fast scan, full TCP scan, detailed service scan, and UDP scan.
- Enumerate every open service, including non-standard ports.
- Validate scanner findings manually before exploitation.

### Fast Scan

```bash
nmap -T4 -F 10.10.10.10 -oA scans/fast
```

Use when:
- You need a first look within seconds.
- You are triaging many hosts.
- You want to identify obvious web, SSH, SMB, RDP, or database exposure.

Do not stop here. `-F` intentionally skips many ports.

### Full Scan

```bash
nmap -p- -T4 10.10.10.10 -oA scans/full
```

Use when:
- The host is confirmed alive.
- The fast scan is complete.
- You need confidence that services are not hiding on high ports.

This scan finds ports such as `8080`, `8443`, `9000`, `10000`, and custom admin services that default scans often miss.

### Detailed Scan

```bash
nmap -sC -sV -p 22,80,443,8080 10.10.10.10 -oA scans/detailed
```

Use when:
- You already know which ports are open.
- You need service versions, default script output, titles, headers, host keys, or SMB metadata.
- You are preparing service-specific enumeration.

Run detailed scans against open ports only. It is faster, cleaner, and less noisy.

---

## Output Interpretation

Read scan output as instructions for the next command.

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.54
443/tcp open ssl/http nginx
```

Actions:
- `PORT 80 open` -> HTTP found -> run `whatweb`, `curl -i`, `ffuf`, and check login/API paths.
- `PORT 443 open` -> HTTPS found -> inspect certificate, headers, redirects, and run web enum.
- `PORT 22 open` -> OpenSSH found -> check version, auth methods, weak/default creds only if allowed.
- `unknown open` -> run `nmap -sV --version-all`, banner grab with `nc`, and test if it speaks HTTP/TLS.

Example:

```bash
# 80/tcp open http
whatweb http://10.10.10.10/
ffuf -u http://10.10.10.10/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc all -fc 404

# 22/tcp open ssh
nmap -sV --script ssh2-enum-algos,ssh-hostkey -p 22 10.10.10.10

# 9999/tcp open unknown
nmap -sV --version-all -p 9999 10.10.10.10
nc -nv 10.10.10.10 9999
```

---

## 4. Run Detailed Scan

After the fast scan, scan every TCP port. Do not assume closed top ports means the host has no attack surface.

```bash
# Full TCP port scan
sudo nmap -sS -p- --min-rate 5000 10.10.10.10 -oA scans/full-tcp
```

Extract open ports:

```bash
grep -oP '\d+/open' scans/full-tcp.gnmap | cut -d/ -f1 | paste -sd, -
```

Run service detection only against open ports:

```bash
PORTS="22,80,443,445,8080"
nmap -sV -sC -p "$PORTS" 10.10.10.10 -oA scans/services
```

Reasoning:
- Full-port scan finds services moved to non-standard ports.
- `-sV` fingerprints service versions.
- `-sC` runs safe default NSE scripts for useful metadata.
- Running scripts only on open ports is faster and cleaner.

Add UDP coverage:

```bash
sudo nmap -sU --top-ports 20 10.10.10.10 -oA scans/udp-top20
sudo nmap -sU -p 53,67,68,69,123,135,137,138,161,162,500,514,520,1900 10.10.10.10 -oA scans/udp-targeted
```

Reasoning:
- UDP is slow, but missing UDP means missing DNS, SNMP, TFTP, NTP, and VPN services.
- Start with top UDP ports, then expand when the environment suggests it.

---

## 5. Identify Services

Convert scan output into a service table.

```text
Host          Port    Service    Version              Notes
10.10.10.10   22/tcp  ssh        OpenSSH 7.2p2         Check auth policy, weak creds if allowed
10.10.10.10   80/tcp  http       Apache 2.4.18         Enumerate web app
10.10.10.10   445/tcp smb        Samba 4.x             Enumerate shares/users
```

Confirm uncertain services manually:

```bash
nc -nv 10.10.10.10 22
curl -i http://10.10.10.10/
openssl s_client -connect 10.10.10.10:443 -servername target.local
```

Reasoning:
- Nmap guesses can be wrong on custom services.
- Manual banner grabbing verifies what the scanner reports.
- Version strings guide CVE research, but never exploit from version alone.

---

## 6. Enumerate Each Service

### SSH - 22/tcp

```bash
nmap -sV --script ssh2-enum-algos,ssh-hostkey -p 22 10.10.10.10 -oA scans/ssh
nc -nv 10.10.10.10 22
```

Look for:
- Product and version
- Password authentication enabled
- Weak algorithms
- Reused hostnames or usernames from other services

Do not brute force unless the rules of engagement allow it.

### HTTP/HTTPS - 80, 443, 8080, 8443

```bash
whatweb http://10.10.10.10
curl -i http://10.10.10.10/
nmap --script http-title,http-headers,http-methods -p 80,443,8080 10.10.10.10 -oA scans/http
```

Directory fuzzing:

```bash
ffuf -u http://10.10.10.10/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -mc all -fc 404 \
  -o scans/ffuf-common.json

gobuster dir -u http://10.10.10.10/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak,zip,old \
  -o scans/gobuster-dir.txt
```

Reasoning:
- Headers and titles reveal frameworks, servers, and redirects.
- Fuzzing finds hidden panels, backups, APIs, and static files.
- Extensions matter on PHP, IIS, Java, and legacy applications.

### SMB - 139/445

```bash
nmap --script smb-os-discovery,smb-enum-shares,smb-enum-users -p 139,445 10.10.10.10 -oA scans/smb
smbclient -L //10.10.10.10/ -N
enum4linux-ng -A 10.10.10.10
```

Look for:
- Anonymous shares
- Writable shares
- Usernames
- Host/domain name
- Old SMB versions

### FTP - 21/tcp

```bash
nmap --script ftp-anon,ftp-syst,ftp-bounce -p 21 10.10.10.10 -oA scans/ftp
ftp 10.10.10.10
```

Look for:
- Anonymous login
- Writable directories
- Backup files
- Application source code

### DNS - 53/tcp or 53/udp

```bash
dig @10.10.10.10 target.local A
dig @10.10.10.10 target.local AXFR
nmap --script dns-zone-transfer,dns-nsid -p 53 10.10.10.10 -oA scans/dns
```

Look for:
- Zone transfer
- Internal hostnames
- Subdomains
- Split-horizon DNS leaks

### SNMP - 161/udp

```bash
sudo nmap -sU --script snmp-info,snmp-interfaces,snmp-processes -p 161 10.10.10.10 -oA scans/snmp
snmpwalk -v2c -c public 10.10.10.10
```

Look for:
- Default community strings
- Running processes
- Network interfaces
- Installed software
- Usernames

### Databases

```bash
# MySQL
nmap --script mysql-info,mysql-empty-password -p 3306 10.10.10.10 -oA scans/mysql

# PostgreSQL
nmap --script pgsql-brute -p 5432 10.10.10.10 -oA scans/postgres

# MSSQL
nmap --script ms-sql-info,ms-sql-empty-password -p 1433 10.10.10.10 -oA scans/mssql
```

Look for:
- Exposed databases
- Empty/default passwords
- Version-specific weaknesses
- Database names leaked in banners or errors

---

## 7. Web Enumeration Workflow

Run this for every HTTP service, including non-standard ports.

```bash
URL="http://10.10.10.10:8080"

whatweb "$URL"
curl -i "$URL"
curl -s "$URL/robots.txt"
nikto -h "$URL" -output scans/nikto-8080.txt
ffuf -u "$URL/FUZZ" -w /usr/share/wordlists/dirb/common.txt -mc all -fc 404
```

Then check common high-value paths:

```bash
for path in robots.txt sitemap.xml .git/HEAD .env backup.zip admin login api swagger.json openapi.json; do
  curl -s -o /dev/null -w "%{http_code} %{size_download} $path\n" "$URL/$path"
done
```

Reasoning:
- `robots.txt` and `sitemap.xml` expose intended and hidden routes.
- `.git`, `.env`, backups, and API docs are high-value findings.
- Status code and response size help separate real content from soft 404s.

---

## Decision Trees

```text
IF port 80/443 open
  -> fingerprint with whatweb and curl
  -> check robots.txt, sitemap.xml, headers
  -> run ffuf for hidden directories and files
     ffuf -u http://TARGET/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc all -fc 404
  -> check login panels
     look for /login, /admin, /portal, /dashboard, /wp-admin, /manager
  -> test APIs
     look for /api, /api/v1, /swagger.json, /openapi.json, /graphql
  -> capture technologies, endpoints, login flows, and API docs
```

```text
IF SSH open
  -> check version
     nmap -sV -p 22 TARGET
     nc -nv TARGET 22
  -> check whether password auth is allowed
  -> collect usernames from web/SMB/SNMP
  -> test weak creds only if permitted
     try known/default creds from the app, docs, or ROE-approved list
  -> if creds work, document access level and move to post-auth enumeration
```

```text
IF unknown service open
  -> run nmap version detection
     nmap -sV --version-all -p PORT TARGET
  -> use nmap scripts for that port/service family
     nmap --script "default,safe" -p PORT TARGET
  -> banner grab manually
     nc -nv TARGET PORT
     curl -i http://TARGET:PORT/     # if it might be HTTP
     openssl s_client -connect TARGET:PORT
  -> search the banner, protocol hints, and response format
  -> do not ignore it just because nmap says unknown
```

```text
IF SMB open
  -> enumerate shares anonymously
  -> enumerate users and OS
  -> check readable/writable shares
  -> download only scoped files needed for evidence
```

```text
IF DNS open
  -> query records
  -> attempt zone transfer
  -> add discovered hosts to target list
  -> scan new hosts
```

```text
IF SNMP open
  -> try approved community strings
  -> dump system, process, interface, and user data
  -> use discovered hostnames/IPs to expand enumeration
```

---

## Build The Attack Surface

Your final recon output should be actionable.

```markdown
## Attack Surface

### Hosts
- 10.10.10.10 - Linux host, web + SSH + SMB

### Open Services
- 22/tcp SSH OpenSSH 7.2p2
- 80/tcp HTTP Apache 2.4.18
- 445/tcp SMB Samba 4.x

### Web Findings
- /admin - login panel
- /backup.zip - exposed backup
- /api/swagger.json - API documentation

### Credentials/Usernames
- usernames: alice, bob, svc_backup
- credential testing allowed: yes/no

### Priority Targets
1. Exposed backup file on HTTP
2. Anonymous SMB share
3. Outdated OpenSSH version

### Next Exploitation Steps
- Review backup contents for secrets
- Test SMB write access
- Search confirmed versions for known vulnerabilities
```

---

## Common Mistakes

- Scanning too fast and causing packet loss or unreliable results.
- Trusting only the first Nmap scan.
- Trusting default scan output as complete coverage.
- Skipping the full port scan.
- Missing UDP entirely.
- Missing hidden directories because only the homepage was checked.
- Ignoring services on non-standard ports.
- Skipping manual banner verification.
- Ignoring subdomains and virtual hosts.
- Running directory fuzzing without filtering soft 404s.
- Treating version numbers as proof of vulnerability.
- Not saving scan output.
- Not turning findings into a prioritized attack plan.

---

## What Next

Recon feeds exploitation by converting unknown targets into specific attack paths.

Use recon findings to decide exploitation priorities:
- Login panel found -> test default credentials, password policy, lockout behavior, and username enumeration if allowed.
- API docs found -> map endpoints, auth requirements, object IDs, file upload routes, and admin-only actions.
- Exposed backup or `.git` found -> inspect for credentials, source code, hidden routes, and deployment secrets.
- SMB share found -> check read/write access, scripts, config files, and reusable credentials.
- Confirmed vulnerable version found -> verify exploitability against the exact service, OS, and configuration.

Recon also feeds privilege escalation after initial access:
- Service versions identify local exploit candidates.
- Web config files reveal database credentials and internal hosts.
- SMB/NFS shares reveal scripts, backups, and user files.
- SNMP output can reveal processes, users, routes, and installed software.
- Hostnames and domains guide lateral movement and internal enumeration.

Good handoff format:

```markdown
## Ready For Exploitation
- Target:
- Confirmed entry point:
- Evidence:
- Required credentials:
- Known users:
- Relevant files/paths:
- Risk:
- Next command/test:
```
