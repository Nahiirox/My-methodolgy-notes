# Domain Trusts — Primer & Enumeration

Trusts are frequently misconfigured and overlooked. A soft target in a trusted domain can be the path into the primary domain when the primary domain itself is locked down.

---

## Trust Types to Know

`Parent-child` — two-way transitive, within the same forest. Child domains fully trust parent and vice versa.

`Forest` — transitive trust between two forest root domains. `ForestTransitive: True` in output.

`External` — non-transitive, between domains in separate forests not joined by a forest trust. SID filtering applies.

`Cross-link` — between child domains within the same forest, speeds up auth.

Transitive = trust extends further down the chain. Non-transitive = only the directly trusted domain.

Bidirectional = users from either domain can auth into the other. One-way = only one direction.

> [!tip] Bidirectional forest trust = users in domain B can potentially Kerberoast, spray, or abuse ACLs in domain A and vice versa. Always enumerate what's accessible across each trust.

---

## Enumerating Trusts

Built-in — no tools needed:

```powershell
Import-Module activedirectory
Get-ADTrust -Filter *
```

Key fields: `Direction` / `ForestTransitive` (forest trust = True) / `IntraForest` (child domain = True) / `SIDFilteringQuarantined` (SID filtering on = harder to abuse)

PowerView — cleaner output, maps the full chain:

```powershell
Get-DomainTrust
Get-DomainTrustMapping
```

Enum users in a trusted/child domain:

```powershell
Get-DomainUser -Domain <CHILD_OR_TRUSTED_DOMAIN> | select SamAccountName
```

netdom — built-in, no import needed:

```cmd
netdom query /domain:<DOMAIN> trust
netdom query /domain:<DOMAIN> dc
netdom query /domain:<DOMAIN> workstation
```

BloodHound — pre-built query: **Map Domain Trusts** → visualizes all trust relationships and directions at once. First thing to run after collecting data in multi-domain environments.

---

## Reading the Output

`WITHIN_FOREST` + `IntraForest: True` → child domain of current forest root

`FOREST_TRANSITIVE: True` → cross-forest trust → different forest entirely

`Bidirectional` → auth flows both ways → enumerate and attack in both directions

`SIDFilteringQuarantined: True` → SID history attacks blocked across this trust

Confirm trust is in scope before attacking across it. Note all trusts discovered in reporting — large orgs often don't know all their trusts exist.