# Enumeration

## Enumeration Principles

Enumeration = active + passive information gathering. OSINT is separate, passive only. Loop: every finding feeds the next round.

Three principles:
1. There is more than meets the eye — consider all points of view
2. Distinguish between what we see and what we do not see
3. There are always ways to gain more information — understand the target

The goal is not to get at the systems but to find all the ways in. Brute-forcing SSH/RDP/WinRM on first contact is the wrong move — noisy, gets you blacklisted, and you learn nothing about the infra.

## Enumeration Methodology
![[Pasted image 20261024042605.png]]

Six nested layers — each a boundary to pass. The first two don't fully apply to intranet/AD.

1. **Internet Presence** — Domains, Subdomains, vHosts, ASN, Netblocks, IP addresses, Cloud instances, Security measures
2. **Gateway** — Firewalls, DMZ, IPS/IDS, EDR, Proxies, NAC, Network segmentation, VPN, Cloudflare
3. **Accessible Services** — Service type, Functionality, Configuration, Port, Version, Interface
4. **Processes** — PID, Processed data, Tasks, Source, Destination
5. **Privileges** — Groups, Users, Permissions, Restrictions, Environment
6. **OS Setup** — OS type, Patch level, Network config, OS environment, Config files, Sensitive private files

Not every gap leads inside. Methodology is not a step-by-step guide — it's the summary of systematic procedures. Tools and commands are a cheat sheet, not part of the methodology.

## Domain Information

Passive gathering — stay as a "customer/visitor", don't touch the target directly.

Start from the main website. Read the services offered; each implies specific tech/infra requirements.

**SSL cert** on the main site often lists more domains than expected (SAN field).

**crt.sh** — certificate transparency logs:

```text
https://www.crt.sh/?q=<TARGET_DOMAIN>
```

JSON output:

```bash
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq .
```

Filtered to unique subdomains:

