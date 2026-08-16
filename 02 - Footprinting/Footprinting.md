# Footprinting

## Enumeration Principles

Enumeration = information gathering via active (scans) and passive (third-party providers) methods, run as a loop — what we find determines the next query. OSINT is a separate procedure (purely passive, no active enumeration).

The goal is not to get at the systems but to find all the ways to get there. Don't do the classic mistake: find SSH/RDP/WinRM → immediately brute-force weak creds. That's noisy, gets you blacklisted, and you don't know the defensive measures yet.

Questions to work through against everything found:

```text
What can we see? Why can we see it? What image does it create? What do we gain, how can we use it?
What can we NOT see? Why not? What image results from what we do not see?
```

1. There is more than meets the eye — consider all points of view.
2. Distinguish between what we see and what we do not see.
3. There are always ways to gain more information — understand the target.

## Enumeration Methodology

Six layers, like walls between us and the target — find the gaps instead of smashing through headfirst. The process splits into three levels: infrastructure-based, host-based, and OS-based enumeration. Layers 1–2 don't really apply to intranet/AD (covered in other modules).

1. **Internet Presence** — domains, subdomains, vHosts, ASN, netblocks, IPs, cloud instances, security measures. Goal: identify all target systems and interfaces that can be tested. Even more important when the contract allows looking for additional hosts.
2. **Gateway** — firewalls, DMZ, IPS/IDS, EDR, proxies, NAC, network segmentation, VPN, Cloudflare. Goal: understand what we're dealing with and what to watch out for.
3. **Accessible Services** — service type, functionality, configuration, port, version, interface. Every service was installed for a reason; understand it to communicate with and exploit it. Main layer of this module.
4. **Processes** — PID, processed data, tasks, source, destination. Every command/function execution has at least one source and one target; identify the dependencies.
5. **Privileges** — groups, users, permissions, restrictions, environment. Privileges admins overlook (common in AD and multi-area admin environments).
6. **OS Setup** — OS type, patch level, network config, OS environment, configuration files, sensitive private files. Reflects the admins' skills; what internal info can be gleaned.

A methodology is a summary of systematic procedures, not a step-by-step guide. Collections of tools/commands are a cheat sheet, not the methodology.

Not every gap found leads inside, and time is always limited — but there is nearly always a way in. Even a four-week test can't guarantee no vulnerabilities remain.

## Domain Information

Passive gathering — stay hidden, navigate as a "customer"/"visitor", no direct active scans. Start with the company's main website: read the texts and think about what technologies and structures each offered service requires (app dev, IoT, hosting, data science, IT security...). Take the developer's view — services are bound to technical aspects needed to provide them. Deep OSINT is covered in OSINT: Corporate Recon.

### Online Presence

SSL certificate of the main website is a first source — a cert often covers several (sub)domains, most likely still active.

