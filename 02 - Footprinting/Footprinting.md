## Enumeration Principles

Enumeration is information gathering using active (scans) and passive (third-party) methods. OSINT is passive-only and run separately from enumeration.

1. There is more than meets the eye. Consider all points of view.
2. Distinguish between what we see and what we do not see.
3. There are always ways to gain more information. Understand the target.

## Enumeration Methodology

Methodology is a systematic approach, not a step-by-step guide. Enumeration splits into infrastructure-based, host-based, and OS-based levels, nested in 6 layers, each a barrier to pass through toward the target.

1. Internet Presence: domains, subdomains, vHosts, ASN, netblocks, IPs, cloud instances, security measures. Identify all target systems and interfaces that can be tested.
2. Gateway: firewalls, DMZ, IPS/IDS, EDR, proxies, NAC, network segmentation, VPN, Cloudflare. Understand how the target is protected and where it sits in the network.
3. Accessible Services: service type, functionality, configuration, port, version, interface. Understand each service's purpose well enough to communicate with and exploit it.
4. Processes: PID, processed data, tasks, source, destination. Identify dependencies between processes.
5. Privileges: groups, users, permissions, restrictions, environment. Identify internal permissions and what they allow, often overlooked by admins.
6. OS Setup: OS type, patch level, network config, OS environment, config files, sensitive private files. Reflects internal security and admin skill level.

## Domain Information

Passive gathering of a company's full Internet presence, done without active scans to stay hidden.

### Online Presence

#### Certificate Transparency

crt.sh logs SSL certs issued by CAs (RFC 6962). Used to find subdomains from issued certificates.  
https://www.crt.sh/?q=inlanefreight.com

