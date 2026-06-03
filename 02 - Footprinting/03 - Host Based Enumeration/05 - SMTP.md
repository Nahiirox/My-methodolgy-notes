# SMTP — Footprinting

Ports: TCP/25 (default), TCP/587 (STARTTLS/authenticated submission), TCP/465 (SSL). Banner on connect leaks server software and hostname.

Nmap — `smtp-commands` script fires with `-sC`, lists all supported EHLO extensions:

```bash
sudo nmap <TARGET> -sC -sV -p25
```

Check for open relay — 16 tests, any pass = spoofed email is viable:

```bash
sudo nmap <TARGET> -p25 --script smtp-open-relay -v
```

---

### Manual Interaction

Banner grab + EHLO to see supported extensions:

```bash
telnet <TARGET> 25
EHLO <DOMAIN>
```

`VRFY` to enumerate users — `252` response = server confirms existence (or lies and returns 252 for everything — check a known-invalid user to calibrate):

```bash
VRFY <USERNAME>
```

Send a spoofed email if open relay confirmed:

```bash
MAIL FROM: <spoofed@domain.com>
RCPT TO: <target@domain.com> NOTIFY=success,failure
DATA
From/To/Subject headers + body
.
QUIT
```

> [!warning] VRFY returning `252` for every input including garbage usernames means the server isn't disclosing user existence — don't trust the output. Test with a random string first to establish the baseline response.

> [!tip] `mynetworks = 0.0.0.0/0` in Postfix config = open relay. Open relay → spoof internal sender addresses → phishing with trusted domain = high-value social engineering path. Also note it for the report as critical misconfiguration.

`EHLO` output tells you what attack surface exists: `VRFY` listed = user enum possible / `AUTH PLAIN` listed = creds sent in base64 over cleartext unless STARTTLS active / `ETRN` listed = can trigger outbound queue flush.