crt.sh queries Certificate Transparency logs (RFC 6962 — every CA-issued cert gets logged; Let's Encrypt entries end up there):

```bash
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq .

# unique subdomains only
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

Next, filter out hosts hosted by third-party providers (not allowed to test those without permission) and keep the company-hosted ones:

```bash
for i in $(cat subdomainlist);do host $i | grep "has address" | grep <TARGET_DOMAIN> | cut -d" " -f1,4;done
```

Build an IP list (same loop, `cut -d" " -f4 >> ip-addresses.txt`) and run them through Shodan — it indexes internet-connected systems and their open ports (HTTP/HTTPS, FTP, SSH, SNMP, Telnet, RTSP, SIP), org, SSL versions, DH params:

```bash
for i in $(cat ip-addresses.txt);do shodan host $i;done
```

Note interesting IPs for later active investigation.

### DNS Records

```bash
dig any <TARGET_DOMAIN>
```

- A → IPs behind (sub)domains
- MX → mail servers (hosted at Google in the example → note and skip)
- NS → name servers, usually identifies the hosting provider
- TXT → verification keys for third-party providers + SPF/DMARC/DKIM

TXT records are where the hidden third parties show up. Example findings and what they mean:

- Atlassian → dev/collaboration platform; try it free to learn the structure
- Google Gmail → email managed by Google → maybe open GDrive folders/files via link
- LogMeIn → centralized remote access management; admin access here (e.g. via password reuse) = access to all systems
- Mailgun → email APIs, SMTP relays, webhooks → look for API interfaces to test (IDOR, SSRF, POST/PUT requests)
- Outlook/O365 → OneDrive, Azure blob and file storage; Azure file storage uses SMB
- INWX → hosting/registrar; the `MS=` TXT value is usually similar to the username/ID for the management platform login
- SPF `ip4:` entries → additional internal IPs

## Cloud Resources

S3 buckets (AWS), blobs (Azure), cloud storage (GCP) can be accessible without authentication when misconfigured. Cloud storage often gets a DNS entry so employees can reach it easily — an `s3-website-us-west-2.amazonaws.com` entry in the host resolution is a strong hint.

Ways to find storage:

- Google dorks: `intext:<COMPANY> inurl:amazonaws.com` and `intext:<COMPANY> inurl:blob.core.windows.net` → PDFs, docs, presentations, code
- Page source code — images/JS/CSS are often loaded from blob storage (DNS prefetch/preconnect links)
- domain.glass — infrastructure overview; also shows Cloudflare's security assessment status ("Safe" = a layer-2 security measure found)
- GrayHatWarfare — discover AWS/Azure/GCP storage, sort/filter by file format; also search company-name abbreviations, and search files simultaneously

Leaked files can include `id_rsa` / `id_rsa.pub` — download the private key, log into machines without a password.

## Staff

Employees on LinkedIn/Xing reveal the teams' makeup, technologies, programming languages, software in use.

Job postings are gold — one example listed: Java/C#/C++, Python/Ruby/PHP/Perl, PostgreSQL/MySQL/SQL Server/Oracle, SQLAlchemy/Hibernate/Entity Framework, Flask/Django/Spring/ASP.NET MVC, pytest/JUnit, SOA/microservices & REST, CI, Git/SVN/Mercurial/Perforce; desired: Atlassian suite (Confluence, Jira, Bitbucket), Docker/Kubernetes, Redis, NumPy. Atlassian Suite in use → potentially accessible resources.

Employee profiles: skills show what they work with (React, Svelte, AngularJS, Django...). For a tech like Django, search for its security misconfigurations — e.g. the OWASP Top10 for Django GitHub repo. Best practices tell us what to look for because many follow them blindly and even name files as shown in the instructions.

Public GitHub projects of employees can leak a personal email address in a file, or a hardcoded JWT token in the web app.

LinkedIn search can filter by connections, location, company, school, industry, titles. For infrastructure/tech identification, look for employees working in both development and security — they hint at the security measures in place.

## FTP

Clear-text protocol, credentials usually required, but anonymous FTP is a common configuration. TFTP: UDP-based, no user authentication, access purely by file read/write permissions, no directory listing — local/protected networks only.

### Default Configuration

vsFTPd is the most used FTP server on Linux; config in `/etc/vsftpd.conf` (defaults include `anonymous_enable=NO`, `local_enable=YES`). Also check `/etc/ftpusers` — users listed there are denied FTP login even if they exist on the system.

### Dangerous Settings

- `anonymous_enable=YES` / `anon_upload_enable=YES` / `anon_mkdir_write_enable=YES` / `no_anon_password=YES`
- `anon_root=/home/<user>/ftp` — directory for anonymous
- `write_enable=YES` — enables STOR, DELE, RNFR, RNTO, MKD, RMD, APPE, SITE
- `chown_uploads=YES` + `chown_username=<user>` — ownership of anon uploads
- `chroot_local_user=YES` / `chroot_list_enable=YES` — local users locked to home dir
- `hide_ids=YES` — all UIDs/GIDs shown as `ftp`; hides local usernames (they'd otherwise be brute-force targets, though fail2ban usually blocks that)
- `ls_recurse_enable=YES` — allows `ls -R`, full recursive listing at once

Anonymous login — response code 220 comes with the banner (service description, version, system type):

```bash
ftp <TARGET_IP>          # login as anonymous
```

Inside the session: `status` for an overview, `debug` and `trace` make the server show more (raw commands like `---> PORT`, `---> LIST`).

Download a single file with `get <file>`, upload with `put <file>`. Everything at once (noisy — nobody legitimately downloads all files, may trip alarms):

```bash
wget -m --no-passive ftp://anonymous:anonymous@<TARGET_IP>
```

FTP shares synced with a web server are common (devs want quick file access; internal systems are often poorly hardened because "they can't be reached from outside"). Upload access there → LFI to execute system commands, or direct webshell/reverse shell. FTP logs can also lead to RCE.

### Footprinting the Service

```bash
sudo nmap --script-updatedb          # update NSE DB first
sudo nmap -sV -p21 -sC -A <TARGET_IP>
```

`ftp-anon` checks anonymous access and renders the root directory (flags writeable dirs), `ftp-syst` runs STAT (config + version). Useful FTP NSE scripts: ftp-anon, ftp-syst, ftp-bounce, ftp-brute, ftp-vsftpd-backdoor, ftp-proftpd-backdoor, ftp-vuln-cve2010-4221, ftp-libopie. Locate them with `find / -type f -name ftp* 2>/dev/null | grep scripts`.

`--script-trace` shows the NSE network conversation (commands sent, ports, responses). Manual interaction also works:

```bash
nc -nv <TARGET_IP> 21
telnet <TARGET_IP> 21

# FTP with TLS/SSL — also reveals the certificate (hostname, email, locations)
openssl s_client -connect <TARGET_IP>:21 -starttls ftp
```

## SMB

Samba provides SMB on Linux/Unix (SMB/CIFS). Older NetBIOS-based communication runs over TCP 137–139, CIFS direct over TCP 445.

### Default Configuration

```bash
cat /etc/samba/smb.conf | grep -v "#\|\;"
```

`[global]` section plus one section per share; share settings override global ones (and are often configured wrong). Relevant options: `workgroup`, `server string`, `path`, `unix password sync`, `usershare allow guests`, `map to guest = bad user`, `browseable`, `guest ok`, `read only`, `create mask`. `print$` and `IPC$` exist by default.

### Dangerous Settings

- `browseable = yes` — share listing visible (to employees AND to us after access)
- `read only = no` / `writable = yes` — create/modify files
- `guest ok = yes` — connect without password
- `enable privileges = yes` — honor privileges assigned to specific SID
- `create mask = 0777` / `directory mask = 0777`
- `logon script = script.sh` — executed on user login
- `magic script = script.sh` / `magic output = script.out` — executed when closed, output stored

Example share using all of them (the kind left over from testing on an internal subnet):

```text
[notes]
    comment = CheckIT
    path = /mnt/notes/
    browseable = yes
    read only = no
    writable = yes
    guest ok = yes
    enable privileges = yes
    create mask = 0777
    directory mask = 0777
```

After editing smb.conf: `sudo systemctl restart smbd`.

Enumerate shares via null session (`-N` = no password), connect, and use `help` inside for available commands:

```bash
smbclient -N -L //<TARGET_IP>
smbclient //<TARGET_IP>/notes
```

`get <file>` downloads; `!<cmd>` runs a local system command without dropping the connection (`!ls`, `!cat prep-prod.txt`). On the server side, `smbstatus` shows who is connected from which host and to which share (as root).

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -sC -p139,445
```

Nmap gives little (version, NetBIOS name, "message signing enabled but not required") — go manual with rpcclient (MS-RPC functions), null session:

```bash
rpcclient -U "" <TARGET_IP>
```

Key queries:

```text
srvinfo                   server info
enumdomains              all deployed domains
querydominfo              domain/server/user counts, server role
netshareenumall           all shares incl. paths
netsharegetinfo <share>   share details + DACL/ACE/SID
enumdomusers              domain users with RIDs
queryuser <RID>           user details (home drive, profile path, password times, bad pwd count)
querygroup <RID>          group info
```

If `enumdomusers` is blocked, `queryuser <RID>` usually still works → brute-force RIDs:

```bash
for i in $(seq 500 1100);do rpcclient -N -U "" <TARGET_IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
```

Alternatives:

```bash
samrdump.py <TARGET_IP>                                  # impacket, users + details
smbmap -H <TARGET_IP>                                    # share permissions per share
crackmapexec smb <TARGET_IP> --shares -u '' -p ''        # shows READ/WRITE + signing status
```

enum4linux-ng automates most of this:

```bash
git clone https://github.com/cddmp/enum4linux-ng.git && cd enum4linux-ng
pip3 install -r requirements.txt
enum4linux-ng.py <TARGET_IP> -A
```

Highlights from its output: null session allowed (and random-username session — rerunning with that user may give more), supported SMB dialects, signing required or not, domain/workgroup, users, shares with mapping/listing status, and the password policy (`min_pw_length: 5`, `DOMAIN_PASSWORD_COMPLEX: false`, `lockout_threshold: None` in the example — useful before any brute-force).

Use more than one tool — they're programmed differently and return different information; verify manually, don't blindly trust automation.

## NFS

Linux/Unix file sharing, no authentication mechanism of its own — AUTHENTICATION sits in the RPC options, authorization maps UID/GID. Client and server don't need matching UID/GID mappings, so it belongs in trusted networks only.

### Default Configuration

`/etc/exports` lists shared filesystems with allowed hosts/subnets and options. Relevant options: `rw`, `ro`, `sync`, `async`, `secure` (only ports <1024), `insecure`, `no_subtree_check`, `root_squash` (maps root UID/GID 0 to anonymous — prevents root from accessing the mount).

### Dangerous Settings

- `rw` — read/write
- `insecure` — ports above 1024 usable
- `nohide` — export filesystems mounted under an exported directory
- `no_root_squash` — files created by root keep UID/GID 0

### Footprinting the Service

Essential ports: TCP 111 and 2049.

```bash
sudo nmap <TARGET_IP> -p111,2049 -sV -sC            # rpcinfo lists nfs, mountd, nlockmgr ports
sudo nmap --script nfs* <TARGET_IP> -sV -p111,2049  # nfs-ls (contents!), nfs-showmount, nfs-statfs
```

Mount and inspect:

```bash
showmount -e <TARGET_IP>
mkdir target-NFS
sudo mount -t nfs <TARGET_IP>:/ ./target-NFS/ -o nolock
```

`ls -l` shows usernames/groups, `ls -n` the raw UIDs/GIDs. Recreate a user with matching UID/GID locally → view and modify their files. If `root_squash` is set, root-owned files can't be edited even as root. Escalation path: with SSH access to the system, upload a shell with the SUID bit of a target user to the NFS share and run it via the SSH user.

```bash
sudo umount ./target-NFS
```

## DNS

Records worth remembering: A/AAAA (→ IP), MX (mail servers), NS (nameservers), TXT (verification keys, SPF/DMARC/DKIM), CNAME (alias), PTR (reverse), SOA (zone + admin contact).

The SOA rname converts to an email by replacing the first `.` with `@`:

```bash
dig soa <TARGET_DOMAIN>
# ... SOA ns-161.awsdns-20.com. awsdns-hostmaster.amazon.com. ...  →  awsdns-hostmaster@amazon.com
```

### Default Configuration

Three config file types: local config, zone files, reverse resolution files. Bind9 local configs: `named.conf.local`, `named.conf.options`, `named.conf.log`. Zones are defined there:

```text
zone "domain.com" {
    type master;
    file "/etc/bind/db.domain.com";
    allow-update { key rndc-key; };
};
```

Zone file (`/etc/bind/db.domain.com`) holds the forward records in BIND format — exactly one SOA and at least one NS required; a syntax error makes the whole zone unusable (server answers SERVFAIL). Reverse zone (`/etc/bind/db.10.129.14`, `$ORIGIN 14.129.10.in-addr.arpa`) maps last IP octet → PTR hostname.

### Dangerous Settings

- `allow-query` — hosts allowed to send requests
- `allow-recursion` — hosts allowed to send recursive requests
- `allow-transfer` — hosts allowed to receive zone transfers
- `zone-statistics` — collects zone stats

### Footprinting the Service

```bash
dig ns <TARGET_DOMAIN> @<TARGET_IP>          # other known nameservers — query them too
dig CH TXT version.bind <TARGET_IP>          # BIND version, if the entry exists
dig any <TARGET_DOMAIN> @<TARGET_IP>         # everything the server is willing to disclose
```

Zone transfer (AXFR, over TCP 53). If `allow-transfer` was set to a subnet, `any`, or a leftover workaround, we can pull the entire zone file — including other zones with internal IPs and hostnames:

```bash
dig axfr <TARGET_DOMAIN> @<TARGET_IP>
dig axfr internal.<TARGET_DOMAIN> @<TARGET_IP>   # example internal zone exposed dc1, dc2, mail1, vpn, ws1, ws2, susws
```

Subdomain brute-forcing with a SecLists hostname list:

```bash
for sub in $(cat subdomains-top1million-110000.txt);do dig $sub.<TARGET_DOMAIN> @<TARGET_IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt;done

dnsenum --dnsserver <TARGET_IP> --enum -p 0 -s 0 -o subdomains.txt -f subdomains-top1million-110000.txt <TARGET_DOMAIN>
```

## SMTP

Postfix config in `/etc/postfix/main.cf`. Relevant commands: `HELO`/`EHLO` (start session), `AUTH PLAIN`, `MAIL FROM`, `RCPT TO`, `DATA`, `RSET`, `VRFY`, `EXPN`, `NOOP`, `QUIT`.

```bash
telnet <TARGET_IP> 25
```

`VRFY` checks whether a mailbox exists — but depending on configuration the server may answer `252 2.0.0 <anything>` even for garbage usernames, so never fully trust it (or the automated tools that use it). Through a web proxy: `CONNECT <TARGET_IP>:25 HTTP/1.0`.

Sending a mail manually (spoofed sender works — senders are not reliably authenticated):

```text
EHLO <TARGET_DOMAIN>
MAIL FROM: <user@<TARGET_DOMAIN>
RCPT TO: <user2@<TARGET_DOMAIN> NOTIFY=success,failure
DATA
From: <user@<TARGET_DOMAIN>
To: <user2@<TARGET_DOMAIN>
Subject: DB

body text
.
QUIT
```

### Dangerous Settings

Open relay — admins often don't know which IP ranges to allow and just allow all:

```text
mynetworks = 0.0.0.0/0
```

(Default example: `mynetworks = 127.0.0.0/8 <internal subnet>`.) An open relay lets us send fake emails / initialize communication between parties, or spoof and read mail.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sC -sV -p25                    # smtp-commands: EHLO capability list (incl. VRFY)
sudo nmap <TARGET_IP> -p25 --script smtp-open-relay   # 16 relay tests
```

## IMAP / POP3

IMAP manages mail online on the server with folder structures and multi-client sync; POP3 only lists/retrieves/deletes. Ports: 143/110 plaintext, 993/995 TLS/SSL.

### Default Configuration

Dovecot (`dovecot-imapd`, `dovecot-pop3d`). Core IMAP commands after connecting: `1 LOGIN <user> <pass>`, `1 LIST "" *`, `1 SELECT INBOX`, `1 FETCH <ID> all`, `1 LOGOUT`. POP3: `USER`, `PASS`, `STAT`, `LIST`, `RETR <id>`, `DELE <id>`, `CAPA`, `QUIT`.

### Dangerous Settings

- `auth_debug` — logs all authentication debug output
- `auth_debug_passwords` — submitted passwords + scheme get logged
- `auth_verbose` — logs failed authentication attempts and reasons
- `auth_verbose_passwords` — passwords logged (possibly truncated)
- `auth_anonymous_username` — username for the ANONYMOUS SASL mechanism

Misconfigurations here can mean reading all sent/received mail — often confidential.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
```

Cert subject gives hostname (e.g. mail1.<domain>), organization, location; capabilities list the available commands per port.

With credentials (e.g. username reused as password — check reuse):

```bash
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD>       # lists mailboxes
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD> -v    # adds TLS version, cert, banner (server version)

openssl s_client -connect <TARGET_IP>:pop3s
openssl s_client -connect <TARGET_IP>:imaps
```

## SNMP

UDP 161 (traps on 162). v1/v2c have no real security — the community string (acts as a password) travels in plaintext; many orgs still run v2c because the v3 migration is complex. v3 adds user/password auth and encryption. MIB = text file describing queryable objects; OID = numbered node in the tree hierarchy (longer chain = more specific).

### Default Configuration

```bash
cat /etc/snmp/snmpd.conf | grep -v "#" | sed -r '/^\s*$/d'
```

Defaults restrict the view: `view systemonly included .1.3.6.1.2.1.1` and `.1.3.6.1.2.1.25.1`, `rocommunity public`.

### Dangerous Settings

- `rwuser noauth` — full OID tree access without authentication
- `rwcommunity <community string> <IPv4 address>` — full OID tree access regardless of request source
- `rwcommunity6` — same for IPv6

### Footprinting the Service

```bash
snmpwalk -v2c -c public <TARGET_IP>
```

Leaks (with the right string + no-auth version): OS version/kernel, syscontact email, hostname, location, boot image, installed packages. If the community string is unknown:

```bash
onesixtyone -c snmp.txt <TARGET_IP>      # SecLists snmp.txt
```

Community strings are sometimes bound to specific IPs and named after the hostname, possibly with symbols — in big networks there's a pattern, guessable with custom wordlists (crunch).

Once a string is known, brute-force OIDs:

```bash
braa public@<TARGET_IP>:.1.3.6.*
```

## MySQL

Config in `/etc/mysql/mysql.conf.d/mysqld.cnf`, port 3306. Databases are often exported as single `.sql` files (e.g. wordpress.sql).

### Dangerous Settings

- `user` / `password` / `admin_address` — plaintext entries in the config; wrong file permissions + any file-read or shell → full database access
- `debug`, `sql_warnings` — verbose error output, often surfaced in web apps; confirms the app talks to the DB differently than intended, aids SQLi manipulation
- `secure_file_priv` — limits data import/export operations

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -sC -p3306 --script mysql*
```

`mysql-info` gives the version reliably. `mysql-empty-password`/`mysql-brute`/`mysql-enum` results can be false positives — always verify manually:

```bash
mysql -u root -h <TARGET_IP>                 # no password
mysql -u root -p<PASSWORD> -h <TARGET_IP>    # no space between -p and password
```

Inside:

```sql
select version();
show databases;
use mysql; show tables;
use sys; show tables; select host, unique_users from host_summary;   # which hosts connect
show columns from <table>;
select * from <table>;
select * from <table> where <column> = "<string>";
```

`sys` and `information_schema` are the interesting system databases (metadata).

## MSSQL

Closed source, native .NET support, mostly on Windows. Default system databases: `master` (all system info), `model` (template for new DBs), `msdb` (SQL Server Agent jobs/alerts), `tempdb` (temporary objects), `resource` (read-only system objects).

### MSSQL Clients

SSMS, mssql-cli, SQL Server PowerShell, HeidiSQL, SQLPro, Impacket's `mssqlclient.py` (usually already on pentesting distros). SSMS is client-side — a compromised box with SSMS may hold saved credentials.

### Default Configuration

Runs as `NT SERVICE\MSSQLSERVER`; Windows Authentication by default (login processed by the local SAM or the DC/AD). Compromised account → privesc and lateral movement across the domain.

### Dangerous Settings

- Clients not using encryption; self-signed certificates (spoofable) when encryption is on
- Named pipes in use
- Weak & default `sa` credentials — often forgotten to disable

### Footprinting the Service

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 <TARGET_IP>
```

Gives hostname, instance name, version, named pipe (`\\<TARGET_IP>\pipe\sql\query`), NTLM info (NetBIOS/DNS names, product version). Metasploit alternative:

```text
use auxiliary/scanner/mssql/mssql_ping
set rhosts <TARGET_IP>
run
```

With credentials, connect and enumerate via T-SQL:

```bash
mssqlclient.py <USERNAME>@<TARGET_IP> -windows-auth
```

```sql
select name from sys.databases
```

## Oracle TNS

TNS listener on TCP 1521 (default; remotely manageable in 8i/9i, not in 10g/11g). Configs `tnsnames.ora` (client-side, service names → addresses) and `listener.ora` (server-side) in `$ORACLE_HOME/network/admin`.

Default passwords to remember: Oracle 9 `sys` = `CHANGE_ON_INSTALL`, Oracle 10 has none set, DBSNMP service = `dbsnmp`.

### Footprinting the Service

```bash
sudo nmap -p1521 -sV <TARGET_IP> --open
sudo nmap -p1521 -sV <TARGET_IP> --open --script oracle-sid-brute    # SID guessing, e.g. XE
```

The SID identifies the database instance — wrong SID, no connection. ODAT (Oracle Database Attacking Tool) enumerates and exploits: versions, users, vulnerabilities, misconfigurations. Setup on Pwnbox roughly: `apt-get install libaio1`, build/install cx_Oracle, clone https://github.com/quentinhardy/odat.git plus its pip deps. Run everything:

```bash
odat.py all -s <TARGET_IP>       # tries all modules, flags locked accounts, finds valid creds
```

With found credentials (example: scott/tiger), connect via sqlplus (`apt install oracle-instantclient-sqlplus`; if `libsqlplus.so` missing: `echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf && ldconfig`):

```bash
sqlplus scott/tiger@<TARGET_IP>/XE
sqlplus scott/tiger@<TARGET_IP>/XE as sysdba    # try logging in as sysdba if privileges allow
```

```sql
select table_name from all_tables;
select * from user_role_privs;      -- current user's roles
select name, password from sys.user$;   -- password hashes → crack offline
```

Web shell upload path (needs a web server; test first with a harmless file since AV/IPS):

```bash
echo "Oracle File Upload Test" > testing.txt
odat.py utlfile -s <TARGET_IP> -d XE -U scott -P tiger --sysdb --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
curl -X GET http://<TARGET_IP>/testing.txt
```

Default web roots: Linux `/var/www/html`, Windows `C:\inetpub\wwwroot`.

## IPMI

UDP 623. Hardware-level host management, independent of the OS — works before boot, when powered down, after system failure. BMCs (HP iLO, Dell DRAC, Supermicro IPMI) usually expose a web console, Telnet/SSH, and port 623. BMC access ≈ physical access: monitor, reboot, power off, reinstall the OS. Common on internal tests — always check.

### Footprinting the Service

```bash
sudo nmap -sU --script ipmi-version -p 623 <TARGET_HOST>
```

Metasploit equivalent: `auxiliary/scanner/ipmi/ipmi_version`.

Default credentials worth trying (often lead to quick wins):

- Dell iDRAC — root / calvin
- HP iLO — Administrator / randomized 8-char string of numbers + uppercase letters
- Supermicro IPMI — ADMIN / ADMIN

### Dangerous Settings

RAKP flaw in IPMI 2.0: during authentication the server sends a salted SHA1/MD5 hash of the user's password to the client BEFORE auth — password hashes for ANY valid account can be retrieved and cracked offline. No direct fix (it's part of the spec); mitigations are long passwords or network segmentation.