```bash
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq .
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

#### Company Hosted Servers

```bash
for i in $(cat subdomainlist);do host $i | grep "has address" | grep <TARGET_DOMAIN> | cut -d" " -f1,4;done
```

Identifies which hosts are directly company-hosted vs third-party, since third-party hosts cannot be tested without their permission.

#### Shodan

Searches the Internet for open TCP/IP ports on permanently connected devices (IoT, servers, cameras, controllers).

```bash
for i in $(cat subdomainlist);do host $i | grep "has address" | grep <TARGET_DOMAIN> | cut -d" " -f4 >> ip-addresses.txt;done
for i in $(cat ip-addresses.txt);do shodan host $i;done
```

#### DNS Records

```bash
dig any <TARGET_DOMAIN>
```

`A` IPv4 of domain | `MX` responsible mail servers | `NS` nameservers, often reveals hosting provider | `TXT` verification keys, SPF/DMARC/DKIM, third-party providers

TXT, MX, and NS entries can reveal third-party providers (Atlassian, Google Workspace, LogMeIn, Mailgun, Office 365/Azure), each hinting at further attack surface: open GDrive links, Azure blob/SMB storage, API endpoints for IDOR/SSRF, or a hosting provider's own management portal (the TXT "MS=" style value is often the account ID/username for that portal).

## Cloud Resources

Misconfigured cloud storage (AWS S3, Azure blobs, GCP cloud storage) can be accessed without authentication.

Google dorks: `intext:<COMPANY> inurl:amazonaws.com` | `intext:<COMPANY> inurl:blob.core.windows.net`

Cloud storage links also appear in page source (DNS prefetch/preconnect to blob.core.windows.net) or via GrayHatWarfare, which searches and filters exposed S3/Azure/GCP storage by file format. domain.glass shows infra details and a Cloudflare security assessment status (a gateway-layer security measure).

Exposed storage can leak sensitive files such as SSH private/public keys, giving passwordless login to company systems.

## Staff

Employee profiles (LinkedIn, Xing) and job postings reveal the company's tech stack (languages, frameworks, databases, tools) and further OSINT targets (personal emails, hardcoded secrets in shared code/repos).

## FTP

File Transfer Protocol. Control channel on TCP 21, data channel on TCP 20. Active mode: server connects back to the client (blocked by client-side firewalls). Passive mode: server opens a port for the client to connect to (firewall-friendly). Often allows anonymous login; clear-text, can be sniffed.

### TFTP

Simpler than FTP, uses UDP, no authentication. Access is limited to files/dirs with global read/write permissions, no directory listing. Only safe in local/trusted networks.

`connect` set remote host/port | `get` download | `put` upload | `quit` exit | `status` show transfer mode, connection, timeout | `verbose` toggle extra transfer info

### Default Configuration

Config at /etc/vsftpd.conf. /etc/ftpusers denies listed users FTP access even if they exist on the system.

### Dangerous Settings

`anonymous_enable=YES` allow anonymous login | `anon_upload_enable=YES` allow anonymous upload | `anon_mkdir_write_enable=YES` allow anonymous to create dirs | `no_anon_password=YES` skip password prompt for anonymous | `anon_root=<PATH>` anonymous root dir | `write_enable=YES` allow STOR/DELE/RNFR/RNTO/MKD/RMD/APPE/SITE

`chown_uploads=YES` change ownership of anon uploads | `chown_username=<USERNAME>` owner given to anon uploads | `chroot_local_user=YES` jail local users to home dir | `chroot_list_enable=YES` use list of users to chroot

`hide_ids=YES` shows all UID/GID in listings as `ftp`, hides real ownership | `ls_recurse_enable=YES` allows recursive listing of all content at once

### Anonymous FTP Interaction

```bash
ftp <TARGET_IP>
```

```text
ftp> ls
ftp> status
ftp> debug
ftp> trace
ftp> ls -R
ftp> get <FILE>
ftp> put <FILE>
```

Login as anonymous:anonymous when allowed. `debug` and `trace` show packet-level detail of commands sent/received. `ls -R` needs `ls_recurse_enable=YES` on the server.

```bash
wget -m --no-passive ftp://anonymous:anonymous@<TARGET_IP>
```

Downloads the full accessible tree into a folder named after the target IP. Uploading can lead to RCE if the share backs a web server.

### Footprinting the Service

```bash
sudo nmap --script-updatedb
find / -type f -name ftp* 2>/dev/null | grep scripts
sudo nmap -sV -p21 -sC -A <TARGET_IP>
sudo nmap -sV -p21 -sC -A <TARGET_IP> --script-trace
nc -nv <TARGET_IP> 21
telnet <TARGET_IP> 21
openssl s_client -connect <TARGET_IP>:21 -starttls ftp
```

`ftp-anon` checks anonymous login, lists root dir contents if allowed | `ftp-syst` runs STAT, shows server status and version | `--script-trace` traces NSE network-level activity | `openssl ... -starttls ftp` connects over FTPS and shows the server cert

## SMB

Client-server protocol for file, printer, and resource sharing, also used for inter-process communication. TCP-based (three-way handshake). Access controlled via per-share ACLs, independent of local server rights. Samba is the Linux/Unix implementation; Samba v3 can join an AD domain as a member, v4 can act as an AD domain controller.

CIFS is an SMB1 dialect. NetBIOS-carried SMB (CIFS) uses TCP 137/138/139; direct SMB uses TCP 445 exclusively.

`CIFS` Windows NT 4.0, NetBIOS | `SMB 1.0` Windows 2000, direct TCP | `SMB 2.0` Vista/2008, signing+caching | `SMB 2.1` 7/2008 R2, locking | `SMB 3.0` 8/2012, multichannel+encryption+remote storage | `SMB 3.1.1` 10/2016, integrity checking+AES-128

### Default Configuration

Global settings apply server-wide; per-share settings override them. print$ and IPC$ shares exist by default.

`[sharename]` share name | `workgroup = <GROUP>` workgroup shown to clients | `path = <PATH>` directory granted access | `server string = <STRING>` string shown on connect | `unix password sync = yes` sync UNIX password with SMB password | `usershare allow guests = yes` allow unauthenticated users on defined shares | `map to guest = bad user` fallback for an invalid UNIX user login | `browseable = yes` show share in share list | `guest ok = yes` allow connecting without a password | `read only = yes` read-only access | `create mask = <OCTAL>` permissions for new files

### Dangerous Settings

`browseable = yes` + `read only = no` + `writable = yes` + `guest ok = yes` let anyone list, create, and modify files without credentials | `enable privileges = yes` honor SID-assigned privileges | `create mask = 0777` / `directory mask = 0777` world-writable new files/dirs | `logon script = <SCRIPT>` run on user login | `magic script = <SCRIPT>` run when the script file is closed | `magic output = <FILE>` where the magic script's output is stored

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -sC -p139,445
smbclient -N -L //<TARGET_IP>
smbclient //<TARGET_IP>/<SHARE>
```

