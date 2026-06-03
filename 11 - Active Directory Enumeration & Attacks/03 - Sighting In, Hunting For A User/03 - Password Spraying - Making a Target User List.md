# Building a User List for Spraying

No creds → null session / LDAP anon bind / Kerbrute. Got creds → CME directly.

---

## No Creds — SMB Null Session

enum4linux — clean user list one-liner:

```bash
enum4linux -U <DC_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

rpcclient — interactive, same result:

```bash
rpcclient -U "" -N <DC_IP>
# then:
enumdomusers
```

CME without creds — also shows `badpwdcount` per user:

```bash
crackmapexec smb <DC_IP> --users
```

> [!warning] Check `badpwdcount` before spraying. Any account already at threshold-1 → pull it from the list or you lock it on the first spray.

## No Creds — LDAP Anonymous Bind

ldapsearch — filter for user objects, extract sAMAccountName:

```bash
ldapsearch -h <DC_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
```

windapsearch — simpler syntax:

```bash
./windapsearch.py --dc-ip <DC_IP> -u "" -U
```

## No Creds — Kerbrute

Userenum via Kerberos pre-auth — no 4625 events, no lockouts from enum alone:

```bash
kerbrute userenum -d <DOMAIN> --dc <DC_IP> <WORDLIST>
```

> [!warning] Kerbrute userenum generates event ID 4768 (TGT requested) — visible if Kerberos logging is on. Switching to spray mode with Kerbrute **does** count toward lockout. Enum is safe; spray is not.

## With Creds

CME — also surfaces `badpwdcount` per user:

```bash
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users
```

No null session + no LDAP bind + no creds → fall back to linkedin2username or external email harvesting → feed into Kerbrute to validate before spraying.