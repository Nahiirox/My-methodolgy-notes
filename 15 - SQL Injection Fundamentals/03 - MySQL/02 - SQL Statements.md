# SQL Statements

Inside a DB shell, insert a full row or specify columns to skip defaults (id, timestamps):

```sql
INSERT INTO <TABLE> VALUES (val1, val2, val3);
INSERT INTO <TABLE>(col2, col3) VALUES ('val2', 'val3'), ('val2b', 'val3b');
```

> [!warning] Skipping a NOT NULL column errors out — check schema first with `DESCRIBE <TABLE>;`

Pull everything or specific columns:

```sql
SELECT * FROM <TABLE>;
SELECT col1, col2 FROM <TABLE>;
```

Modify schema post-access:

```sql
ALTER TABLE <TABLE> ADD <COL> INT;
ALTER TABLE <TABLE> RENAME COLUMN <COL> TO <NEWCOL>;
ALTER TABLE <TABLE> MODIFY <COL> DATE;
ALTER TABLE <TABLE> DROP <COL>;
```

Update records — always scope with WHERE or you hit every row:

```sql
UPDATE <TABLE> SET <COL>='<VAL>' WHERE id > 1;
```

> [!warning] `DROP TABLE <TABLE>` is instant, permanent, no prompt — don't run it unless you mean it