# Credentialed Enumeration — Windows

## ActiveDirectory PowerShell Module

Load if not imported:

```powershell
Import-Module ActiveDirectory
```

Domain info — child domains, SID, DCs, FSMO roles:

```powershell
Get-ADDomain
```

Users with SPN set — Kerberoasting candidates:

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

All domain trusts — direction, intra/cross-forest:

```powershell
Get-ADTrust -Filter *
```

Group membership:

```powershell
Get-ADGroup -Filter * | select name
Get-ADGroupMember -Identity "<GROUP>"
```

---

## PowerView

Import:

```powershell
Import-Module .\PowerView.ps1
```

Detailed user info — pull `admincount`, `useraccountcontrol`, `memberof`, SPN:

```powershell
Get-DomainUser -Identity <USER> -Domain <DOMAIN> | Select-Object -Property name,samaccountname,description,memberof,pwdlastset,admincount,useraccountcontrol,serviceprincipalname
```

Domain Admins with nested group membership resolved:

```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

All trust mappings including child domains:

```powershell
Get-DomainTrustMapping
```

Test local admin access on a host:

```powershell
Test-AdminAccess -ComputerName <HOSTNAME>
```

All users with SPN — Kerberoasting list:

```powershell
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

> [!tip] `admincount=1` on a user = was protected by AdminSDHolder at some point. Worth targeting even if not currently in a privileged group.

---

## SharpView

.NET port of PowerView — use when PS is blocked or constrained. Same function names:

```powershell
.\SharpView.exe Get-DomainUser -Identity <USER>
.\SharpView.exe Get-DomainUser -Help
```

---

## Snaffler

Crawls all domain shares for high-value files — run from domain-joined host, output to file and review after:

```powershell
.\Snaffler.exe -s -d <DOMAIN> -o <OUTFILE>.log -v data
```

Color coding: `{Red}` = high value (`.key`, `.sqldump`, `.mdf`, `.keychain`), `{Black}` = interesting (`.kdb`, `.ppk`, `.psafe3`), `{Green}` = accessible share.

> [!tip] Snaffler output makes great client deliverable — shows exactly which shares expose sensitive file types. Save the log.

---

## SharpHound (Windows)

Collect everything, zip output:

```powershell
.\SharpHound.exe -c All --zipfilename <OUTFILE>
```

Transfer zip to attack host → upload to BloodHound GUI.

Key BloodHound queries to run first:

- **Find Shortest Paths To Domain Admins** — attack path chains
- **Find Computers with Unsupported Operating Systems** — legacy = quick SYSTEM
- **Find Computers where Domain Users are Local Admin** — any cred = local admin on those hosts

> [!warning] Legacy hosts flagged by BloodHound may be fragile/critical systems. Confirm with client before exploiting — check if they're even live first.