```text
smb: \> ls
smb: \> get <FILE>
smb: \> !<CMD>
```

`-N` null session (anonymous, no creds). `!<cmd>` runs a local system command without breaking the connection.

```bash
rpcclient -U "" <TARGET_IP>
```

`srvinfo` server info | `enumdomains` enumerate domains | `querydominfo` domain/server/user info | `netshareenumall` enumerate all shares | `netsharegetinfo <SHARE>` info on one share | `enumdomusers` enumerate domain users | `queryuser <RID>` user info by RID | `querygroup <RID>` group info by RID

```text
rpcclient $> srvinfo
rpcclient $> enumdomains
rpcclient $> querydominfo
rpcclient $> netshareenumall
rpcclient $> netsharegetinfo <SHARE>
rpcclient $> enumdomusers
rpcclient $> queryuser <RID>
rpcclient $> querygroup <RID>
```

```bash
for i in $(seq 500 1100);do rpcclient -N -U "" <TARGET_IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
```

Brute forces RIDs to enumerate users when `queryuser` is allowed even if `enumdomusers` is restricted.

```bash
samrdump.py <TARGET_IP>
smbmap -H <TARGET_IP>
crackmapexec smb <TARGET_IP> --shares -u '' -p ''
smbstatus
```

`smbstatus` lists active SMB connections, run on the server itself.

```bash
git clone https://github.com/cddmp/enum4linux-ng.git
cd enum4linux-ng
pip3 install -r requirements.txt
./enum4linux-ng.py <TARGET_IP> -A
```

## NFS

Network File System, accesses remote filesystems over the network as if local. NFSv3 authenticates the client machine; NFSv4 authenticates the user, uses Kerberos, supports ACLs, is stateful, and needs only port 2049 (no portmapper). Uses ONC-RPC on TCP/UDP 111. No built-in authentication; relies on RPC for auth and UNIX UID/GID for authorization. Client and server UID/GID mappings don't have to match, so only trust this method on trusted networks.

`NFSv2` UDP only, older | `NFSv3` variable file size, better error reporting, not fully compatible with v2 | `NFSv4` Kerberos, firewall/Internet friendly, ACLs, stateful, single port 2049

### Default Configuration

/etc/exports lists shared filesystems and client access rules.

`rw` read/write | `ro` read only | `sync` synchronous, slower | `async` asynchronous, faster | `secure` ports <1024 only | `insecure` ports >1024 allowed | `no_subtree_check` skip subdirectory checks | `root_squash` maps root UID/GID 0 to anonymous, blocks root access to files

### Dangerous Settings

