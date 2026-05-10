# VAPT Handbook Improvement Plan

## Goal

Refactor this handbook from a mostly classic web-pentest playbook into a modern, practical application security training path:

- less repeated theory
- more realistic labs
- better coverage of current stacks, APIs, auth, cloud, and supply chain
- less PHP-centric framing
- clearer progression from recon to exploitation to reporting

---

## Executive Summary

The repo is strong on execution style, command density, and beginner momentum. It already does a good job of getting someone into Burp, `curl`, Docker labs, and attacker thinking quickly.

The main issues are:

1. Modern technology coverage is inconsistent.
2. Too much content is repeated across `notes.md`, `methodology.md`, `tools.md`, and `labs.md`.
3. Several exercises still center on older app patterns such as DVWA/PHP upload shells while modern production targets are API-first, token-based, cloud-backed, and ORM-heavy.
4. Some modules are large but not well prioritized, which makes the learning path feel broader than it is current.

The handbook should move from:

- "classic vulnerable web app exploitation"

to:

- "modern application attack paths across web, API, auth, cloud, CI/CD, and business logic"

---

## What The Repo Does Well

- Strong hands-on tone in the root [README](README.md).
- Good beginner-friendly use of Juice Shop and Burp.
- Solid operational framing in [01-fundamentals/methodology.md](01-fundamentals/methodology.md).
- Good attacker mindset language in [00-setup-and-mindset/notes.md](00-setup-and-mindset/notes.md).
- Useful cloud direction already exists in [06-cloud-security](06-cloud-security).

Keep those strengths. Update the target selection and teaching structure.

---

## Main Gaps Found In The Current Repo

### 1. Placeholder module entry points

Every module README after the root is effectively a stub:

- [01-fundamentals/README.md](01-fundamentals/README.md)
- [02-recon/README.md](02-recon/README.md)
- [03-web-exploitation/README.md](03-web-exploitation/README.md)
- [04-privilege-escalation/README.md](04-privilege-escalation/README.md)
- [05-post-exploitation/README.md](05-post-exploitation/README.md)
- [06-cloud-security/README.md](06-cloud-security/README.md)
- [07-ctf-writeups/README.md](07-ctf-writeups/README.md)
- [08-bug-bounty/README.md](08-bug-bounty/README.md)

Impact:

- navigation feels unfinished
- module outcomes are hidden
- the repo does not communicate a modern curriculum at a glance

### 2. Fundamentals is good, but too broad and repetitive

The fundamentals section is large:

- [01-fundamentals/notes.md](01-fundamentals/notes.md)
- [01-fundamentals/labs.md](01-fundamentals/labs.md)
- [01-fundamentals/tools.md](01-fundamentals/tools.md)

Problems:

- repeated checklists and repeated "try it now" structure
- some concepts appear in multiple files with minor variation
- practical modern web/API architecture is mentioned, but not taught as a first-class path

Example of underused modern topics already present but not elevated:

- WebSockets
- microservice trust boundaries
- Sequelize error leakage
- frontend framework fingerprinting

### 3. Modern ORM-backed app behavior is barely covered

This is the gap you noticed correctly.

Current state:

- ORM is mostly referenced as remediation, not as an attack surface
- one explicit ORM mention appears in [03-web-exploitation/methodology.md](03-web-exploitation/methodology.md)
- `SequelizeDatabaseError` appears as observed output in [01-fundamentals/labs.md](01-fundamentals/labs.md)

What is missing:

- how modern ORMs change exploitation patterns
- unsafe raw query fallbacks inside otherwise "safe" ORM stacks
- injection through sort/filter builders
- mass assignment / over-posting
- N+1 and object property authorization leaks
- ORM-generated GraphQL/REST layers

### 4. Web exploitation still overweights classic PHP-era targets

A lot of web exploitation and post-exploitation content still assumes:

- file upload -> PHP shell
- classic `.php` endpoints
- DVWA-style request handling
- Apache/IIS/legacy behaviors as the main path

Examples:

- [03-web-exploitation/labs.md](03-web-exploitation/labs.md)
- [03-web-exploitation/notes.md](03-web-exploitation/notes.md)
- [05-post-exploitation/payloads.md](05-post-exploitation/payloads.md)
- [05-post-exploitation/notes.md](05-post-exploitation/notes.md)

This content is still useful, but it should no longer dominate the learning path.

### 5. API security should be a primary track, not a side effect of web testing

