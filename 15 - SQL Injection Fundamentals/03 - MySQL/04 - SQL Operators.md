# SQL Operators

AND, OR, NOT — symbol equivalents: `&&`, `||`, `!` / `!=`

Combine conditions in WHERE:

```sql
SELECT * FROM logins WHERE username != 'john' AND id > 1;
```

Operator precedence (high → low): arithmetic `* / %` → `+ -` → comparisons `= > < != LIKE` → `NOT` → `AND` → `OR`

OR evaluates last — critical for SQLi auth bypass payloads like `' OR 1=1 --` working even when the first condition is false.

> [!tip] Precedence matters for injections. `' OR 1=1 AND id > 0 --` — AND evaluates before OR, so AND resolves first then OR. To force OR to win, use parentheses: `' OR (1=1) --`. Know the order before crafting blind injection conditions.