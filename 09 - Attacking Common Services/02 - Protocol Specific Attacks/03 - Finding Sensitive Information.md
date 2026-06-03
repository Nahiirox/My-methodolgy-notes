# Finding Sensitive Information

Chain information across services — a username from FTP becomes a login for email, which yields DB creds, which yields RCE. Nothing is insignificant. Document everything immediately.

---

## The Recon Chain Mindset

Every service you enumerate feeds the next one. Usernames, email addresses, file names, error messages — all are potential pivot points. The example chain: FTP anon access → filename `johnsmith` → email login as `johnsmith` → email search for "password" → MSSQL creds → `xp_cmdshell` → RCE.

---

## What to Hunt For

On any service: usernames, email addresses, passwords (especially in emails and config files), DNS records, IP addresses, source code, config files, PII.

---

## Service-Specific Targets

File shares → config files (`.xml`, `.ini`, `.config`, `.env`), scripts with hardcoded creds, password files, private keys

Email → search keywords: `password`, `credentials`, `login`, `secret`, `key`, `access` — look for onboarding emails, IT helpdesk threads, password reset emails

Databases → look for stored creds in tables (`users`, `accounts`, `config`, `settings`), linked servers, stored procedures

---

## Two Prerequisites Before Any Hunt

Know how the service works technically (auth mechanisms, access model, query syntax).

Know what you're looking for before you start — don't scroll aimlessly, search specifically.

> [!tip] First search in any email access: `password`. Second: `credentials`. Third: sender/recipient patterns with IT/helpdesk accounts. One email thread from a password reset will often give you lateral movement directly.