The repo touches APIs, but modern API security needs its own structure.

Missing or underweighted:

- OWASP API Top 10 driven testing
- BOLA/BFLA/property-level auth
- pagination, filtering, and bulk action abuse
- webhooks and callback validation
- inventory drift across `/api`, `/v1`, `/internal`, `/partner`
- unsafe third-party API consumption
- rate-limit and resource consumption testing

### 6. Modern auth is underrepresented

Current auth coverage is heavily JWT-focused, which is useful, but incomplete.

Need stronger coverage for:

- OAuth 2.0 / OIDC flows
- token exchange and refresh token abuse
- device code and magic link flows
- multi-tenant SSO mistakes
- passkeys / WebAuthn
- MFA bypass patterns
- session boundaries across SPA + API + mobile

### 7. Cloud is good but AWS-centric

[06-cloud-security](06-cloud-security) is one of the stronger sections, but it is still heavily centered on:

- AWS IAM
- S3
- EC2 metadata

Keep that, but expand modern cloud-native attack surfaces:

- GitHub Actions / CI secrets
- container registry exposure
- Kubernetes workload identity
- serverless event abuse
- IaC drift and state exposure
- multi-cloud patterns at a conceptual level

### 8. Business logic and real-world exploitation paths need more weight

The repo teaches vulnerability classes well. It should teach product abuse better.

Add more focus on:

- account takeover chains
- coupon / wallet / credit abuse
- checkout and payment edge cases
- invite / organization / tenant boundary flaws
- webhook poisoning
- race conditions
- approval workflow bypass

---

## What To Remove Or Compress

These are the "boring theory" areas to reduce.

### Remove repetition, not fundamentals

Cut repeated versions of:

- generic HTTP anatomy explanations
- generic request/response checklists
- repeated "why this matters" paragraphs
- duplicated Burp / `curl` setup guidance
- repeated expected-output blocks when they teach the same observation

### Downgrade older patterns from core path to appendix

Move these into an "older but still relevant" appendix instead of core learning:

- PHP web shell upload as a headline exercise
- null-byte PHP bypass discussion as a major lesson
- old IIS/Apache parser quirks as default examples
- narrow focus on classic monolith login forms

### Shrink theory-only sections

If a concept does not change what the learner does in Burp, terminal, browser DevTools, cloud CLI, or a lab target, shorten it.

Rule:

- 20% concept
- 80% testing workflow, evidence, exploit chain, reporting

---

## What To Add

## 1. A New "Modern App Architecture" layer in Fundamentals

Add a new first-class fundamentals block:

- SPA + API + CDN + identity provider + object storage
- BFF and microservice patterns
- API gateway and service-to-service trust
- queue and async processing surfaces
- pre-signed URLs
- webhooks
- WebSockets / GraphQL subscriptions

Practical lab objectives:

- map the full request path from browser -> API -> worker -> storage
- identify where auth is enforced versus assumed
- find user-controlled data that reaches downstream services

## 2. ORM-Specific attack content

Add a dedicated subsection: `ORMs, Query Builders, and Data Access Bugs`

Cover:

- Prisma raw queries
- Sequelize raw query misuse
- SQLAlchemy text/raw execution
- Hibernate/JPA query mistakes
- EF Core raw SQL and mass assignment / over-posting issues

Practical labs:

- safe ORM endpoint vs unsafe raw query fallback
- filter/sort injection through query builders
- over-posting role field in JSON body
- hidden admin-only properties exposed through generic serializers

## 3. API Security as a top-level rewrite priority

Restructure [03-web-exploitation](03-web-exploitation) to become:

- Web and API Exploitation

Put these near the top:

- BOLA / IDOR
- function-level auth
- property-level auth
- mass assignment
- resource exhaustion
- webhook SSRF
- API inventory drift
- unsafe consumption of partner APIs

## 4. Modern auth module content

Add explicit practical coverage for:

- OAuth authorization code flow
- PKCE
- OIDC ID token vs access token misuse
- refresh token theft / replay
- email magic link abuse
- invite token abuse
- passkeys / WebAuthn basics for testers
- MFA recovery flow bypass

## 5. Modern bug bounty target models

Update [08-bug-bounty](08-bug-bounty) around real present-day target patterns:

- SPA frontends with JSON APIs
- mobile backends
- GraphQL gateways
- staging + preview deployments
- CI/CD artifacts
- public buckets / object storage
- webhook endpoints
- AI/LLM features where applicable