```text
use auxiliary/scanner/ipmi/ipmi_dumphashes
set rhosts <TARGET_IP>
run
```

Module options include CRACK_COMMON (auto-cracks common passwords) and hashcat/john output files. Offline cracking:

```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u    # HP iLO factory-default style: 8 chars, digits+uppercase
```

Cracked BMC passwords are frequently reused elsewhere — one engagement cracked an IPMI hash and used it to SSH as root into many critical servers plus web consoles of monitoring tools.

## Linux Remote Management Protocols

Remote management services are high-value: misconfiguration can mean direct access, and they're a prime target for weak/default credentials and password reuse.

### SSH

Dangerous `sshd_config` settings: `PasswordAuthentication yes`, `PermitEmptyPasswords yes`, `PermitRootLogin yes`, `Protocol 1` (outdated), `X11Forwarding yes` (had a command injection vuln in OpenSSH 7.2p1), `AllowTcpForwarding yes`, `PermitTunnel`, `DebianBanner yes`.

Banner reading: `SSH-1.99-OpenSSH_3.9p1` accepts both protocol versions 1 and 2; `SSH-2.0-OpenSSH_8.2p1` only v2. Old versions have known vulns (e.g. CVE-2020-14145 MITM on the initial connection).

Audit server-side crypto config:

