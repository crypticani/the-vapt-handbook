# 07 — CTF Writeups: Tools

> Essential tools for CTF competitions across all categories.

## 🔴 Web
```bash
Burp Suite       # Request interception and manipulation
curl / httpie    # Command-line HTTP requests
SQLMap           # Automated SQL injection
ffuf / gobuster  # Directory and parameter fuzzing
jwt_tool         # JWT analysis and attacks
```

## 🔴 Binary / Pwn
```bash
GDB + GEF        # Debugger with exploit-dev extensions (https://github.com/hugsy/gef)
pwntools         # Python exploit development library (pip3 install pwntools)
checksec         # Check binary protections (ASLR, NX, PIE, canary)
ROPgadget        # Find ROP gadgets in binaries
one_gadget       # Find single-gadget RCE in libc
Ghidra           # Free reverse engineering tool (NSA)
radare2          # CLI reverse engineering
```

## 🔴 Crypto
```bash
CyberChef        # Browser-based encoding/decoding (gchq.github.io/CyberChef)
RsaCtfTool       # RSA attack tool (https://github.com/RsaCtfTool/RsaCtfTool)
hashcat / john   # Hash cracking
SageMath         # Mathematical computations for crypto
pycryptodome     # Python crypto library
```

## 🔴 Forensics
```bash
Wireshark        # Network capture analysis
volatility3      # Memory forensics
binwalk          # Firmware/file extraction
foremost         # File carving
exiftool         # Metadata extraction
steghide         # Steganography (JPEG)
zsteg            # Steganography (PNG)
autopsy          # Disk forensics
```

## 🔴 Universal
```bash
Python3          # Scripting everything
CyberChef        # Quick encoding/decoding/analysis
strings          # Extract readable strings from any file
xxd              # Hex dump
file             # Identify file types
```
