# Internal Password Spraying — Windows

On a domain-joined host, DomainPasswordSpray auto-generates the user list from AD, pulls the password policy, and removes accounts within one attempt of lockout — no manual prep needed:

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password <PASS> -OutFile <OUTFILE> -ErrorAction SilentlyContinue
```

Not domain-authenticated → add `-UserList <USERLIST>` manually.

Hits land in `<OUTFILE>`. Kerbrute on Windows works identically to Linux if you prefer it (`C:\Tools`).

---

## Detection Signatures to Know

Event ID 4625 (failed logon) → SMB spray. Event ID 4771 (Kerberos pre-auth failed) → Kerbrute/LDAP spray. Kerbrute generates less 4625 noise but 4771 is visible if Kerberos logging is on.

> [!tip] External spray targets if you ever pivot that way: O365, OWA, Exchange Web Access, Citrix, RDS portals, VPNs (SonicWall/Fortinet/OpenVPN), VMware Horizon — anything running AD auth.

Got valid creds → move to credentialed enum. BloodHound, PowerView, CME are next.