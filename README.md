# The VAPT Handbook

An execution-focused handbook for learning application security, web/API testing, cloud abuse paths, and reporting with a pentester mindset.

This repo is not meant to be read like theory notes. It is meant to be used while testing:

- capture traffic
- trace trust boundaries
- change one thing at a time
- verify impact
- write evidence as you go

## What This Handbook Covers

The current curriculum is organized around a modern application attack path:

1. modern app fundamentals
2. recon and inventory
3. web and API exploitation
4. privilege escalation
5. post-exploitation
6. cloud and CI/CD attack surface
7. documentation and writeups
8. bug bounty workflows

The core learning shift in this repo is:

- from classic web forms only
- to SPA + API + token auth + object storage + webhook + cloud-backed systems

Legacy targets and techniques still exist here, but they should support the learning path, not define it.

## Recommended Learning Tracks

### Track 1: Modern App Pentesting

Start here if your goal is current web and API work.

- [00-setup-and-mindset](00-setup-and-mindset/README.md)
- [01-fundamentals](01-fundamentals/README.md)
- [02-recon](02-recon/README.md)
- [03-web-exploitation](03-web-exploitation/README.md)
- [08-bug-bounty](08-bug-bounty/README.md)

### Track 2: Infrastructure and Cloud Follow-Through

Use this after you are comfortable with application testing.

- [04-privilege-escalation](04-privilege-escalation/README.md)
- [05-post-exploitation](05-post-exploitation/README.md)
- [06-cloud-security](06-cloud-security/README.md)

### Track 3: Practice and Reporting

- [07-ctf-writeups](07-ctf-writeups/README.md)
- [08-bug-bounty](08-bug-bounty/README.md)

## Lab Philosophy

Every module should answer:

- what is the target architecture?
- where does user-controlled data enter?
- where is auth enforced?
- what downstream system trusts this data?
- what evidence proves impact?

Prefer realistic workflows over memorizing long payload lists.

## Baseline Lab Stack

### Keep Using

- OWASP Juice Shop: modern-ish browser/API workflow and easy repetition
- DVWA: controlled legacy vulnerability mechanics and defense progression
- PortSwigger Web Security Academy: deeper web exploit practice

### Treat As Core Concepts

Even when practicing on Juice Shop or DVWA, think in terms of:

- SPA frontend
- REST or GraphQL API
- JWT, session, OAuth, or WebAuthn auth flows
- ORM-backed data access
- object storage and pre-signed URLs
- webhooks and async workers
- cloud metadata and workload identity

## Quick Start

### 1. Verify prerequisites

```bash
docker --version
python3 --version
curl --version
```

### 2. Launch baseline lab targets

```bash
docker run -d -p 3000:3000 --name juiceshop bkimminich/juice-shop
docker run -d -p 80:80 --name dvwa vulnerables/web-dvwa
```

### 3. Verify targets

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:80
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected baseline:

- Juice Shop: `200`
- DVWA: `302`

### 4. Install core tools

```bash
sudo apt install nmap ffuf nikto sqlmap whatweb -y
```

Burp Suite Community Edition is still the main manual testing tool:

- https://portswigger.net/burp/communitydownload

## Repository Structure

```text
the-vapt-handbook/
├── 00-setup-and-mindset
├── 01-fundamentals
├── 02-recon
├── 03-web-exploitation
├── 04-privilege-escalation
├── 05-post-exploitation
├── 06-cloud-security
├── 07-ctf-writeups
└── 08-bug-bounty
```

Most modules contain:

- `README.md`: scope, outcomes, study order
- `notes.md`: mental models and attack surfaces
- `methodology.md`: testing workflow
- `tools.md`: practical tool use
- `labs.md`: repeatable hands-on exercises
- `payloads.md`: first-touch detection and exploit payloads

## How To Study A Module

Use this order:

1. `README.md`
2. `notes.md`
3. `methodology.md`
4. `tools.md`
5. `labs.md`
6. `payloads.md`

Keep notes while working:

- request
- mutation
- response delta
- conclusion
- evidence

## Current Modernization Priorities

This repo is being updated toward current application security work. The priority areas are:

- fundamentals for SPA/API/cloud-backed apps
- API authorization and business logic testing
- ORM/query-builder attack surface
- modern auth flows
- cloud and CI/CD connected exploit paths

The repo-level plan for this work lives in [IMPROVEMENT-PLAN.md](IMPROVEMENT-PLAN.md).

## Recommended Platforms

- PortSwigger Web Security Academy: https://portswigger.net/web-security
- TryHackMe: https://tryhackme.com/
- Hack The Box: https://www.hackthebox.com/
- PentesterLab: https://pentesterlab.com/
- VulnHub: https://www.vulnhub.com/

## Simple Rules

- Do not run tools without a question.
- Do not trust one response; compare baselines.
- Do not stop at reflected data; trace it to the sink.
- Do not treat auth as one check; test it per object, function, and field.
- Do not separate app bugs from cloud bugs when the app can reach the cloud.

## Legal

Use this repository only for authorized testing and education. Never test systems without explicit written permission.

## License

MIT
