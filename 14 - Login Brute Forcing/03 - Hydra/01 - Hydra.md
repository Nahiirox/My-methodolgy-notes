# Hydra

Flags worth knowing: `-l`/`-L` (single/list user), `-p`/`-P` (single/list pass), `-t` (threads), `-f` (stop on first hit), `-s` (non-default port), `-V` (verbose, shows each attempt), `-M` (multi-target file).

Single cred pair, single target:

```bash
hydra -l <USER> -p <PASS> <TARGET> <service>
```

Wordlists, single target:

```bash
hydra -L <WORDLIST_U> -P <WORDLIST_P> <TARGET> <service>
```

Multi-target from file (`-M` replaces `<TARGET>`):

```bash
hydra -l <USER> -p <PASS> -M targets.txt ssh
```

Non-standard port with verbose:

```bash
hydra -L <WORDLIST_U> -P <WORDLIST_P> -s <PORT> -V <TARGET> ftp
```

HTTP basic auth:

```bash
hydra -L <WORDLIST_U> -P <WORDLIST_P> <TARGET> http-get
```

HTTP POST form — tweak path, params, and success condition per app. `S=` matches success string/code, `F=` matches failure string:

```bash
hydra -l <USER> -P <WORDLIST_P> <TARGET> http-post-form "/login:user=^USER^&pass=^PASS^:S=302"
```

> [!warning] Swap `S=` for `F=` if it's easier to identify the failure response — using the wrong one means every attempt looks like a hit or a miss.

Mask attack (generate passwords on the fly, format `min:max:charset`):

```bash
hydra -l <USER> -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 <TARGET> rdp
```

Quick service reference: `ssh` `ftp` `rdp` `vnc` `mysql` `mssql` `smtp` `pop3` `imap` `http-get` `http-post-form`