`rw` + `insecure` + `no_root_squash` together let a client write as root with root's UID/GID preserved on the share | `nohide` exports a filesystem mounted under an already-exported directory as its own separate entry

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -p111,2049 -sV -sC
sudo nmap <TARGET_IP> -p111,2049 -sV --script nfs*
showmount -e <TARGET_IP>
```

`rpcinfo` lists running RPC services/ports | `nfs-ls` lists share contents and permissions | `nfs-showmount` lists exports | `nfs-statfs` shows filesystem stats

```bash
mkdir <MOUNT_DIR>
sudo mount -t nfs <TARGET_IP>:/ ./<MOUNT_DIR>/ -o nolock
sudo umount ./<MOUNT_DIR>
```

If `root_squash` is set, files cannot be edited as root even as root locally; check real ownership with `ls -n`. If `no_root_squash` is set, upload a SUID shell to the share to escalate to another user once combined with local SSH access.

## DNS

Resolves domain names to IP addresses, no central database, distributed across name servers.

`Root` top of hierarchy, last resort | `Authoritative` binding answers for its own zone | `Non-authoritative` collects answers via recursive/iterative queries | `Caching` caches other servers' answers for a set time | `Forwarding` only forwards queries elsewhere | `Resolver` local, non-authoritative resolution

`A` IPv4 | `AAAA` IPv6 | `MX` mail servers | `NS` nameservers | `TXT` verification/SPF/DMARC/DKIM/misc | `CNAME` alias to another domain | `PTR` reverse lookup, IP to name | `SOA` zone info plus admin email

```bash
dig soa <TARGET_DOMAIN>
```

SOA admin email replaces the first `.` with `@` (hostmaster.inwx.net. → hostmaster@inwx.net).

### Default Configuration

Bind9 local config in named.conf (named.conf.local, named.conf.options, named.conf.log). Zone options override global options. A zone file needs exactly one SOA and at least one NS record; a syntax error makes the whole zone unusable and the server answers with SERVFAIL. Reverse lookup zone files map hostnames to the last IP octet via PTR records.

### Dangerous Settings

`allow-query` hosts allowed to query | `allow-recursion` hosts allowed recursive queries | `allow-transfer` hosts allowed zone transfers | `zone-statistics` collects zone statistics

### Footprinting the Service

```bash
dig ns <TARGET_DOMAIN> @<TARGET_IP>
dig CH TXT version.bind <TARGET_IP>
dig any <TARGET_DOMAIN> @<TARGET_IP>
dig axfr <TARGET_DOMAIN> @<TARGET_IP>
```

Zone transfer (AXFR) runs over TCP 53 to sync zone data between master/slave DNS servers. If `allow-transfer` is misconfigured (a wide subnet or `any`), anyone can pull the full zone file, including internal hostnames and IPs.

```bash
for sub in $(cat subdomains-top1million-110000.txt);do dig $sub.<TARGET_DOMAIN> @<TARGET_IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt;done
dnsenum --dnsserver <TARGET_IP> --enum -p 0 -s 0 -o subdomains.txt -f subdomains-top1million-110000.txt <TARGET_DOMAIN>
```

## SMTP

Sends email over IP networks. Port 25 by default, port 587 for authenticated submission with STARTTLS, port 465 for implicit TLS. Unencrypted by default; ESMTP adds STARTTLS and AUTH PLAIN.

Flow: client (MUA) → submission agent (MSA/relay) → MTA → MDA → mailbox (POP3/IMAP). An open relay lets anyone relay mail through the server (spam, spoofing).

`AUTH PLAIN` authenticate client | `HELO`/`EHLO` start session | `MAIL FROM` declare sender | `RCPT TO` declare recipient | `DATA` start message body | `RSET` abort transmission, keep connection | `VRFY` check if a mailbox exists, can enumerate users | `EXPN` check if a mailbox exists | `NOOP` keep-alive | `QUIT` end session

### Dangerous Settings

```text
mynetworks = 0.0.0.0/0
```

Allows open relay, any source IP can relay mail through the server.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sC -sV -p25
sudo nmap <TARGET_IP> -p25 --script smtp-open-relay -v
```

`smtp-commands` lists supported commands via EHLO | `smtp-open-relay` runs 16 open-relay tests

```bash
telnet <TARGET_IP> 25
```

```text
HELO <TARGET_DOMAIN>
EHLO <TARGET_DOMAIN>
VRFY <USERNAME>
MAIL FROM: <<USERNAME>@<TARGET_DOMAIN>>
RCPT TO: <<USERNAME_2>@<TARGET_DOMAIN>> NOTIFY=success,failure
DATA
QUIT
```

`VRFY` can return code 252 for a name regardless of whether it exists, giving false positives.

Through a web proxy:

```http
CONNECT <TARGET_IP>:25 HTTP/1.0
```

## IMAP / POP3

IMAP: online email management on the server, folders, multiple simultaneous clients, port 143 (993 TLS). POP3: only list/retrieve/delete, no server-side folder management, port 110 (995 TLS). Both plaintext by default.