```bash
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

Identify company-hosted hosts (not third-party):

```bash
for i in $(cat subdomainlist);do host $i | grep "has address" | grep <TARGET_DOMAIN> | cut -d" " -f1,4;done
```

**Shodan** — passive port/service lookup:

```bash
for i in $(cat subdomainlist);do host $i | grep "has address" | grep <TARGET_DOMAIN> | cut -d" " -f4 >> ip-addresses.txt;done
for i in $(cat ip-addresses.txt);do shodan host $i;done
```

Full DNS record dump:

```bash
dig any <TARGET_DOMAIN>
```

Interpretation:

- **A** — IPs pointing to a (sub)domain
- **MX** — mail provider; note and skip if third-party
- **NS** — name servers; usually reveals hosting provider
- **TXT** — verification keys, SPF, DMARC, DKIM — the gold record
- **SOA** — zone admin contact

TXT often leaks third-party services: Atlassian → software dev/collab; Google → Gmail/GDrive; LogMeIn → remote access; Mailgun → email APIs/SMTP relays/webhooks; Outlook/Office 365 → OneDrive + Azure blob/file storage; INWX → hosting/domain registrar. Each implies concrete follow-up (IDOR/SSRF on Mailgun APIs, Azure Files = SMB, password reuse on LogMeIn = everything).

SPF `ip4:` entries inside TXT often leak additional IPs.

## Cloud Resources

Cloud storage (S3 buckets, Azure blobs, GCP storage) frequently ends up in DNS when used for admin purposes. Misconfigured → readable without auth.

Google dorks:

```text
intext:<COMPANY> inurl:amazonaws.com
intext:<COMPANY> inurl:blob.core.windows.net
```

Check site source code — images/JS/CSS often load from cloud storage; DNS prefetch and preconnect links expose blob URLs.

Third-party tools:
- **domain.glass** — infra overview, Cloudflare security rating (also tells you a security measure exists → layer 2)
- **GrayHatWarfare** — searchable index of S3/Azure/GCP buckets, filterable by file type

Search company-name abbreviations too. Files worth grabbing from an open bucket: PDFs, docs, presentations, source, leaked SSH keys (`id_rsa`, `id_rsa.pub`).

## Staff

Job posts + LinkedIn profiles describe the stack without a single scan.

From job posts: programming languages, databases, web frameworks, REST APIs, version control, Atlassian suite, containerization. Each item = concrete enumeration target.

From employee profiles: specific technologies (React, Svelte, Angular, Kafka, Elastic) and sometimes personal email or hardcoded JWT in a public repo.

Filter LinkedIn by company/school/title. Look for both dev and security staff — the security team's skills tell you what defensive measures to expect.

## FTP

Two channels — control on TCP 21, data on TCP 20. Active vs passive matters when firewalls are in play; passive makes the client initiate the data connection.

**vsFTPd** default config `/etc/vsftpd.conf`, plus `/etc/ftpusers` for explicit login deny list.

```bash
cat /etc/vsftpd.conf | grep -v "#"
cat /etc/ftpusers
```

Key settings to recognise:

- `anonymous_enable=YES` — anonymous login
- `anon_upload_enable=YES` — anonymous write
- `anon_mkdir_write_enable=YES` — anonymous dir creation
- `no_anon_password=YES` — no password prompt for anon
- `anon_root=/home/username/ftp` — anon root
- `write_enable=YES` — enables STOR, DELE, RNFR, RNTO, MKD, RMD, APPE, SITE
- `dirmessage_enable=YES` — message on dir entry
- `chown_uploads=YES` + `chown_username=username` — ownership of anon uploads
- `hide_ids=YES` — all files shown as `ftp:ftp`
- `ls_recurse_enable=YES` — enables `ls -R`

Anonymous login:

```bash
ftp <TARGET_IP>
```

Inside: `status`, `debug`, `trace`, `ls`, `ls -R`, `get <file>`, `put <file>`.

Bulk download:

```bash
wget -m --no-passive ftp://anonymous:anonymous@<TARGET_IP>
```

Footprinting:

```bash
sudo nmap -sV -p21 -sC -A <TARGET_IP>
sudo nmap -sV -p21 -sC -A <TARGET_IP> --script-trace
```

Useful NSE scripts: `ftp-anon`, `ftp-syst`, `ftp-brute`, `ftp-proftpd-backdoor`, `ftp-vsftpd-backdoor`.

Manual interaction:

```bash
nc -nv <TARGET_IP> 21
telnet <TARGET_IP> 21
openssl s_client -connect <TARGET_IP>:21 -starttls ftp    # TLS + cert
```

TLS cert often leaks hostname and admin email.

## SMB

**SMB** — Windows file-sharing protocol. Versions: CIFS/SMB1 (outdated), SMB2.x, SMB3.x. **Samba** is the Unix implementation (`smbd` and `nmbd`).

Default config `/etc/samba/smb.conf`:

```bash
cat /etc/samba/smb.conf | grep -v "#\|\;"
```

Dangerous settings:

- `browseable = yes` — list shares from within a share
- `read only = no` / `writable = yes` — write access
- `guest ok = yes` — no password
- `enable privileges = yes` — honors SID-assigned privileges
- `create mask = 0777` / `directory mask = 0777` — world-writable new files/dirs
- `logon script = script.sh` — script executed on user login
- `magic script = script.sh` — script executed when closed

Null-session enumeration:

```bash
smbclient -N -L //<TARGET_IP>
smbclient //<TARGET_IP>/<SHARE>
```

Inside smbclient: `help`, `ls`, `get <file>`, `!ls`, `!cat <file>`.

**RPC enumeration**:

```bash
rpcclient -U "" <TARGET_IP>
```

Queries:
- `srvinfo`
- `enumdomains`
- `querydominfo`
- `netshareenumall`
- `netsharegetinfo <SHARE>`
- `enumdomusers`
- `queryuser <RID>`
- `querygroup <RID>`

RID brute force:

```bash
for i in $(seq 500 1100);do rpcclient -N -U "" <TARGET_IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
```

Impacket `samrdump.py`:

```bash
samrdump.py <TARGET_IP>
```

Other enumerators:

```bash
smbmap -H <TARGET_IP>
crackmapexec smb <TARGET_IP> --shares -u '' -p ''
./enum4linux-ng.py <TARGET_IP> -A
```

`enum4linux-ng` covers NetBIOS, SMB dialect, null session, domain info, OS info, users, groups, shares, policies. Cross-check with others anyway.

Server-side status:

```bash
smbstatus
```

## NFS

**NFS** — Unix/Linux file-sharing. NFSv3 auths the client machine; NFSv4 auths the user (Kerberos). Exports table `/etc/exports`:

```bash
cat /etc/exports
```

Options:
- `rw` / `ro`
- `sync` / `async`
- `secure` / `insecure` — ports >1024 rejected / allowed
- `no_subtree_check`
- `root_squash` — root UID/GID mapped to anonymous

Dangerous:
- `rw`
- `insecure`
- `nohide` — nested mounts re-exported separately
- `no_root_squash` — root-created files keep UID/GID 0 (write a SUID shell here)

If `root_squash` is set, you cannot edit files as root.

Footprinting:

```bash
sudo nmap <TARGET_IP> -p111,2049 -sV -sC
sudo nmap --script nfs* <TARGET_IP> -sV -p111,2049
```

Show and mount:

```bash
showmount -e <TARGET_IP>
mkdir target-NFS
sudo mount -t nfs <TARGET_IP>:/ ./target-NFS/ -o nolock
```

Inspect ownership — `ls -l` for names, `ls -n` for UID/GID. Match UIDs on your system to read/write.

Unmount:

```bash
cd ..
sudo umount ./target-NFS
```

For escalation: upload a shell to the share with SUID set for the target user, execute via SSH session as that user.

## DNS
![[Pasted image 20261024042718.png]]
Records: **A / AAAA**, **MX**, **NS**, **TXT**, **CNAME**, **PTR**, **SOA**.

`dig soa <TARGET_DOMAIN>` — admin email with `.` replaced by `@`.

Config files (Bind9): `/etc/bind/named.conf.local`, `named.conf.options`, `named.conf.log`, `db.domain.com`, `db.10.129.14`.

Dangerous settings:
- `allow-query`
- `allow-recursion`
- `allow-transfer` — if `any` or broad subnet → full zone dump
- `zone-statistics`

Footprinting:

```bash
dig ns <TARGET_DOMAIN> @<TARGET_IP>
dig CH TXT version.bind <TARGET_IP>       # Bind version disclosure
dig any <TARGET_DOMAIN> @<TARGET_IP>       # all records the server will disclose
dig axfr <TARGET_DOMAIN> @<TARGET_IP>      # zone transfer
dig axfr internal.<TARGET_DOMAIN> @<TARGET_IP>
```

Subdomain brute force:

```bash
for sub in $(cat subdomains-top1million-110000.txt);do dig $sub.<TARGET_DOMAIN> @<TARGET_IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt;done
```

```bash
dnsenum --dnsserver <TARGET_IP> --enum -p 0 -s 0 -o subdomains.txt -f subdomains-top1million-110000.txt <TARGET_DOMAIN>
```

## SMTP

Sending only — retrieval is IMAP/POP3. Default TCP 25; submission on 587 with STARTTLS; implicit TLS on 465.

Postfix config `/etc/postfix/main.cf`:

```bash
cat /etc/postfix/main.cf | grep -v "#" | sed -r "/^\s*$/d"
```

Commands:
- `AUTH PLAIN`
- `HELO <name>`
- `EHLO <name>`
- `MAIL FROM:<addr>`
- `RCPT TO:<addr>`
- `DATA` — end with `.` on its own line
- `RSET`
- `VRFY <name>` — unreliable, often 252 for everything
- `EXPN`
- `NOOP`
- `QUIT`

Manual session:

```bash
telnet <TARGET_IP> 25
```

```text
EHLO <DOMAIN>
MAIL FROM: <sender@<TARGET_DOMAIN>>
RCPT TO: <recipient@<TARGET_DOMAIN>> NOTIFY=success,failure
DATA
From: <sender@<TARGET_DOMAIN>>
To: <recipient@<TARGET_DOMAIN>>
Subject: test
.
QUIT
```

Open relay config:

```text
mynetworks = 0.0.0.0/0
```

Footprinting:

```bash
sudo nmap <TARGET_IP> -sC -sV -p25
sudo nmap <TARGET_IP> -p25 --script smtp-open-relay -v
```

`smtp-open-relay` runs 16 relay tests.

## IMAP / POP3

IMAP = online mailbox management (folders, multiple mailboxes). POP3 = list, retrieve, delete. IMAP over 143 / 993 (TLS); POP3 over 110 / 995.

IMAP commands:
- `1 LOGIN <USERNAME> <PASSWORD>`
- `1 LIST "" *`
- `1 LSUB "" *`
- `1 SELECT INBOX`
- `1 FETCH <ID> all`
- `1 UNSELECT INBOX`
- `1 CREATE "INBOX"`
- `1 DELETE "INBOX"`
- `1 RENAME "ToRead" "Important"`
- `1 CLOSE`
- `1 LOGOUT`

POP3 commands:
- `USER <USERNAME>`
- `PASS <PASSWORD>`
- `STAT`
- `LIST`
- `RETR <id>`
- `DELE <id>`
- `CAPA`
- `RSET`
- `QUIT`

Dangerous Dovecot settings:
- `auth_debug`
- `auth_debug_passwords`
- `auth_verbose`
- `auth_verbose_passwords`
- `auth_anonymous_username`

Footprinting:

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
```

