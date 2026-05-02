# 00 - Setup & Mindset: Notes

## Attacker Mindset

Think in paths, not tools.

Ask:

- What can I reach?
- What can I control?
- What trusts this system?
- What data or access would matter?
- What is the shortest path to impact?

---

## Input -> Processing -> Output

Every target has this model:

```text
Input -> Processing -> Output
```

Test each part:

- Input: parameters, headers, uploads, forms, API bodies.
- Processing: auth, parsing, database queries, file handling, business logic.
- Output: errors, redirects, files, tokens, records, permissions.

Most vulnerabilities happen when attacker-controlled input reaches sensitive processing.

---

## Common Beginner Mistakes

- Running tools without a question.
- Skipping notes and losing findings.
- Stopping after the first open port.
- Trusting scanner output without manual checks.
- Ignoring boring-looking services.
- Exploiting before understanding the attack surface.
- Forgetting scope.

