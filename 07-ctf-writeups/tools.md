# 07 — CTF Writeups: Tools

> Essential tools for CTF competitions across all categories.

> 📋 **What You Will Do In This Section**
> - [ ] Install and verify tools for Web, Binary, Crypto, and Forensics
> - [ ] Know which tool to use for each challenge type
> - [ ] Set up your CTF environment before competition day

---

## 🔴 Web

> 💡 **Why This Matters**
> Web challenges are the most common CTF category and the most relevant to real-world pentesting. These tools let you intercept requests, fuzz parameters, and exploit vulnerabilities.

```bash
Burp Suite       # Request interception and manipulation (ESSENTIAL)
curl / httpie    # Command-line HTTP requests
SQLMap           # Automated SQL injection
ffuf / gobuster  # Directory/parameter fuzzing
jwt_tool         # JWT analysis and attacks
```

#### 🧪 Try It Now — Web Tool Check

```bash
for tool in curl sqlmap ffuf gobuster; do
  which $tool &>/dev/null && echo "✅ $tool installed" || echo "❌ $tool missing"
done
```

---

## 🔴 Binary / Pwn

> 💡 **Why This Matters**
> Binary challenges teach low-level exploitation — buffer overflows, format strings, ROP chains. These skills are tested in OSCP and are essential for understanding how software is exploited at the memory level.

```bash
GDB + GEF        # Debugger with exploit-dev extensions
                  # Install GEF: bash -c "$(curl -fsSL https://gef.blah.cat/sh)"
pwntools         # Python exploit development library
                  # Install: pip3 install pwntools
checksec         # Check binary protections (ASLR, NX, PIE, canary)
                  # Usage: checksec --file=binary
ROPgadget        # Find ROP gadgets in binaries
one_gadget       # Find single-gadget RCE in libc
Ghidra           # Free reverse engineering tool (NSA)
                  # Download: ghidra-sre.org
radare2          # CLI reverse engineering framework
```

#### 🧪 Try It Now — Binary Tool Check

```bash
for tool in gdb python3 checksec objdump strings file; do
  which $tool &>/dev/null && echo "✅ $tool" || echo "❌ $tool"
done
python3 -c "from pwn import *; print('✅ pwntools')" 2>/dev/null || echo "❌ pwntools (pip3 install pwntools)"
```

---

## 🔴 Crypto

> 💡 **Why This Matters**
> Crypto challenges teach you to break weak encryption — a skill that's directly applicable to finding JWT flaws, weak password hashing, and custom crypto implementations in real applications.

```bash
CyberChef        # Browser-based encoding/decoding/analysis
                  # URL: gchq.github.io/CyberChef (BOOKMARK THIS)
RsaCtfTool       # Automated RSA attacks
                  # Install: git clone https://github.com/RsaCtfTool/RsaCtfTool
hashcat / john   # Hash cracking
SageMath         # Mathematical computations for advanced crypto
pycryptodome     # Python crypto library
                  # Install: pip3 install pycryptodome
```

---

## 🔴 Forensics

> 💡 **Why This Matters**
> Forensics tools help you analyze files, network captures, and memory dumps. In real-world incident response, these same tools are used to investigate breaches.

```bash
Wireshark        # Network capture analysis (GUI)
tshark           # Wireshark CLI
volatility3      # Memory forensics
binwalk          # Firmware/file extraction and embedded file detection
foremost         # File carving from binary data
exiftool         # Metadata extraction (EXIF, XMP, etc.)
steghide         # Steganography extraction (JPEG)
zsteg            # Steganography extraction (PNG)
autopsy          # Disk forensics
```

#### 🧪 Try It Now — Forensics Tool Check

```bash
for tool in wireshark tshark binwalk foremost exiftool steghide strings xxd file; do
  which $tool &>/dev/null && echo "✅ $tool" || echo "❌ $tool"
done
```

---

## 🔴 Universal (Always Needed)

```bash
Python3          # Scripting everything — your #1 CTF weapon
CyberChef        # Quick encoding/decoding/analysis (BOOKMARK IT)
strings          # Extract readable strings from any file
xxd              # Hex dump and reverse
file             # Identify file types (first command on any unknown file)
base64           # Encode/decode base64
```

---

## 📋 Tool Installation One-Liner

```bash
# Install most CTF essentials on Ubuntu/Kali
sudo apt update && sudo apt install -y \
  python3 python3-pip gdb binwalk foremost exiftool steghide \
  wireshark tshark nmap gobuster sqlmap john hashcat \
  curl wget git jq xxd

# Python libraries
pip3 install pwntools requests pycryptodome

# GEF for GDB
bash -c "$(curl -fsSL https://gef.blah.cat/sh)"

# ffuf
go install github.com/ffuf/ffuf/v2@latest
```

---

## 🧠 If You're Stuck

1. **Don't know which tool to use**: Start with `file` and `strings` for ANY unknown file. For web, start with `curl` and view source.
2. **Tool not installing**: Try using Kali Linux — it comes with most tools pre-installed.
3. **CyberChef confusing**: Use the "Magic" recipe — it auto-detects encoding.
4. **GDB too complex**: Install GEF — it adds colors, better output, and exploit-dev helpers.
