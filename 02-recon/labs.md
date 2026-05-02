# 02 - Recon & Enumeration: Labs

Use these labs to practice the loop:

```text
Scan -> Identify -> Enumerate -> Prepare for exploitation
```

Do not jump to exploitation until your attack surface is documented.

---

## TryHackMe Rooms

### Nmap

Practice:
- Host discovery
- TCP scans
- UDP scans
- Service detection
- NSE scripts

Expected outcome:
- You can choose between fast, full, service, and UDP scans.
- You can explain what each flag does.
- You can produce a clean open-port table.

### Network Services

Practice:
- SMB enumeration
- FTP enumeration
- NFS enumeration
- SMTP basics

Expected outcome:
- You can enumerate common network services without relying on a vulnerability scanner.
- You can identify anonymous access, shares, banners, and user leaks.

### Network Services 2

Practice:
- NFS
- SMTP
- MySQL
- service-specific enumeration commands

Expected outcome:
- You can move from an open port to service-specific findings.
- You can record evidence that supports later exploitation.

### Web Scanning

Practice:
- Web fingerprinting
- Content discovery
- Basic scanner validation
- Manual verification

Expected outcome:
- You can use `whatweb`, `nikto`, `ffuf`, and browser inspection together.
- You can separate real findings from scanner noise.

### OWASP Juice Shop

Practice:
- Web route discovery
- API discovery
- Client-side file review
- Authentication surface mapping

Expected outcome:
- You can find exposed paths such as `/ftp`, API routes, JavaScript files, and login flows.
- You can build a web attack surface map before testing vulnerabilities.

### Vulnversity

Practice:
- Nmap scanning
- Gobuster directory discovery
- Web service enumeration

Expected outcome:
- You can scan a target, identify HTTP, discover upload/admin paths, and prepare exploitation notes.

---

## Hack The Box Easy Machines

### Lame

Practice:
- Full TCP scanning
- SMB version detection
- FTP anonymous checks

Expected outcome:
- You can identify exposed legacy services.
- You can document version-based leads without exploiting immediately.

### Blue

Practice:
- Windows host enumeration
- SMB scanning
- NSE script usage

Expected outcome:
- You can enumerate SMB safely and identify high-risk Windows exposure.
- You can distinguish service detection from exploit validation.

### Jerry

Practice:
- Web service discovery on non-standard ports
- Tomcat fingerprinting
- Default path checks

Expected outcome:
- You can recognize Apache Tomcat, locate management interfaces, and record credential-testing opportunities.

### Nibbles

Practice:
- Web enumeration
- Directory brute forcing
- Application fingerprinting

Expected outcome:
- You can discover hidden application paths and identify the CMS or app stack.

### Bashed

Practice:
- Web content discovery
- Interesting file identification
- Attack surface prioritization

Expected outcome:
- You can find exposed tools or files and rank them above low-value version findings.

### Legacy

Practice:
- Windows service enumeration
- SMB-focused scanning
- Risk identification from old services

Expected outcome:
- You can identify outdated Windows exposure and prepare a concise exploitation plan.

---

## Lab Deliverable Template

For every room or machine, produce this:

```markdown
## Target
- Name:
- IP:
- Scope:

## Scan Summary
- Fast scan:
- Full TCP scan:
- UDP scan:
- Service detection:

## Services
| Port | Service | Version | Enumeration Done | Notes |
|---|---|---|---|---|
| 22/tcp | ssh | OpenSSH x.x | ssh-hostkey, banner | Password auth enabled |
| 80/tcp | http | Apache x.x | whatweb, ffuf, nikto | /admin found |

## Attack Surface
- Web paths:
- Credentials/usernames:
- Shares/files:
- Interesting versions:
- High-value findings:

## Next Steps
1. Verify high-value web findings manually.
2. Test allowed credential paths.
3. Research confirmed service versions.
```

Success criteria:
- Every open port has an enumeration note.
- Every web service has content discovery results.
- UDP was considered.
- Findings are ranked by exploitation value.