With credentials:

```bash
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD>
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD> -v
```

Direct TLS:

```bash
openssl s_client -connect <TARGET_IP>:pop3s
openssl s_client -connect <TARGET_IP>:imaps
```

## SNMP

UDP 161 queries, UDP 162 traps. **MIB** = tree of queryable objects. **OID** = node.

Versions:
- v1 — no auth, no encryption
- v2c — community string, plaintext
- v3 — username/password auth + encryption

Community strings = passwords, sent in plaintext.

Default config `/etc/snmp/snmpd.conf`:

```bash
cat /etc/snmp/snmpd.conf | grep -v "#" | sed -r '/^\s*$/d'
```

Dangerous:
- `rwuser noauth`
- `rwcommunity <string> <IPv4>`
- `rwcommunity6 <string> <IPv6>`

Footprinting:

```bash
snmpwalk -v2c -c <COMMUNITY> <TARGET_IP>
```

Brute-force community strings:

```bash
onesixtyone -c snmp.txt <TARGET_IP>
```

Brute-force OIDs:

```bash
braa <COMMUNITY>@<TARGET_IP>:.1.3.6.*
```

## MySQL

Default TCP 3306. MariaDB is a fork, mostly interchangeable.

Config `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```bash
cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep -v "#" | sed -r '/^\s*$/d'
```

Security-relevant settings:
- `user`
- `password` — plaintext
- `admin_address`
- `debug`
- `sql_warnings`
- `secure_file_priv`

Footprinting:

```bash
sudo nmap <TARGET_IP> -sV -sC -p3306 --script mysql*
```

NSE output: `mysql-empty-password`, `mysql-enum`, `mysql-info`. `mysql-brute` false-positives common — verify by hand:

```bash
mysql -u root -h <TARGET_IP>
```

With credentials:

```bash
mysql -u <USERNAME> -p<PASSWORD> -h <TARGET_IP>
```

No space between `-p` and password.

Commands:

```sql
show databases;
use <DATABASE>;
show tables;
show columns from <TABLE>;
select * from <TABLE>;
select * from <TABLE> where <COLUMN> = "<STRING>";
select version();
```

## MSSQL

Default TCP 1433. Windows Authentication (SAM or DC) or SQL logins.

Default system databases: `master`, `model`, `tempdb`, `msdb`, `resource`.

Dangerous:
- clients connecting without encryption
- self-signed certs
- named pipes enabled
- weak/default `sa` credentials

Footprinting:

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 <TARGET_IP>
```