```bash
ssh-audit.py <TARGET_IP>      # from https://github.com/jtesta/ssh-audit
```

Shows banner/version and flags weak KEX/host-key algorithms (weak curves, weak hashing, weak RNG).

Discover allowed auth methods, and force one for brute-force:

```bash
ssh -v <USERNAME>@<TARGET_IP>
ssh -v <USERNAME>@<TARGET_IP> -o PreferredAuthentications=password
```

### Rsync

Port 873, often unauthenticated. Worth checking any found credentials against — sensitive files pulled this way can give remote access.

```bash
sudo nmap -sV -p 873 <TARGET_IP>

nc -nv <TARGET_IP> 873
@RSYNCD: 31.0
#list            # type this — lists shares
```

```bash
rsync -av --list-only rsync://<TARGET_IP>/dev    # enumerate a share (watch for .ssh, secrets.yaml...)
rsync -av rsync://<TARGET_IP>/dev                # sync everything down
rsync -av -e ssh rsync://<TARGET_IP>/dev         # if rsync runs over SSH; -e "ssh -p2222" for non-standard port
```

### R-Services

Legacy, unencrypted, occasionally still found internally. Ports: rexec 512/TCP, rlogin 513/TCP, rsh + rcp 514/TCP. rsh opens a shell with no login procedure; rcp copies bidirectionally with no overwrite warning.

