# Living Off the Land — Windows Native Enum

No tools available → built-in only. Fewer logs than importing external tools, blends into normal admin activity.

## Host Recon

Quick single-command host summary — generates one log entry instead of many:

```cmd
systeminfo
```

Check who else is logged on before doing anything noisy:

```powershell
qwinsta
```

PS history — may contain plaintext creds or paths to scripts with creds:

```powershell
Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt
```

Firewall state across all profiles:

```cmd
netsh advfirewall show allprofiles
```

Defender state:

```cmd
sc query windefend
```

---

## OPSEC — PS Downgrade

PS v2 has no script block logging — drops below the 3.0 threshold where event logging kicks in:

```powershell
powershell.exe -version 2
```

> [!warning] The downgrade command itself is logged before the new session starts. Defenders see the v2 launch in the Windows PowerShell log even if nothing after is captured. It will look suspicious.

---

## Network Recon

ARP table — hosts the box has recently talked to, faster than scanning:

```cmd
arp -a
```

Routing table — reveals network segments reachable from this host, pivot candidates:

```cmd
route print
```

---

## WMI

Domain info — domain name, DC address, child domains, forest trusts in one shot:

```cmd
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
```

Patch level:

```cmd
wmic qfe get Caption,Description,HotFixID,InstalledOn
```

Running processes, local users, service accounts:

```cmd
wmic process list /format:list
wmic useraccount list /format:list
wmic sysaccount list /format:list
```

---

## Net Commands

> [!warning] `net.exe` commands are commonly monitored by EDR. Substitute `net1` for `net` to avoid the exact string match trigger.

Password policy:

```cmd
net accounts /domain
```

Domain groups, DA membership, DCs:

```cmd
net group /domain
net group "Domain Admins" /domain
net group "Domain Controllers" /domain
```

Specific user detail:

```cmd
net user <USER> /domain
```

Shares, visible computers:

```cmd
net share
net view /all /domain
```

---

## Dsquery

Exists on any host with AD DS role installed. `dsquery.dll` is on all modern Windows by default — always available with SYSTEM or elevated prompt.

All users / computers:

```powershell
dsquery user
dsquery computer
```

Wildcard — all objects in an OU:

```powershell
dsquery * "CN=Users,DC=<DOMAIN>,DC=<TLD>"
```

Users with `PASSWD_NOTREQD` set (UAC bit 32) — no password required, easy spray targets:

```powershell
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
```

Find DCs (UAC bit 8192):

```powershell
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName
```

LDAP OID matching rules for dsquery filters: `1.2.840.113556.1.4.803` = exact bit match / `1.2.840.113556.1.4.804` = any bit match / `1.2.840.113556.1.4.1941` = match through DN chain (membership/ownership). Combine with `(&(filter1)(filter2))`, negate with `(!filter)`.