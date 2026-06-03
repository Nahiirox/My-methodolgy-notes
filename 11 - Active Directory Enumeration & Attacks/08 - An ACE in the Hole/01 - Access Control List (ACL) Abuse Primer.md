# ACL Abuse — Primer

ACL misconfigs are invisible to vuln scanners, persist for years, and are often the only path forward in hardened environments. All abusable with PowerView; all visualizable in BloodHound.

---

## ACE Types Worth Knowing

`ForceChangePassword` → reset target's password without knowing current → `Set-DomainUserPassword`

`GenericWrite` → write any non-protected attribute → over user: assign SPN → Kerberoast / over group: add member / over computer: RBCD attack

`GenericAll` → full control → user: force password change or targeted Kerberoast / group: add member / computer + LAPS: read LAPS password → local admin

`AddSelf` → add yourself to a group → `Add-DomainGroupMember`

`WriteOwner` → change object owner to attacker-controlled principal → `Set-DomainObjectOwner`

`WriteDACL` → write new ACEs onto an object → `Add-DomainObjectACL` → grant yourself GenericAll or DCSync rights

`AllExtendedRights` → includes both ForceChangePassword and Add Members → `Set-DomainUserPassword` or `Add-DomainGroupMember`

> [!warning] ForceChangePassword and any write operation that modifies AD objects = destructive. Get written client approval before executing. Document before/after and revert.

---

## Finding ACL Attack Paths

BloodHound — visual attack chains, edges include all the above rights. First query: **Find Shortest Paths to Domain Admins** often surfaces ACL chains. Check **Outbound Object Control** on any account you control.

PowerView — targeted enumeration:

```powershell
Find-InterestingDomainAcl -ResolveGUIDs
```

Filter to your controlled user's outbound rights:

```powershell
$sid = Convert-NameToSid <USER>
Get-DomainObjectAcl -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

> [!tip] Exchange installs, legacy configs, and "quick fix" permission grants are the most common sources of ACL misconfigs. If you see a help desk account or service account with GenericAll over a privileged group, that's the path.