## 6. Supply chain and CI/CD security

Expand either [06-cloud-security](06-cloud-security) or add a new section:

- GitHub Actions secrets leakage
- poisoned artifacts
- exposed package registries
- container image secrets
- dependency confusion concepts
- Terraform state and deployment pipeline abuse

## 7. AI/LLM surface as an optional modern module

Not core before APIs/auth/cloud, but worth adding after the main refresh:

- prompt injection
- insecure tool use
- data exfiltration through connected systems
- output handling issues

Use this as an advanced appendix or new module once the handbook is modernized elsewhere.

---

## Recommended New Lab Stack

Replace the current lab mix of "mostly Juice Shop + DVWA" with a broader lab portfolio.

### Keep

- Juice Shop for beginner web concepts
- DVWA for controlled classic vulnerability mechanics

### Add

1. A modern Node.js app
- Express or NestJS
- Prisma or Sequelize
- JWT + refresh token flow
- REST + WebSocket endpoint

2. A modern Python API
- FastAPI
- SQLAlchemy
- OpenAPI/Swagger exposure
- background task or webhook processing

3. A Java or .NET API
- Spring Boot + Hibernate/JPA
or
- ASP.NET Core + EF Core

4. A cloud-native lab target
- object storage
- signed URLs
- CI secret leak
- IAM role misuse

5. A GraphQL target
- schema discovery
- resolver-level auth bugs
- nested query abuse

### Minimum viable stack for this repo

If you only add one new target, make it:

- a REST API with ORM + JWT + role model + file upload to object storage + webhook callback

That one target can teach:

- IDOR
- mass assignment
- raw query injection
- auth bugs
- SSRF
- object storage mistakes
- business logic abuse

---

## Proposed Structural Changes

## Root README

Update [README.md](README.md) to show:

- classic track
- modern app/API track
- cloud track
- bug bounty track

Also replace "DVWA + Juice Shop only" as the entire default lab story.

## Module format

Current pattern:

- `notes.md`
- `methodology.md`
- `tools.md`
- `labs.md`
- `payloads.md`

Recommended pattern:

1. `README.md`
- outcomes
- prerequisites
- modern target types
- time estimate

2. `playbook.md`
- concise mental model
- testing workflow

3. `labs.md`
- main learning asset

4. `tooling.md`
- only practical commands

5. `findings.md`
- example report-ready findings

This reduces overlap and makes labs the center of learning.

---

## Rewrite Priority By Section

## Phase 1: Highest ROI

1. [README.md](README.md)
2. [01-fundamentals](01-fundamentals)
3. [03-web-exploitation](03-web-exploitation)
4. [08-bug-bounty](08-bug-bounty)

Reason:

- these sections define the learner's mental model
- this is where modern app/API trends should be most visible

## Phase 2: High value

5. [06-cloud-security](06-cloud-security)
6. [02-recon](02-recon)

Reason:

- recon should emphasize API, cloud, preview envs, CI, and hidden inventory
- cloud should connect directly to app exploit chains

## Phase 3: Keep but rebalance

7. [04-privilege-escalation](04-privilege-escalation)
8. [05-post-exploitation](05-post-exploitation)
9. [07-ctf-writeups](07-ctf-writeups)

Reason:

- useful, but less urgent for making the repo feel current

---

## Section-by-Section Improvement Plan

## 00 - Setup & Mindset

Keep:

- attacker mindset
- input -> processing -> output model

Add:

- "modern target model" diagram
- testing notes template
- evidence collection template

## 01 - Fundamentals

Cut:

- duplicate theory
- repeated request anatomy explanations

Add:

- SPA/API architecture mapping
- auth boundary mapping
- ORM/query-builder basics for testers
- API versioning and inventory drift
- object storage and signed URL basics
- WebSocket and webhook basics

Outcome:

- learner can map a modern app, not just a classic website

## 02 - Recon

Add stronger focus on:

- subdomain + preview deployment discovery
- OpenAPI / Swagger / GraphQL discovery
- mobile API endpoint collection
- JS endpoint extraction
- hidden parameter and undocumented route discovery
- asset relationships across CDN, object storage, CI, and cloud hosts

## 03 - Web Exploitation

Rename in practice to:

- Web and API Exploitation

Move to top:

- IDOR / BOLA
- authZ bugs
- mass assignment
- property-level exposure
- filter/sort/search abuse
- webhook SSRF
- GraphQL auth failures

