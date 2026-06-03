# SQL Query Results — Control & Filtering

Sort ascending (default) or descending, multi-column sort for tiebreaking:

```sql
SELECT * FROM logins ORDER BY password DESC, id ASC;
```

Limit rows returned — `LIMIT <count>` or `LIMIT <offset>, <count>` (offset is 0-indexed):

```sql
SELECT * FROM logins LIMIT 2;
SELECT * FROM logins LIMIT 1, 2;
```

Filter with WHERE — strings need quotes, numbers don't:

```sql
SELECT * FROM logins WHERE id > 1;
SELECT * FROM logins WHERE username = 'admin';
```

Pattern match with LIKE — `%` = zero or more chars wildcard, `_` = exactly one char:

```sql
SELECT * FROM logins WHERE username LIKE 'admin%';
SELECT * FROM logins WHERE username LIKE '___';
```

> [!tip] These matter for SQLi — `LIMIT 0,1` is the standard single-row extraction pattern in blind injection. `LIKE` with `%` is useful when you don't know exact values. `ORDER BY <n>` is used to enumerate column count before UNION attacks.