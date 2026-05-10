# 01 - Fundamentals

This module teaches the operating model for modern application testing.

Use it to understand how data moves through:

- SPA frontends
- REST and GraphQL APIs
- token-based auth
- ORMs and serializers
- object storage
- webhooks and async workers
- cloud-connected backends

## Outcomes

By the end of this module you should be able to:

- capture and replay requests with Burp and `curl`
- fingerprint frontend and backend technologies
- map trust boundaries across browser, API, worker, and storage
- tell the difference between object-level, function-level, and field-level authorization
- recognize when an ORM or query builder changes the likely bug pattern
- build a practical test plan before moving to recon or exploitation

## Recommended Order

1. [notes.md](notes.md)
2. [methodology.md](methodology.md)
3. [tools.md](tools.md)
4. [labs.md](labs.md)
5. [payloads.md](payloads.md)

## Focus Areas

- request and response analysis
- architecture mapping
- auth boundary mapping
- API inventory awareness
- storage and webhook abuse paths
- first-touch payload selection

## Lab Targets

- OWASP Juice Shop for browser/API flow and JWT handling
- DVWA for legacy mechanics and defense progression

Treat both as practice surfaces, not as the definition of modern production architecture.

## What To Carry Forward

After this module, you should be able to look at a target and answer:

- where can user input enter?
- where does identity get enforced?
- what data objects matter?
- what service trusts upstream data too much?
- what would count as evidence?
