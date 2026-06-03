# SQL Injection — Intro

SQLi = user input escapes its intended context and modifies the actual SQL query the app sends to the DB.

Entry point: inject `'` or `"` to break out of the input boundary → confirm the query breaks (error or unexpected behavior) → build from there.

Two steps to exploitation: (1) inject outside the expected input limits, (2) make the resulting query valid and useful — via stacked queries or UNION.

Impact tiers to know for scope decisions: data exfil (creds, PII) → auth bypass (login without creds, access locked features) → file read/write on the server → webshell → full backend compromise.

> [!tip] File read/write via SQLi is only possible if the DB user has FILE privilege and secure_file_priv is permissive. Check this early — it's the direct path from SQLi to RCE.