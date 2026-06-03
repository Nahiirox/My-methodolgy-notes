# Enumerating Password Policy

Get it before spraying. Determines safe spray count and delay window.

---

## From Linux — With Creds

```bash
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --pass-pol
```

## From Linux — No Creds (SMB Null Session)

Check if null session is open first:

```bash
rpcclient -U "" -N <DC_IP>
```

Then inside rpcclient:

```
querydominfo
getdompwinfo
```

enum4linux-ng — cleaner output, saves JSON/YAML for later use:

```bash
enum4linux-ng -P <DC_IP> -oA <OUTFILE>
```

## From Linux — No Creds (LDAP Anonymous Bind)

```bash
ldapsearch -h <DC_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

> [!tip] Newer ldapsearch: `-H ldap://<DC_IP>` replaces `-h`.

## From Windows

No tools available — built-in:

```cmd
net accounts
```

With PowerView:

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

Null session check from Windows — error codes matter:

```cmd
net use \\<DC>\ipc$ "" /u:""
```

`1331` = account disabled / `1326` = wrong creds / `1909` = account locked out

---

## Reading the Policy for Spray Math

Key fields: `lockoutThreshold` + `lockoutDuration` + `minPwdLength`.

Threshold 5, duration 30 min → safe to spray 2-3 attempts, then wait 31+ min. Never hit the threshold. If duration says "admin must unlock" — one spray only, maybe none.

`PasswordComplexity=1` → weak passwords still satisfy it: `Welcome1`, `Password1`, `Winter2024` all count.

No policy available → one spray, wait 1+ hour, maybe don't bother. Ask the client if the engagement allows it.