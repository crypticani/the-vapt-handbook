# 02 - Recon & Enumeration: Tools

Use tools to answer specific questions:

- What hosts are alive?
- What ports are open?
- What services and versions are exposed?
- What web paths, files, and virtual hosts exist?
- What should be attacked first?

---

## nmap

Purpose: host discovery, port scanning, service detection, and safe script-based enumeration.

Use when:
- You have an IP, hostname, or subnet in scope.
- You need to identify open TCP/UDP ports.
- You need service names, versions, and basic script results.

### Fast Scan

```bash
sudo nmap -sS -T4 --top-ports 1000 10.10.10.10 -oA scans/fast
```

When to use:
- First active scan against a host.
- Quick triage during a CTF, lab, or time-boxed engagement.

Example output:

```text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
445/tcp open  microsoft-ds
```

### Full Scan

```bash
sudo nmap -sS -p- --min-rate 5000 10.10.10.10 -oA scans/full-tcp
```

When to use:
- After the fast scan.
- When top ports are empty but the host is alive.
- Before declaring a host low-value.

Example output:

```text
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
8080/tcp  open  http-proxy
49152/tcp open  unknown
```

### Service Detection

```bash
nmap -sV -sC -p 22,80,445,8080 10.10.10.10 -oA scans/services
```

When to use:
- After identifying open ports.
- Before researching vulnerabilities.
- Before service-specific enumeration.

Example output:

```text
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp   open  http        Apache httpd 2.4.18
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu
8080/tcp open  http        Jetty 9.4.z
```

### UDP Scan

```bash
sudo nmap -sU --top-ports 20 10.10.10.10 -oA scans/udp-top20
```

When to use:
- On every real assessment after TCP triage.
- When you suspect DNS, SNMP, TFTP, NTP, or VPN exposure.

Example output:

```text
PORT    STATE         SERVICE
53/udp  open          domain
161/udp open          snmp
500/udp open|filtered isakmp
```

### Targeted NSE Scripts

```bash
# HTTP
nmap --script http-title,http-headers,http-methods -p 80,443,8080 10.10.10.10

# SMB
nmap --script smb-os-discovery,smb-enum-shares,smb-enum-users -p 139,445 10.10.10.10

# SSH
nmap --script ssh2-enum-algos,ssh-hostkey -p 22 10.10.10.10

# SSL/TLS
nmap --script ssl-cert,ssl-enum-ciphers -p 443,8443 10.10.10.10
```

When to use:
- After service detection confirms the service.
- When you need structured details before manual testing.

---

## ffuf

Purpose: fast web fuzzing for directories, files, parameters, and virtual hosts.

Use when:
- HTTP/HTTPS is open.
- You need hidden content.
- You need to test vhosts or parameters.

### Directory Fuzzing

```bash
ffuf -u http://10.10.10.10/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -mc all -fc 404 \
  -o scans/ffuf-common.json
```

When to use:
- First pass against a web service.
- Good for finding `/admin`, `/backup`, `/uploads`, `/api`, `/dev`.

### Extension Fuzzing

```bash
ffuf -u http://10.10.10.10/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .php,.txt,.bak,.old,.zip \
  -mc all -fc 404
```

When to use:
- PHP/IIS/legacy apps.
- When source backups or old files are likely.

### Virtual Host Fuzzing

```bash
ffuf -u http://10.10.10.10/ \
  -H "Host: FUZZ.target.local" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 1234
```

When to use:
- The web server serves different sites based on the `Host` header.
- DNS is incomplete or internal names are suspected.

Tip:
- Use `-fs`, `-fw`, or `-fl` to filter the default response size, words, or lines.
- Always inspect a few negative responses before trusting filters.

---

## gobuster

Purpose: directory, file, DNS, and vhost brute forcing.

Use when:
- You want a simple, reliable alternative to `ffuf`.
- You need readable output during manual testing.

### Directory Fuzzing

```bash
gobuster dir -u http://10.10.10.10/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -o scans/gobuster-common.txt
```

When to use:
- Quick content discovery on any web service.

### Directory Fuzzing With Extensions

```bash
gobuster dir -u http://10.10.10.10/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak,old,zip,conf \
  -o scans/gobuster-ext.txt
```

When to use:
- You see PHP, Apache, IIS, Tomcat, or old application patterns.

### Vhost Fuzzing

```bash
gobuster vhost -u http://10.10.10.10/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -o scans/gobuster-vhost.txt
```

When to use:
- A host returns a default site but may contain hidden virtual hosts.

---

## whatweb

Purpose: web technology fingerprinting.

Use when:
- HTTP/HTTPS is open.
- You need quick stack identification before deeper testing.

Commands:

```bash
# Basic fingerprint
whatweb http://10.10.10.10/

# More detail
whatweb -a 3 http://10.10.10.10/

# Multiple URLs
whatweb -i urls.txt --log-brief scans/whatweb.txt
```

Example output:

```text
http://10.10.10.10 [200 OK] Apache[2.4.18], Bootstrap, Country[RESERVED][ZZ], HTML5, IP[10.10.10.10], PHP[7.0.33], Title[Admin Portal]
```

When to use:
- Before directory fuzzing.
- Before CVE research.
- When deciding which wordlists/extensions to use.

---

## nikto

Purpose: web server misconfiguration and known-file checks.

Use when:
- You have permission for noisy active scanning.
- You want quick checks for default files, dangerous methods, outdated server banners, and common misconfigurations.

Commands:

```bash
# Basic scan
nikto -h http://10.10.10.10/ -output scans/nikto.txt

# HTTPS with non-standard port
nikto -h https://10.10.10.10:8443/ -output scans/nikto-8443.txt

# Tune for interesting files and misconfigurations
nikto -h http://10.10.10.10/ -Tuning x6 -output scans/nikto-files.txt
```

Example output:

```text
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ /server-status: This reveals Apache status information.
+ /backup/: Directory indexing found.
```

When to use:
- After confirming a web service exists.
- Before manual web testing.
- As a checklist, not as proof. Manually verify every interesting result.

---

## Tool Selection

```text
Need live hosts       -> nmap -sn
Need open ports       -> nmap fast scan, then full scan
Need service versions -> nmap -sV -sC
Need web tech stack   -> whatweb
Need web paths        -> ffuf or gobuster
Need web misconfigs   -> nikto
Need vhosts           -> ffuf/gobuster vhost mode
Need UDP exposure     -> nmap -sU
```