`1 LOGIN <USERNAME> <PASSWORD>` login | `1 LIST "" *` list directories | `1 CREATE "<FOLDER>"` create mailbox | `1 DELETE "<FOLDER>"` delete mailbox | `1 RENAME "<OLD>" "<NEW>"` rename mailbox | `1 SELECT INBOX` select mailbox | `1 FETCH <ID> all` retrieve message | `1 CLOSE` purge deleted | `1 LOGOUT` close connection

`USER <USERNAME>` identify user | `PASS <PASSWORD>` authenticate | `STAT` number of saved emails | `LIST` size/number of emails | `RETR <ID>` retrieve email | `DELE <ID>` delete email | `CAPA` show server capabilities | `RSET` reset session | `QUIT` close connection

### Dangerous Settings

`auth_debug` logs all auth debug info | `auth_debug_passwords` logs submitted passwords+scheme | `auth_verbose` logs failed auth attempts and reasons | `auth_verbose_passwords` logs passwords used for auth | `auth_anonymous_username` sets the username for ANONYMOUS SASL login

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD>
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD> -v
openssl s_client -connect <TARGET_IP>:pop3s
openssl s_client -connect <TARGET_IP>:imaps
```

## SNMP

Monitors and manages network devices remotely (routers, switches, IoT). Client sends commands to agents over UDP 161. Traps (unsolicited server-to-client notifications) use UDP 162.

A MIB is a text file (ASN.1) listing a device's queryable SNMP objects in a tree. An OID is a dotted-integer address identifying a node in that tree; a longer chain is more specific.

`SNMPv1` no auth, no encryption, plaintext | `SNMPv2c` community-string based, still plaintext | `SNMPv3` username/password auth plus pre-shared-key encryption, more complex

Community strings act as passwords and are sent in plaintext on v1/v2c, so they can be sniffed.

### Dangerous Settings

`rwuser noauth` full OID tree access without authentication | `rwcommunity <STRING> <IPv4>` full OID tree access from any source IP | `rwcommunity6 <STRING> <IPv6>` same for IPv6

### Footprinting the Service

```bash
snmpwalk -v2c -c <STRING> <TARGET_IP>
onesixtyone -c <WORDLIST> <TARGET_IP>
braa <STRING>@<TARGET_IP>:.1.3.6.*
```

`snmpwalk` queries OIDs with their data. `onesixtyone` brute forces community strings. `braa` brute forces individual OIDs once the community string is known.

## MySQL

Open-source SQL RDBMS, client-server. Data stored in tables (columns, rows, types), often exported/backed up as .sql files. MariaDB is a fork of MySQL by its original lead developer.

### Dangerous Settings

`user` MySQL service run-as user, plaintext in config | `password` MySQL user password, plaintext in config | `admin_address` IP for the admin TCP/IP interface, plaintext | `debug` verbose debug output on errors | `sql_warnings` info string on single-row INSERT warnings | `secure_file_priv` limits data import/export operations

`debug` and `sql_warnings` can leak sensitive info via verbose errors on web apps, sometimes exploitable via SQLi to run system commands.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -sC -p3306 --script mysql*
mysql -u <USERNAME> -p<PASSWORD> -h <TARGET_IP>
```

`show databases;` list DBs | `use <DB>;` select DB | `show tables;` list tables | `show columns from <TABLE>;` list columns | `select * from <TABLE>;` dump table | `select * from <TABLE> where <COLUMN> = "<STRING>";` search

Nmap's `mysql-brute`/`mysql-enum` scripts can report false-positive empty passwords; confirm manually with the client.

## MSSQL

Microsoft's closed-source RDBMS, originally Windows-only, strong .NET integration.

### MSSQL Clients

