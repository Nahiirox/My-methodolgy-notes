# ACL Enumeration

`Find-InterestingDomainAcl` returns everything — too noisy to be useful. Always enumerate targeted from a user you control.

Get SID of your controlled user, then find all objects they have rights over:

```powershell
$sid = Convert-NameToSid <USER>
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

> [!warning] Without `-ResolveGUIDs` the `ObjectAceType` field returns raw GUIDs — unreadable under pressure. Always include it.

If you need to resolve a GUID manually (no PowerView):

```powershell
$guid = "<GUID>"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,rightsGuid | ? {$_.rightsGuid -eq $guid}
```

Native PS fallback — no PowerView available:

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
foreach($line in [System.IO.File]::ReadLines("ad_users.txt")) {get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match '<DOMAIN>\\<USER>'}}
```

Once you have a target object, get its SID and chain forward:

```powershell
$sid2 = Convert-NameToSid <NEXT_USER_OR_GROUP>
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid2}
```

Check if a group is nested — inherited rights flow up the chain:

```powershell
Get-DomainGroup -Identity "<GROUP>" | select memberof
```

---

## BloodHound ACL Enumeration

Set your controlled user as start node → **Node Info** → **Outbound Control Rights**:

- **First Degree Object Control** — direct ACE rights from this user
- **Transitive Object Control** — full chained path through nested groups and ACL hops

Right-click any edge → **Help** → shows abuse method, tools, commands, and opsec notes for that specific right.

Pre-built query **Find Principals with DCSync Rights** — confirms if any account in your chain has `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-In-Filtered-Set` → DCSync viable.

> [!tip] BloodHound Transitive Object Control count is the fastest way to assess how far an ACL chain goes. If it's high, you likely have a path to DA — follow the edges with the Help menu.