Metasploit alternative:

```text
use auxiliary/scanner/mssql/mssql_ping
set rhosts <TARGET_IP>
run
```

Connect with Impacket:

```bash
mssqlclient.py <USERNAME>@<TARGET_IP> -windows-auth
mssqlclient.py <USERNAME>:<PASSWORD>@<TARGET_IP>
```

Inside:

```sql
select name from sys.databases
```

## Oracle TNS

Default TCP 1521. **SID** identifies the instance.

Config files: `tnsnames.ora` (client), `listener.ora` (server) in `$ORACLE_HOME/network/admin`.

Default credentials:
- Oracle 9: `CHANGE_ON_INSTALL`
- Oracle 10+: none
- DBSNMP: `dbsnmp`

`tnsnames.ora` descriptor:

```text
ORCL =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = <TARGET_IP>)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
```

Footprinting:

```bash
sudo nmap -p1521 -sV <TARGET_IP> --open
sudo nmap -p1521 -sV <TARGET_IP> --open --script oracle-sid-brute
```

**ODAT**:

```bash
./odat.py all -s <TARGET_IP>
```

Common finds: `scott/tiger`, default accounts (`dbsnmp`, `mdsys`, `outln`, `oracle_ocm`).

Connect via sqlplus:

```bash
sqlplus <USERNAME>/<PASSWORD>@<TARGET_IP>/<SID>
```

Inside:

```sql
select table_name from all_tables;
select * from user_role_privs;
select * from session_privs;
```

Escalate to sysdba:

```bash
sqlplus <USERNAME>/<PASSWORD>@<TARGET_IP>/<SID> as sysdba
```

Hash dump:

```sql
select name, password from sys.user$;
```

File upload via `utlfile` (requires web root path):

```bash
./odat.py utlfile -s <TARGET_IP> -d <SID> -U <USERNAME> -P <PASSWORD> --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
curl -X GET http://<TARGET_IP>/testing.txt
```

## IPMI

UDP 623. **BMC** — embedded ARM Linux on motherboard. Common: HP iLO, Dell DRAC, Supermicro IPMI.

Footprinting:

```bash
sudo nmap -sU --script ipmi-version -p 623 <TARGET_IP>
```

Metasploit:

```text
use auxiliary/scanner/ipmi/ipmi_version
set rhosts <TARGET_IP>
run
```

