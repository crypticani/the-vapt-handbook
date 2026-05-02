# 00 - Setup & Mindset: Methodology

## Pentesting Loop

```text
Enumerate -> Identify -> Exploit -> Escalate
```

- Enumerate: find hosts, ports, services, paths, users, and files.
- Identify: decide what each finding means and what is worth testing.
- Exploit: test the highest-value weakness with evidence.
- Escalate: expand access, privileges, or impact after initial foothold.

Repeat the loop until the scope is covered.

---

## Attack Surface Checklist

- [ ] Hosts and domains
- [ ] Open TCP ports
- [ ] UDP services
- [ ] Web apps and APIs
- [ ] Login panels
- [ ] File uploads
- [ ] Exposed files/backups
- [ ] Usernames and credentials
- [ ] Internal names and paths
- [ ] High-value services

---

## Quick Start Workflow

```text
1. Confirm scope.
2. Create notes.
3. Run recon.
4. List exposed services.
5. Pick the highest-value target.
6. Test manually.
7. Save evidence.
8. Decide next action.
```

Every step should produce either a finding, a ruled-out path, or a better question.

