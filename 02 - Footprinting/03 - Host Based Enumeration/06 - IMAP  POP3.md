# IMAP / POP3 — Footprinting

Ports: IMAP 143 (plain) / 993 (TLS). POP3 110 (plain) / 995 (TLS). SSL cert on both leaks hostname, org, email address — read it.

Nmap — capabilities output tells you auth methods and available commands:

```bash
sudo nmap <TARGET> -sV -p110,143,993,995 -sC
```

---

### With Creds — curl (fastest)

List mailbox folders over IMAPS:

```bash
curl -k 'imaps://<TARGET>' --user <USER>:<PASS>
```

Verbose — shows TLS version, full cert chain, server banner + version:

```bash
curl -k 'imaps://<TARGET>' --user <USER>:<PASS> -v
```

---

### Manual Interaction — openssl (TLS)

POP3S:

```bash
openssl s_client -connect <TARGET>:995
```

IMAPS:

```bash
openssl s_client -connect <TARGET>:imaps
```

After connecting — IMAP commands:

```
1 LOGIN <USER> <PASS>
1 LIST "" *
1 SELECT INBOX
1 FETCH 1 all
1 LOGOUT
```

POP3 commands:

```
USER <USER>
PASS <PASS>
LIST
RETR <ID>
QUIT
```

---

### What to Hunt Once Logged In

Search for keywords across all folders: `password`, `credentials`, `login`, `secret`, `key`. IMAP lets you stay in the mailbox and browse — POP3 is fetch-only, no folder structure.

> [!tip] SSL cert `emailAddress` field often contains the admin's email and confirms internal username format. Grab it from the Nmap output or openssl handshake before doing anything else.

> [!warning] `auth_debug_passwords` or `auth_verbose_passwords` in Dovecot config = cleartext passwords written to logs. If you have file read on the mail server, check `/var/log/dovecot*` immediately.