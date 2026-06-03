# Intro to MySQL

Connect — no space between -p and password if passing inline (avoid, stays in history):

```bash
mysql -u <USER> -p
mysql -u <USER> -p<PASS>
mysql -u <USER> -h <TARGET> -P <PORT> -p
```

DB basics once inside:

```sql
SHOW DATABASES;
CREATE DATABASE <DB>;
USE <DB>;
SHOW TABLES;
DESCRIBE <TABLE>;
```

Full CREATE TABLE with all the constraints you'll actually need:

```sql
CREATE TABLE logins (
    id INT NOT NULL AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(100) NOT NULL,
    date_of_joining DATETIME DEFAULT NOW(),
    PRIMARY KEY (id)
);
```

Constraint cheatsheet — inline, not a list because it flows: `NOT NULL` = required, `AUTO_INCREMENT` = auto-id, `UNIQUE` = no dupes, `DEFAULT NOW()` = timestamp on insert, `PRIMARY KEY` = unique row identifier.