Keep but demote:

- PHP upload shell path
- legacy parser bypasses

## 04 - Privilege Escalation

Keep most content.

Add:

- container-aware privesc flow
- Kubernetes node/pod escape decision tree
- secrets-to-cloud escalation path

## 05 - Post Exploitation

Reduce old PHP-webshell emphasis.

Add:

- cloud credential harvesting
- workload identity abuse
- service token reuse
- secrets manager abuse
- lateral movement through CI/CD and internal APIs

## 06 - Cloud Security

Keep:

- AWS IAM
- S3
- EC2 metadata
- Kubernetes

Add:

- GitHub Actions / pipeline abuse
- registry and artifact attacks
- workload identity
- serverless event abuse
- Terraform/Helm/Kustomize review tips
- cloud-to-app and app-to-cloud exploit chains

## 07 - CTF Writeups

Keep concise.

Shift emphasis from:

- broad CTF motivation

to:

- writeups that mirror real pentest evidence
- short reproducible attack-chain reports

## 08 - Bug Bounty

Add practical hunting paths for:

- preview/staging environments
- mobile APIs
- GraphQL
- webhooks
- multi-tenant SaaS
- password reset / invite / checkout / credits abuse
- unsafe file processing flows
- CI leaks and object storage exposures

---

## Suggested New Labs

Add labs that look like real production behavior:

1. Mass assignment lab
- change hidden `role`, `plan`, `tenantId`, or `isAdmin`

2. ORM raw query lab
- app uses ORM for most routes, but one report/search route falls back to raw SQL

3. Property-level auth lab
- normal users can see admin-only JSON fields by changing query params

4. Webhook SSRF lab
- attacker controls callback URL

5. Signed URL lab
- object storage upload/download abuse

6. OAuth/OIDC lab
- token audience confusion, redirect abuse, invite flow issues

7. GraphQL resolver auth lab
- schema is visible, field auth is weak

8. CI/CD secret leak lab
- secrets in logs, artifacts, or workflow history

9. Tenant escape lab
- org/user/project IDs leak across boundaries

10. Business logic lab
- coupon reuse, wallet double-spend, race condition, approval bypass

---

## Practical Editorial Rules For The Rewrite

Apply these rules across the repo:

1. Every theory section must end in a lab or repeatable test.
2. Every tool section must answer: when do I use this, against what, and what signal proves progress?
3. Every vulnerability class must include:
- detection
- validation
- exploit chain
- evidence
- remediation note

4. Prefer one realistic lab over five generic payload lists.
5. Prefer API requests, JSON bodies, token flows, and cloud-linked targets over old form-post-only examples.
6. Keep legacy examples, but label them as legacy or niche.

---

## 30-Day Rewrite Plan

## Week 1

- rewrite root [README.md](README.md)
- expand all module `README.md` files from placeholders into real entry points
- redesign [01-fundamentals](01-fundamentals) around modern app architecture

## Week 2

- rewrite [03-web-exploitation](03-web-exploitation) into web + API exploitation
- add ORM-specific material
- add modern auth coverage

## Week 3

- update [02-recon](02-recon) and [08-bug-bounty](08-bug-bounty)
- add preview env, API inventory, JS endpoint, and GraphQL recon flows

## Week 4

- extend [06-cloud-security](06-cloud-security)
- rebalance [05-post-exploitation](05-post-exploitation)
- move legacy-heavy content into appendices or clearly marked optional sections

---

## External References For The Modern Refresh

Use these as primary anchors while updating content:

- OWASP API Security Top 10 2023: https://owasp.org/API-Security/
- Prisma ORM docs: https://docs.prisma.io/docs/orm
- SQLAlchemy ORM docs: https://docs.sqlalchemy.org/20/orm/
- Hibernate ORM docs: https://hibernate.org/orm/
- Entity Framework Core docs: https://learn.microsoft.com/en-us/ef/core/
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications
- WebAuthn / passkeys guidance: https://web.dev/articles/passkey-registration

---

## Recommended First Edits

If updating this repo incrementally, start here:

1. Replace placeholder module READMEs.
2. Rewrite fundamentals around modern app architecture.
3. Add a dedicated ORM/API auth section.
4. Demote PHP upload-shell labs from core path to legacy appendix.
5. Add one modern end-to-end lab target and rewrite web exploitation around it.

That will change the repo's perceived quality more than any smaller cleanup.