SSMS is a GUI client for initial config and long-term DB management, installable separately from the server (may hold saved creds on an admin's workstation). Other clients: mssql-cli, SQL Server PowerShell, HeidiSQL, SQLPro, Impacket's mssqlclient.py.

```bash
locate mssqlclient
```

### MSSQL Databases

`master` tracks all instance system info | `model` template for every new database | `msdb` SQL Server Agent jobs and alerts | `tempdb` temporary objects | `resource` read-only system objects

### Default Configuration

Service runs by default as NT SERVICE\MSSQLSERVER. Windows Authentication uses the local SAM or AD to authorize logins; encryption is not enforced by default.

### Dangerous Settings

Clients connecting without encryption | self-signed certs, spoofable | named pipes enabled | weak or default `sa` credentials

### Footprinting the Service

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=<PORT>,mssql.username=<USERNAME>,mssql.password=<PASSWORD>,mssql.instance-name=MSSQLSERVER -sV -p 1433 <TARGET_IP>
```

```text
msf6 > use auxiliary/scanner/mssql/mssql_ping
msf6 auxiliary(scanner/mssql/mssql_ping) > set rhosts <TARGET_IP>
msf6 auxiliary(scanner/mssql/mssql_ping) > run
```

#### Connecting with Mssqlclient.py

```bash
python3 mssqlclient.py <USERNAME>@<TARGET_IP> -windows-auth
```

```text
SQL> select name from sys.databases
```
## Oracle TNS

Communication protocol between Oracle databases and client applications, supports TCP/IP and IPX/SPX, has built-in encryption. Listener defaults to TCP 1521. Remotely manageable in Oracle 8i/9i, not in 10g/11g by default.

Config files: tnsnames.ora (client-side, resolves service names to addresses) and listener.ora (server-side, defines what the listener serves), under $ORACLE_HOME/network/admin.

`DESCRIPTION` name and connection type for the DB | `ADDRESS` hostname and port | `PROTOCOL` network protocol used | `PORT` port used | `CONNECT_DATA` service name/SID, protocol, instance | `INSTANCE_NAME` instance to connect to | `SERVICE_NAME` service to connect to | `SERVER` dedicated or shared | `USER`/`PASSWORD` auth credentials | `SECURITY` connection security type | `VALIDATE_CERT` validate SSL/TLS cert | `SSL_VERSION` TLS version | `CONNECT_TIMEOUT`/`RECEIVE_TIMEOUT`/`SEND_TIMEOUT` connection time limits | `SQLNET.EXPIRE_TIME` time to detect a dead connection | `TRACE_LEVEL`/`TRACE_DIRECTORY`/`TRACE_FILE_NAME`/`LOG_FILE` tracing and logging

Oracle 9 default password: `CHANGE_ON_INSTALL`. Oracle 10+ has no default password. DBSNMP service default password: `dbsnmp`.

A PL/SQL Exclusion List (PlsqlExclusionList) can blacklist PL/SQL packages/types from execution, placed in $ORACLE_HOME/sqldeveloper.

### Setting up ODAT

```bash
sudo apt-get update
sudo apt-get install -y build-essential python3-dev libaio1
wget https://files.pythonhosted.org/packages/source/c/cx_Oracle/cx_Oracle-8.3.0.tar.gz
tar xzf cx_Oracle-8.3.0.tar.gz
cd cx_Oracle-8.3.0
python3 setup.py build
sudo python3 setup.py install
cd ~
git clone https://github.com/quentinhardy/odat.git
cd odat/
pip install python-libnmap
git submodule init
git submodule update
sudo apt-get install python3-scapy -y
sudo pip3 install colorlog termcolor passlib python-libnmap
sudo apt-get install build-essential libgmp-dev -y
pip3 install pycryptodome
pip3 install openpyxl
```

### Footprinting the Service

```bash
sudo nmap -p1521 -sV <TARGET_IP> --open
sudo nmap -p1521 -sV <TARGET_IP> --open --script oracle-sid-brute
```

A System Identifier (SID) identifies a specific database instance; the client must specify the correct SID to connect.

```bash
./odat.py all -s <TARGET_IP>
```

Runs all ODAT modules: enumerates DB names, versions, processes, users, vulnerabilities, misconfigurations, and brute forces credentials.

### Setting up SQLplus

```bash
sudo apt update
sudo apt upgrade parrot-core
sudo apt update
sudo apt install oracle-instantclient-sqlplus
sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf";sudo ldconfig
sqlplus -v
```

### SQLplus Interaction

```bash
sqlplus <USERNAME>/<PASSWORD>@<TARGET_IP>/<SID>
sqlplus <USERNAME>/<PASSWORD>@<TARGET_IP>/<SID> as sysdba
```

```text
SQL> select table_name from all_tables;
SQL> select * from user_role_privs;
SQL> select name, password from sys.user$;
```

`as sysdba` attempts login with elevated System Database Admin privileges if the account has them. `sys.user$` holds password hashes for offline cracking.

### File Upload via ODAT

```bash
echo "<STRING>" > testing.txt
./odat.py utlfile -s <TARGET_IP> -d <SID> -U <USERNAME> -P <PASSWORD> --sysdba --putFile <TARGET_PATH> testing.txt ./testing.txt
```

Default webroot paths: Linux `/var/www/html`, Windows `C:\inetpub\wwwroot`.

```bash
curl -X GET http://<TARGET_IP>/testing.txt
```

## IPMI

Hardware-based host management, independent of OS/BIOS/CPU. Works even when the host is powered off, as long as the IPMI module (BMC) has power and a LAN connection. Used pre-boot for BIOS config, when the host is fully off, or after a system failure. Also monitors temperature, voltage, fans, power, and can alert via SNMP.

Requires: BMC (micro-controller core), ICMB (inter-chassis comms), IPMB (extends the BMC), IPMI memory (event log, repository data), communications interfaces (local, serial, LAN, PCI management bus).

### Footprinting the Service

UDP port 623. Common BMCs: HP iLO, Dell DRAC, Supermicro IPMI. Gaining BMC access is near-equivalent to physical access (reboot, power off, reinstall OS).

```bash
sudo nmap -sU --script ipmi-version -p 623 <TARGET_IP>
```

```text
msf6 > use auxiliary/scanner/ipmi/ipmi_version
msf6 auxiliary(scanner/ipmi/ipmi_version) > set rhosts <TARGET_IP>
msf6 auxiliary(scanner/ipmi/ipmi_version) > run
```

`Dell iDRAC` root / calvin | `HP iLO` Administrator / randomized 8-char string | `Supermicro IPMI` ADMIN / ADMIN

### Dangerous Settings

IPMI 2.0's RAKP protocol sends a salted SHA1/MD5 hash of the user's password to the client before authentication, for ANY valid username. This hash can be captured and cracked offline. No fix exists since it is core to the spec; mitigate with long passwords or network segmentation.

```text
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > set rhosts <TARGET_IP>
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > run
```

```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

Mode 7300 cracks IPMI RAKP hashes. Mask tries all upper-case letter + digit combos for an 8-char password (e.g. default HP iLO passwords).

## Linux Remote Management Protocols

### SSH

Encrypted remote connection, standard TCP port 22, works cross-platform. OpenSSH is the common open-source fork. SSH-2 improves on SSH-1 (SSH-1 is vulnerable to MITM).

Auth methods: password, public-key, host-based, keyboard, challenge-response, GSSAPI.

Public key auth: server sends its host key to verify server identity, then client proves identity via its private key solving a server-issued cryptographic challenge. Private key stays local, protected by a passphrase; public key is stored server-side.

#### Dangerous Settings

`PasswordAuthentication yes` allows password brute-forcing | `PermitEmptyPasswords yes` allows blank passwords | `PermitRootLogin yes` allows root login | `Protocol 1` outdated, weak encryption | `X11Forwarding yes` GUI app forwarding, had a command injection CVE in OpenSSH 7.2p1 (2016) | `AllowTcpForwarding yes` allows TCP port forwarding | `PermitTunnel` allows tunneling | `DebianBanner yes` shows a distro-specific banner

#### Footprinting the Service

```bash
git clone https://github.com/jtesta/ssh-audit.git && cd ssh-audit
./ssh-audit.py <TARGET_IP>
```

Shows the banner (reveals OpenSSH version, e.g. vulnerable to CVE-2020-14145 MITM on initial connection) and supported algorithms.

```bash
ssh -v <USERNAME>@<TARGET_IP>
ssh -v <USERNAME>@<TARGET_IP> -o PreferredAuthentications=password
```

Banner format: protocol version + server version, e.g. `SSH-1.99-OpenSSH_3.9p1` supports both SSH-1/2; `SSH-2.0-OpenSSH_8.2p1` is SSH-2 only.

### Rsync

Fast file copy/sync tool, local and remote, delta-transfer algorithm (only sends file differences). Default port 873. Can tunnel over SSH.

```bash
sudo nmap -sV -p 873 <TARGET_IP>
nc -nv <TARGET_IP> 873
```

```text
#list
```

Lists available rsync modules/shares via raw protocol.

```bash
rsync -av --list-only rsync://<TARGET_IP>/<SHARE>
rsync -av rsync://<TARGET_IP>/<SHARE> <OUTPUT_DIR>
rsync -av -e ssh rsync://<TARGET_IP>/<SHARE> <OUTPUT_DIR>
```

`-e ssh` tunnels the sync over SSH if the server requires it; use `-e "ssh -p<PORT>"` for a non-standard SSH port.

### R-Services

Legacy Unix remote access suite, unencrypted, superseded by SSH. Ports 512/513/514.

`rcp` rshd, 514/TCP, copy files, no overwrite warning | `rsh` rshd, 514/TCP, opens a shell with no login, trusts hosts.equiv/.rhosts | `rexec` rexecd, 512/TCP, runs commands, needs user/pass unless trusted via hosts.equiv/.rhosts | `rlogin` rlogind, 513/TCP, remote login like telnet but Unix-only, trusts hosts.equiv/.rhosts

/etc/hosts.equiv (global) and ~/.rhosts (per-user) list trusted `<hostname> <username>` pairs that bypass authentication entirely. `+` is a wildcard for any host/user.

```text
htb-student     10.0.17.5
+               10.0.17.10
+               +
```

#### Footprinting and Abuse

```bash
sudo nmap -sV -p 512,513,514 <TARGET_IP>
rlogin <TARGET_IP> -l <USERNAME>
rwho
rusers -al <TARGET_IP>
```

`rwho` lists interactive sessions on the local network (broadcast-based, UDP 513). `rusers` gives more detail: username, host, TTY, login time, idle time, origin host.

## Windows Remote Management Protocols

### RDP

Microsoft's GUI remote access protocol, TCP (and optionally UDP) port 3389. Needs firewall + NAT port forwarding to reach internal hosts externally. TLS/SSL supported since Vista, but many servers keep weak encryption and self-signed certs, so clients can't verify authenticity (cert warning). Default setting requires Network Level Authentication (NLA).

#### Footprinting the Service

```bash
nmap -sV -sC <TARGET_IP> -p3389 --script rdp*
```

`rdp-enum-encryption` shows supported security layers (NLA, RDSTLS) | `rdp-ntlm-info` shows hostname, domain, product version

```bash
nmap -sV -sC <TARGET_IP> -p3389 --packet-trace --disable-arp-ping -n
```

Nmap's RDP cookie (`mstshash=nmap`) can be fingerprinted by EDR/threat hunters, risking detection/lockout.

```bash
sudo cpan
```

```text
cpan[1]> install Encoding::BER
```

```bash
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git && cd rdp-sec-check
./rdp-sec-check.pl <TARGET_IP>
```

Unauthenticated check of RDP server security settings via handshake analysis.

```bash
xfreerdp /u:<USERNAME> /p:"<PASSWORD>" /v:<TARGET_IP>
```

### WinRM

SOAP-based Windows remote management, must be explicitly enabled pre-Windows 10 (enabled by default from Server 2012). TCP 5985 (HTTP) and 5986 (HTTPS). WinRS lets WinRM execute arbitrary remote commands.

#### Footprinting the Service

```bash
nmap -sV -sC <TARGET_IP> -p5985,5986 --disable-arp-ping -n
```

```powershell
Test-WsMan <TARGET_IP>
```

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>
```

### WMI

Microsoft's CIM/WBEM implementation, read/write access to nearly all Windows settings, accessed via PowerShell, VBScript, or WMIC.

#### Footprinting the Service

Initializes on TCP 135, then moves to a random port.

```bash
wmiexec.py <USERNAME>:"<PASSWORD>"@<TARGET_IP> "<CMD>"
```