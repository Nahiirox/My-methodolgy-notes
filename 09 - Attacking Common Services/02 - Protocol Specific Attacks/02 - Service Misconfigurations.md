# Service Misconfigurations

These are the first checks on any new service — fast wins before doing anything complex.

---

## Authentication Misconfigs

Default creds are still common on older software and embedded devices. Try before anything else:

`admin:admin` / `admin:password` / `admin:<blank>` / `root:12345678` / `administrator:Password`

Workflow: grab banner → identify software version → search default creds → try weak combos if no defaults found.

Anonymous authentication — service accepts connections with no creds at all. Common on FTP, LDAP, SMB null sessions. Always test unauthenticated first.

---

## Access Rights Misconfigs

User has more access than their role requires. Common pattern: upload-only account given read on the whole share → hunt for config files, plaintext creds, PII, proprietary data.

Check what every compromised account can actually reach — don't assume low-priv means low-value access.

---

## Unnecessary Defaults

Unused ports/services left enabled, default accounts not disabled, debug/error messages leaking stack traces, old security features disabled after upgrades.

Quick wins: verbose error messages reveal stack traces → framework/version disclosure. Admin interfaces left exposed → try default creds. Unused services running → additional attack surface.

---

## What to Check on Every Service

- Anonymous/unauthenticated access possible?
- Default creds in use?
- Overly permissive account rights?
- Debug mode / verbose errors enabled?
- Admin interface exposed?
- Unnecessary features/ports running?

> [!tip] OWASP Top 10 A05 — Security Misconfiguration. These findings are almost always present in real environments and are often the actual path to initial access or escalation. Never skip the basics under time pressure.