```bash
sudo nmap -sV -p 512,513,514 <TARGET_IP>
```

Trust lives in `/etc/hosts.equiv` (global) and `~/.rhosts` (per-user), syntax `<username> <ip/hostname>`, `+` as wildcard:

```text
htb-student     10.0.17.5
+               10.0.17.10    # any user from this host
+               +             # anyone from anywhere
```

Misconfigured trust → login without credentials:

```bash
rlogin <TARGET_IP> -l <USERNAME>
rwho                # interactive sessions on the local network (user + host)
rusers -al <TARGET_IP>   # detailed: user, host, TTY, login time, idle, remote host
```

### Final Thoughts

Probe remote management services for everything they'll give — especially once credentials have been compiled elsewhere in the network; weak/default credentials and password reuse are the recurring wins.

## Windows Remote Management Protocols

RDP, WinRM, WMI are the main components.

### RDP

TCP 3389 (UDP possible); TLS since Vista but certs are self-signed by default. NLA is the default requirement.

```bash
nmap -sV -sC <TARGET_IP> -p3389 --script rdp*
```

`rdp-enum-encryption` shows the security layer (CredSSP/NLA), `rdp-ntlm-info` leaks Target_Name, NetBIOS/DNS computer names, product version.

Caveat: `--packet-trace` reveals the RDP cookie `mstshash=nmap` — identifiable by threat hunters/EDR, can lock us out on hardened networks:

```bash
nmap -sV -sC <TARGET_IP> -p3389 --packet-trace --disable-arp-ping -n
```

Unauthenticated security-settings check (Cisco CX Security tool; needs `cpan > install Encoding::BER`):

```bash
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git && cd rdp-sec-check
rdp-sec-check.pl <TARGET_IP>
```

Connect with credentials (also rdesktop, Remmina):

```bash
xfreerdp /u:<USERNAME> /p:"<PASSWORD>" /v:<TARGET_IP>
```

### WinRM

SOAP-based, TCP 5985 (HTTP) / 5986 (HTTPS; often only 5985 is used). WinRS executes arbitrary remote commands; PowerShell remote sessions require it.

```bash
nmap -sV -sC <TARGET_IP> -p5985,5986 --disable-arp-ping -n
```

Reachability check from PowerShell: `Test-WsMan <host>`. From Linux:

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>
```

### WMI

Read/write access to nearly all Windows settings; accessed via PowerShell, VBScript, or WMIC. Communication starts on TCP 135, then moves to a random port.

```bash
wmiexec.py <USERNAME>:"<PASSWORD>"@<TARGET_IP> "hostname"
```