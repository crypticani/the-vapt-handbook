# 07 — CTF Writeups: Labs

> 📋 **What You Will Do In This Section**
> - [ ] Complete a TryHackMe room and write a full writeup
> - [ ] Solve 5 PicoCTF challenges across different categories
> - [ ] Compete in a live CTF from CTFtime
> - [ ] Complete a HackTheBox machine from scan to root

---

## 🧪 Lab 1: Complete a TryHackMe Room

### Objective
Complete a structured CTF room and write your first full writeup.

> 💡 **Why This Matters**
> TryHackMe rooms are guided — they teach you the technique AND let you practice it. This is the perfect starting point because you get hints when stuck. The writeup practice builds report-writing muscle memory.

### Steps

```
1. Sign up at https://tryhackme.com (free)

2. Start with ONE of these rooms:
   ├── "OWASP Top 10" (free) — Web vulnerabilities
   ├── "Linux PrivEsc" (free) — Privilege escalation (pairs with Day 4)
   ├── "Web Fundamentals" (free) — HTTP, cookies, authentication
   └── "Nmap" (free) — Network scanning basics

3. Complete every task in the room

4. Write a full writeup using the template from notes.md
   └── Save as: writeups/tryhackme-room-name.md

5. Include:
   ├── Every command you ran
   ├── Every flag you captured
   ├── What you learned
   └── What surprised you
```

> ✅ **Expected Output**
> A complete walkthrough of the TryHackMe room with every answer, every command, and a "Lessons Learned" section.

> 🔧 **If Stuck**
> - Room requires VPN → Download the OpenVPN config from TryHackMe and connect: `sudo openvpn tryhackme.ovpn`
> - Can't reach the target → Ensure VPN is connected: `ip addr show tun0`
> - Machine not responding → Try deploying a fresh machine (green button)

### ✅ Success Criteria
- [ ] TryHackMe account created
- [ ] Room selected and completed
- [ ] Full writeup written using the template
- [ ] All flags captured

---

## 🧪 Lab 2: Solve 5 PicoCTF Challenges

### Objective
Solve challenges across multiple categories to build breadth.

> 💡 **Why This Matters**
> PicoCTF is designed for beginners but teaches real techniques. Solving 5 challenges across different categories exposes you to Web, Crypto, Forensics, and General Skills — all of which you'll encounter in real CTFs and pentests.

### Steps

```
1. Go to https://picoctf.org (free, no VPN needed)

2. Solve 5 challenges — pick from different categories:
   ├── 1x Web Exploitation
   ├── 1x Cryptography
   ├── 1x Forensics
   ├── 1x General Skills
   └── 1x Any category (your choice)

3. For EACH challenge, write a mini-writeup in this format:
```

```markdown
### [Challenge Name] — [Points]
**Category**: Web / Crypto / Forensics / etc.
**Key Insight**: [One sentence — what was the trick?]
**Commands Used**:
```bash
# exact commands here
```
**Flag**: `picoCTF{...}`
**What I Learned**: [One sentence]
```

> 🔧 **If Stuck**
> - Web challenges → Always view source first, check cookies, try basic SQLi
> - Crypto → Use CyberChef, try Base64/hex/ROT13 decoding
> - Forensics → Run `file`, `strings`, `binwalk` on every file
> - General Skills → These usually involve Linux commands. Read the description carefully.

### ✅ Success Criteria
- [ ] PicoCTF account created
- [ ] 5 challenges solved (at least 3 different categories)
- [ ] Mini-writeup for each challenge
- [ ] Can explain WHY each solution works

---

## 🧪 Lab 3: Compete in a Live CTF

### Objective
Participate in a real, timed CTF competition.

> 💡 **Why This Matters**
> Live CTFs have time pressure and competition — this simulates real pentest deadlines. You'll learn to triage challenges (which to attempt first), manage your time, and work under pressure.

### Steps

```
1. Go to https://ctftime.org
2. Find an upcoming CTF (look for "beginner friendly" or "easy")
3. Register (solo or find a team on CTF Discord servers)
4. During the CTF:
   ├── Read ALL challenge descriptions first (5 min)
   ├── Sort by points (lowest = easiest)
   ├── Start with easy challenges in your strongest category
   ├── Time-box each attempt (30 min max per challenge)
   └── If stuck for 30 min → move to the next challenge
5. After the CTF:
   ├── Write detailed writeups for solved challenges
   ├── Read OTHER people's writeups for challenges you couldn't solve
   └── Note new techniques you learned
```

> 🔧 **If Stuck**
> - Can't find a team → Many CTF Discord servers have team-finding channels
> - CTF too hard → Try easier ones. Search CTFtime for "high school" or "beginners"
> - Didn't solve anything → That's normal for your first CTF. Focus on learning from writeups posted after.

### ✅ Success Criteria
- [ ] Registered for a live CTF
- [ ] Solved at least 1 challenge during the event
- [ ] Wrote a detailed writeup for solved challenges
- [ ] Read writeups for unsolved challenges

---

## 🧪 Lab 4: HackTheBox Machine

### Objective
Complete a full machine from initial scan to root flag.

> 💡 **Why This Matters**
> HackTheBox machines combine recon → exploitation → privilege escalation in one challenge. This is the closest to a real pentest you'll get in a CTF format. Every OSCP candidate practices on HTB.

### Steps

```
1. Sign up at https://hackthebox.com
2. Choose an EASY-rated retired machine
   ├── Recommended first boxes:
   │   ├── Lame (Easy, Linux)
   │   ├── Jerry (Easy, Windows)
   │   ├── Blue (Easy, Windows)
   │   └── Bashed (Easy, Linux)
3. Connect to HTB VPN
4. Follow this workflow:
   ├── nmap -sC -sV -oA scans/initial TARGET
   ├── Enumerate each open service
   ├── Find the vulnerability
   ├── Exploit for initial shell
   ├── Escalate to root
   └── Capture both user.txt and root.txt flags
5. Write a full machine writeup covering:
   ├── Enumeration
   ├── Initial foothold
   ├── Privilege escalation
   └── Lessons learned
```

> 🔧 **If Stuck**
> - Nmap shows no ports → Check VPN is connected: `ip addr show tun0`
> - Can't exploit the vulnerability → Google the service version + "exploit"
> - Got user but can't get root → Run LinPEAS (from Day 4!)
> - Completely stuck → HTB retired machines have official writeups available

### ✅ Success Criteria
- [ ] HTB account created and VPN connected
- [ ] Easy machine selected
- [ ] Got user flag (user.txt)
- [ ] Got root flag (root.txt)
- [ ] Full writeup documenting the attack chain

---

## 📋 Lab Completion Checklist

- [ ] Lab 1: TryHackMe room completed + writeup
- [ ] Lab 2: 5 PicoCTF challenges solved + mini-writeups
- [ ] Lab 3: Participated in live CTF + post-CTF learning
- [ ] Lab 4: HackTheBox machine rooted + full writeup

---

## 🧠 If You're Stuck

1. **TryHackMe VPN issues**: Use the attack box (in-browser VM) instead of VPN
2. **PicoCTF too easy**: Move to HackTheBox challenges or harder PicoCTF practice challenges
3. **Live CTF too hard**: That's normal. The value is in post-CTF writeup reading — you learn more from losses
4. **HTB machines overwhelming**: Watch IppSec's walkthroughs on YouTube (ippsec.rocks) — best resource for HTB learning
5. **Don't know where to start**: TryHackMe → PicoCTF → Live CTF → HTB. That's the progression.
