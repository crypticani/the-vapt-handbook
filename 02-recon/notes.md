# 02 - Recon & Enumeration: Notes

## What Is Enumeration?

Enumeration is the process of turning a target into specific, testable facts:

- live hosts
- open ports
- service names
- service versions
- web paths
- users
- shares
- technologies
- trust relationships

The output is not "Nmap found ports." The output is an attack surface.

---

## Passive vs Active Recon

Passive recon does not directly touch the target.

Examples:
- search engines
- certificate transparency
- public DNS records
- GitHub searches
- Wayback Machine
- Shodan/Censys data

Active recon directly interacts with the target.

Examples:
- port scanning
- service detection
- directory fuzzing
- vhost fuzzing
- banner grabbing
- Nikto scans

Use passive recon first, then active recon inside the approved scope.

---

## Attack Surface Concept

Attack surface means every place an attacker can interact with the target.

Examples:
- SSH login
- web login page
- exposed API
- SMB share
- admin panel
- database port
- upload form
- forgotten staging host
- old backup file

Good enumeration reduces guessing. Each finding should answer:

- What is exposed?
- What version or technology is it?
- Can users interact with it?
- Does it leak files, users, paths, or credentials?
- What should be tested next?

---

## Decision Trees

```text
IF port 80/443 open
  -> run whatweb
  -> inspect headers
  -> check robots.txt and sitemap.xml
  -> run ffuf/gobuster
  -> review login, upload, admin, API, and backup paths
```

```text
IF SSH open
  -> check version
  -> capture banner
  -> collect usernames from other services
  -> test weak creds only if allowed
```

```text
IF SMB open
  -> list shares
  -> test anonymous access
  -> enumerate users
  -> check readable/writable shares
```

```text
IF UDP 161 open
  -> test approved SNMP community strings
  -> enumerate processes, users, interfaces, and routes
```

---

## Common Mistakes

- scanning too fast and missing ports
- stopping after top 1000 TCP ports
- missing UDP
- ignoring subdomains
- ignoring virtual hosts
- skipping non-standard web ports
- not filtering soft 404 responses
- trusting scanner output without manual verification
- failing to save raw output
- collecting data without ranking attack paths