Default credentials:
- Dell iDRAC: `root` / `calvin`
- HP iLO: `Administrator` / randomized 8-char (numbers + uppercase)
- Supermicro IPMI: `ADMIN` / `ADMIN`

**RAKP flaw** — server sends salted SHA1/MD5 hash before auth. Crackable offline.

Dump hashes:

```text
use auxiliary/scanner/ipmi/ipmi_dumphashes
set rhosts <TARGET_IP>
run
```

Hashcat mode 7300. HP iLO factory 8-char mask attack:

```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

No patch exists.

## Linux Remote Management Protocols

### SSH

TCP 22. OpenSSH on Linux. SSH-2 only.

Config `/etc/ssh/sshd_config`:

```bash
cat /etc/ssh/sshd_config | grep -v "#" | sed -r '/^\s*$/d'
```

Dangerous settings:
- `PasswordAuthentication yes`
- `PermitEmptyPasswords yes`
- `PermitRootLogin yes`
- `Protocol 1`
- `X11Forwarding yes`
- `AllowTcpForwarding yes`
- `PermitTunnel`
- `DebianBanner yes`

Fingerprint:

```bash
./ssh-audit.py <TARGET_IP>
```

Check auth methods:

```bash
ssh -v <USERNAME>@<TARGET_IP>
```

Force password auth:

```bash
ssh -v <USERNAME>@<TARGET_IP> -o PreferredAuthentications=password
```

Banner reading: `SSH-1.99-OpenSSH_3.9p1` = SSH-1 and SSH-2; `SSH-2.0-OpenSSH_8.2p1` = SSH-2 only.

### Rsync

TCP 873. Delta-transfer, backups.

Scan:

```bash
sudo nmap -sV -p 873 <TARGET_IP>
```

Probe:

```bash
nc -nv <TARGET_IP> 873
```

```text
#list
```

List share:

```bash
rsync -av --list-only rsync://<TARGET_IP>/<SHARE>
```

Pull:

```bash
rsync -av rsync://<TARGET_IP>/<SHARE>
```

Over SSH:

```bash
rsync -av -e ssh rsync://<TARGET_IP>/<SHARE>
rsync -av -e "ssh -p2222" rsync://<TARGET_IP>/<SHARE>
```

Look for `.ssh`, `secrets.yaml`, build scripts.

### R-Services

Ports 512 (`rexec`), 513 (`rlogin`), 514 (`rsh`, `rcp`).

Trust files: `/etc/hosts.equiv` (global), `~/.rhosts` (per-user). Syntax `<username> <hostname/IP>`. `+` is wildcard.

```bash
cat /etc/hosts.equiv
```

Example granting passwordless access:

```text
pwnbox cry0l1t3
```

`.rhosts` dangerous example:

```text
htb-student     10.0.17.5
+               10.0.17.10
```

Scan:

```bash
sudo nmap -sV -p 512,513,514 <TARGET_IP>
```

Login:

```bash
rlogin <TARGET_IP> -l <USERNAME>
```

List logged-in users:

```bash
rwho
rusers -al <TARGET_IP>
```

## Windows Remote Management Protocols

### RDP

TCP 3389. TLS since Vista; self-signed certs default.

Footprinting:

```bash
nmap -sV -sC <TARGET_IP> -p3389 --script rdp*
```

`rdp-enum-encryption` shows NLA, security layers. `rdp-ntlm-info` leaks target name, NetBIOS domain, DNS names, product version.

Packet-level:

```bash
nmap -sV -sC <TARGET_IP> -p3389 --packet-trace --disable-arp-ping -n
```

Nmap probe sends `Cookie: mstshash=nmap` — visible to EDR.

`rdp-sec-check.pl`:

```bash
./rdp-sec-check.pl <TARGET_IP>
```

Output: PROTOCOL_SSL, PROTOCOL_HYBRID (NLA), PROTOCOL_RDP support.

Connect:

```bash
xfreerdp /u:<USERNAME> /p:<PASSWORD> /v:<TARGET_IP>
```

Alternatives: `rdesktop`, Remmina.

### WinRM

TCP 5985 (HTTP), 5986 (HTTPS). SOAP-based. Default-on since Windows Server 2012.

Footprinting:

```bash
nmap -sV -sC <TARGET_IP> -p5985,5986 --disable-arp-ping -n
```

`Microsoft HTTPAPI httpd 2.0` + `http-title: Not Found` = WinRM.

Connect:

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>
```

### WMI

TCP 135 start, then random port.

Impacket `wmiexec.py`:

```bash
wmiexec.py <USERNAME>:"<PASSWORD>"@<TARGET_IP> "hostname"
```