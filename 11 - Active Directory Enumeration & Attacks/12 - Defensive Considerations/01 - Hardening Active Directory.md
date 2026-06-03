# AD Hardening — Key Mitigations by Attack Surface

Reference for reporting recommendations and understanding what you're up against.

---

## Mitigations That Block Specific Attacks

`ms-DS-MachineAccountQuota = 0` → blocks NoPac and RBCD attacks that require adding computer accounts

`SMB signing enabled + required` → blocks LLMNR/NBT-NS relay attacks

`LDAP signing enabled` → blocks LDAP relay

Disable print spooler → blocks PrintNightmare and Printer Bug (MS-RPRN coercion)

Disable NTLM for DCs + enable Extended Protection for Authentication + require SSL on AD CS → blocks PetitPotam and NTLM relay to AD CS

`RestrictNullSessAccess = 1` → blocks null session enumeration

Disable unconstrained delegation → blocks TGT theft via coerced auth

gMSA/MSA for service accounts → makes Kerberoasting impossible (passwords are 240-char random, auto-rotated)

`Protected Users` group on privileged accounts → blocks NTLM auth, RC4/DES, unconstrained delegation, plaintext cred caching, CredSSP caching. No TGT renewable beyond 4 hours.

> [!warning] Protected Users can cause auth failures. Staged rollout only — never add all DA accounts at once.

Enforce AES-only Kerberos encryption → makes Kerberoasting significantly harder (mode 19700 vs 13100)

---

## Detection Signatures (for reporting context)

LLMNR/NBT-NS poisoning → ports UDP 5355/137 traffic, event IDs 4697/7045

Password spraying → event IDs 4624/4648 (multiple failed logons across accounts in short window)

Kerberoasting → event ID 4769 (TGS requests), RC4 encryption type `0x17` is the red flag

ASREPRoasting → event ID 4768 with pre-auth not required

DCSync → event IDs 4662 (replication rights exercised), unusual replication from non-DC hosts

ACL changes → event ID 5136 (directory service object modified)

---

## High-Impact Quick Wins Worth Recommending to Clients

Clean up DA/EA membership — most orgs have 10x more than needed. Audit `admincount=1` accounts.

Remove passwords from user Description fields and SYSVOL scripts.

Scrub document metadata before publishing externally — leaks username format.

Disable default RID-500 Administrator, deploy LAPS for local admin rotation.

Periodic BloodHound/PingCastle/Grouper runs to catch ACL drift and new misconfigs.

Quarterly or at minimum annual AD security assessments.