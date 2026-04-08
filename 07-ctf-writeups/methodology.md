# 07 — CTF Writeups: Methodology

> How to systematically approach any CTF challenge and document your solution.

> 📋 **What You Will Do In This Section**
> - [ ] Follow a 6-step framework for any CTF challenge
> - [ ] Use time-boxing to avoid getting stuck on one problem
> - [ ] Apply the "If Stuck" checklist when progress stalls
> - [ ] Write writeups that teach your future self

---

## 🔴 The Framework

> 💡 **Why This Matters**
> In a 48-hour CTF, the difference between solving 5 challenges and 15 is methodology — not skill. Time management, triage, and systematic enumeration are what separate winning teams from frustrated ones.

```
STEP 1: UNDERSTAND (2 min)
  → Read the challenge description CAREFULLY. What's the goal?
  → Note the flag format: flag{...}, CTF{...}, picoCTF{...}
  → What files/URLs are provided?
  → Look for hints in the title, description, or tags

STEP 2: ENUMERATE (5 min)
  → Gather ALL available information. Don't attack yet.
  → Web: curl, view source, robots.txt, cookies
  → File: file, strings, binwalk, exiftool
  → Binary: checksec, strings, ltrace
  → PCAP: tshark, strings

STEP 3: HYPOTHESIZE (2 min)
  → What's the likely vulnerability?
  → What's the intended path? (consider the category and point value)
  → Have you seen something similar before?

STEP 4: TEST (time-boxed: 30 min max)
  → Validate your hypothesis
  → If it doesn't work in 15 min → re-hypothesize
  → If it doesn't work in 30 min → MOVE ON

STEP 5: EXPLOIT (varies)
  → Craft the exploit. Capture the flag.
  → If the flag format doesn't match → you might have partial success

STEP 6: DOCUMENT (5 min)
  → Write the writeup while it's fresh
  → Include: commands, outputs, key insight, what you learned
  → Future-you will thank present-you
```

---

## 🔴 CTF Time Management

```
COMPETITION START:
├── First 15 min: Read ALL challenge descriptions
├── Sort by: Point value (low → high)
├── Identify: Which challenges match YOUR strong categories
├── Mark: "Must try" (3-5 challenges)
│
├── First pass (2 hours): Solve easy challenges
│   └── Time-box: 15 min per easy challenge
│
├── Second pass (2 hours): Medium challenges
│   └── Time-box: 30 min per medium challenge
│
├── Third pass: Hard challenges + unsolved mediums
│   └── Time-box: 60 min per hard challenge
│
└── Last hour: Double-check all submitted flags
```

#### 🧪 Try It Now — Triage Practice

```bash
echo "=== CTF Triage Template ==="
echo ""
echo "Challenge 1: [Name] [Points] [Category]"
echo "  Difficulty estimate: Easy/Medium/Hard"
echo "  Time budget: 15/30/60 min"
echo "  Priority: 1-5"
echo ""
echo "When triaging, ask:"
echo "  1. Is this in my strong category? → Higher priority"
echo "  2. Low points = easier = solve first"
echo "  3. Many solves = easier = solve first"
echo "  4. Zero solves = skip until round 3"
```

---

## 🔴 Category-Specific Methodology

### Web Challenges
```
1. curl -sI TARGET → Check headers (Server, X-Powered-By)
2. curl -s TARGET | grep -i "flag\|hidden\|comment" → Easy wins
3. Check /robots.txt, /.git/, /.env, /admin/
4. Intercept with Burp → Modify requests
5. Test SQLi on any input field
6. Check cookies → Decode, modify, resend
```

### Binary/Pwn Challenges
```
1. file binary → What type? (ELF, PE, script?)
2. checksec binary → What protections? (NX, ASLR, canary?)
3. strings binary → Any hardcoded flags or hints?
4. ltrace/strace binary → What system calls?
5. Open in Ghidra / GDB → Find the vulnerability
6. Write exploit with pwntools
```

### Crypto Challenges
```
1. Identify the encoding/cipher
2. Try simple decodings: Base64 → Hex → ROT13
3. If RSA: Check if n is small → factordb.com
4. If XOR: Try single-byte brute force
5. If custom cipher: Analyze the source code
6. Use CyberChef for complex transformations
```

### Forensics Challenges
```
1. file + exiftool → What is this file really?
2. strings | grep flag → Brute force easy win
3. binwalk -e → Extract embedded files
4. If image: steghide/zsteg → Hidden data
5. If PCAP: strings + tshark HTTP filter
6. If memory dump: volatility → hashdump/filescan
```

---

## 🔴 If You're Stuck (Escalation Checklist)

```
LEVEL 1 (spent 5 min):
  → Re-read the challenge description — there's a hint you missed
  → Check the page source / file metadata
  → Google the exact error message

LEVEL 2 (spent 15 min):
  → Try the simplest possible attack
  → Change your approach — is this a different vulnerability type?
  → Look at the challenge tags/category more carefully

LEVEL 3 (spent 30 min):
  → Look for similar writeups from past CTFs (Google: "CTF web login bypass writeup")
  → Ask for hints in the CTF Discord (if allowed)
  → Take a 5-minute break and come back fresh

LEVEL 4 (spent 60 min):
  → MOVE ON to a different challenge
  → Come back later with fresh eyes
  → After the CTF: read the official writeup to learn
```

---

## 🧠 If You're Stuck

1. **First CTF, no solves**: That's completely normal. Focus on learning, not winning.
2. **Know the vuln but can't exploit**: Google the exact technique. CTF writeups are publicly available.
3. **Flag format doesn't match**: Double-check — some CTFs use `CTF{...}`, others use `flag{...}`.
4. **Team communication**: Share discoveries immediately. Use a shared document.
5. **Burnout during long CTF**: Take breaks. Eat food. Sleep. Fresh eyes solve more problems.
