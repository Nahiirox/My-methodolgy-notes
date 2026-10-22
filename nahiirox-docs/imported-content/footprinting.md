---
icon: binoculars
layout:
  width: default
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
  actions:
    visible: true
  anchors:
    visible: true
---

# Footprinting

## Enumeration Principles

Enumeration combines active checks with passive sources. Treat it as a loop: each finding drives the next query. OSINT is separate and remains passive.

Map the attack surface before attempting access. Do not start password attacks after spotting SSH, RDP, or WinRM. That is noisy and reveals little about the target's controls.

For every finding, ask:

```
What can we see? Why can we see it? What image does it create? What do we gain, how can we use it?
What can we NOT see? Why not? What image results from what we do not see?
```

1. Consider other explanations and vantage points.
2. Record both exposed and missing information.
3. Identify the next source or check that can reduce uncertainty.

## Enumeration Methodology

Six layers separate the tester from the target. Look for gaps between them. The process covers infrastructure, hosts, and operating systems. Layers 1–2 rarely apply to intranet or AD testing.

1. **Internet Presence** — domains, subdomains, vHosts, ASN, netblocks, IPs, cloud instances, and security measures. Identify all target systems and testable interfaces. This matters most when the contract permits additional-host discovery.
2. **Gateway** — firewalls, DMZ, IPS/IDS, EDR, proxies, NAC, network segmentation, VPN, and Cloudflare. Identify controls and testing constraints.
3. **Accessible Services** — service type, functionality, configuration, port, version, and interface. Understand each service before interacting with it. This is the module's main layer.
4. **Processes** — PID, processed data, tasks, source, and destination. Every command or function execution has at least one source and one target; identify the dependencies.
5. **Privileges** — groups, users, permissions, restrictions, and environment. Privileges are often overlooked by admins, especially in AD and multi-area admin environments.
6. **OS Setup** — OS type, patch level, network configuration, OS environment, configuration files, and sensitive private files. This reflects the admins' skills and what internal information can be gleaned.

A methodology defines the order and purpose of checks. A command list is only a reference.

Not every finding is exploitable. Time is limited, and a test cannot prove the absence of vulnerabilities.

## Domain Information

Passive gathering uses public sources without sending probes to target infrastructure. Start with the main site. Link each advertised service to the technologies and supporting infrastructure it likely requires.

Think from the delivery team's perspective. Public services require specific platforms, vendors, and integrations. Deep OSINT belongs in Corporate Recon.

### Online Presence

Start with the main site's TLS certificate. Subject alternative names often expose active subdomains.

crt.sh queries Certificate Transparency logs. Under RFC 6962, every CA-issued certificate gets logged; Let's Encrypt entries end up there too.

```bash
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq .

# unique subdomains only
curl -s https://crt.sh/\?q\=<TARGET_DOMAIN>\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

Remove third-party hosts that are out of scope. Retain company-managed hosts:

```bash
for i in $(cat subdomainlist);do host $i | grep "has address" | grep <TARGET_DOMAIN> | cut -d" " -f1,4;done
```

Build an IP list with `cut -d" " -f4 >> ip-addresses.txt`, then check it in Shodan. Shodan can expose indexed ports, banners, organization data, TLS versions, and DH parameters.

```bash
for i in $(cat ip-addresses.txt);do shodan host $i;done
```

Record IPs for later active investigation.

### DNS Records

```bash
dig any <TARGET_DOMAIN>
```

* **A** → IPs behind subdomains
* **MX** → mail servers; if hosted at Google in the example, note and skip
* **NS** → name servers, usually identifying the hosting provider
* **TXT** → verification keys for third-party providers plus SPF/DMARC/DKIM

TXT records can reveal third-party providers.

| Finding            | Meaning                                                                                                                |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| Atlassian          | Development and collaboration platform; try it free to learn the structure.                                            |
| Google Gmail       | Email managed by Google; there may be open GDrive folders or files available via link.                                 |
| LogMeIn            | Centralized remote access management; admin access here, such as via password reuse, can mean access to all systems.   |
| Mailgun            | Email APIs, SMTP relays, and webhooks; look for API interfaces to test, including IDOR, SSRF, and POST/PUT requests.   |
| Outlook/O365       | OneDrive, Azure blob, and file storage; Azure file storage uses SMB.                                                   |
| INWX               | Hosting and registrar; the `MS=` TXT value is usually similar to the username or ID for the management platform login. |
| SPF `ip4:` entries | Additional internal IPs.                                                                                               |

## Cloud Resources

Misconfigured AWS buckets, Azure blobs, and GCP storage may allow unauthenticated access. Organizations often assign DNS names for easy staff access. An `s3-website-us-west-2.amazonaws.com` resolution is a useful indicator.

Ways to find storage:

* Google dorks: `intext:<COMPANY> inurl:amazonaws.com` and `intext:<COMPANY> inurl:blob.core.windows.net` for PDFs, documents, presentations, and code.
* Page source code: images, JavaScript, and CSS are often loaded from blob storage through DNS prefetch or preconnect links.
* domain.glass: infrastructure overview; also shows Cloudflare's security assessment status. `Safe` indicates a layer-2 security measure was found.
* GrayHatWarfare: discover AWS/Azure/GCP storage, sort and filter by file format, search company-name abbreviations, and search files simultaneously.

Look for `id_rsa` and `id_rsa.pub`. A leaked private key can provide passwordless access.

## Staff

LinkedIn and Xing profiles reveal teams, technologies, languages, and products in use.

Job listings often provide the clearest technology inventory:

* Java/C#/C++
* Python/Ruby/PHP/Perl
* PostgreSQL/MySQL/SQL Server/Oracle
* SQLAlchemy/Hibernate/Entity Framework
* Flask/Django/Spring/ASP.NET MVC
* pytest/JUnit
* SOA/microservices and REST
* CI
* Git/SVN/Mercurial/Perforce
* Atlassian suite: Confluence, Jira, Bitbucket
* Docker/Kubernetes
* Redis
* NumPy

Atlassian deployments can expose additional discovery targets.

Profiles can expose application frameworks such as React, Svelte, AngularJS, or Django. For each framework, research common deployment mistakes and published hardening guidance. Teams often reuse documented filenames and layouts.

Public employee repositories may expose personal email addresses or hardcoded tokens.

LinkedIn search can filter by connections, location, company, school, industry, and titles. For infrastructure and technology identification, look for employees working in both development and security; they can hint at security measures in place.

## FTP

FTP is clear text. It normally requires credentials, but anonymous access is common. TFTP uses UDP, has no authentication or directory listing, and relies on file permissions. Use it only on protected networks.

### Default Configuration

vsFTPd is the most used FTP server on Linux. Its configuration is in `/etc/vsftpd.conf`; defaults include `anonymous_enable=NO` and `local_enable=YES`.

Also inspect `/etc/ftpusers`. Users listed there are denied FTP login even if they exist on the system.

### Dangerous Settings

* `anonymous_enable=YES`
* `anon_upload_enable=YES`
* `anon_mkdir_write_enable=YES`
* `no_anon_password=YES`
* `anon_root=/home/<user>/ftp` — directory for anonymous users
* `write_enable=YES` — enables `STOR`, `DELE`, `RNFR`, `RNTO`, `MKD`, `RMD`, `APPE`, and `SITE`
* `chown_uploads=YES` with `chown_username=<user>` — ownership of anonymous uploads
* `chroot_local_user=YES` or `chroot_list_enable=YES` — local users locked to their home directories
* `hide_ids=YES` — all UIDs/GIDs shown as `ftp`; hides local usernames
* `ls_recurse_enable=YES` — allows `ls -R` for full recursive listing

Anonymous login response code 220 includes the banner, which may reveal service description, version, and system type:

```bash
ftp <TARGET_IP>          # login as anonymous
```

Inside the session:

* `status` provides an overview.
* `debug` and `trace` make the server show more, including raw commands such as `---> PORT` and `---> LIST`.
* `get <file>` downloads a file.
* `put <file>` uploads a file.

Download everything at once:

```bash
wget -m --no-passive ftp://anonymous:anonymous@<TARGET_IP>
```

{% hint style="warning" %}
Downloading all files is noisy. Nobody legitimately downloads every file, and it may trigger alarms.
{% endhint %}

FTP shares may synchronize with web roots for developer access. Write access can enable file inclusion abuse, web shells, reverse shells, or log-based RCE.

### Footprinting the Service

```bash
sudo nmap --script-updatedb          # update NSE DB first
sudo nmap -sV -p21 -sC -A <TARGET_IP>
```

`ftp-anon` checks anonymous access and renders the root directory, flagging writable directories. `ftp-syst` runs `STAT` for configuration and version information.

Useful FTP NSE scripts include:

* `ftp-anon`
* `ftp-syst`
* `ftp-bounce`
* `ftp-brute`
* `ftp-vsftpd-backdoor`
* `ftp-proftpd-backdoor`
* `ftp-vuln-cve2010-4221`
* `ftp-libopie`

Locate scripts:

```bash
find / -type f -name ftp* 2>/dev/null | grep scripts
```

`--script-trace` shows the NSE network conversation, including commands sent, ports, and responses. Manual interaction also works:

```bash
nc -nv <TARGET_IP> 21
telnet <TARGET_IP> 21

# FTP with TLS/SSL — also reveals the certificate (hostname, email, locations)
openssl s_client -connect <TARGET_IP>:21 -starttls ftp
```

## SMB

Samba implements SMB/CIFS on Linux and Unix. NetBIOS services use TCP 137–139. Direct CIFS uses TCP 445.

### Default Configuration

```bash
cat /etc/samba/smb.conf | grep -v "#\|\;"
```

The configuration has a `[global]` section and one section per share. Share settings override global settings.

Relevant options include:

* `workgroup`
* `server string`
* `path`
* `unix password sync`
* `usershare allow guests`
* `map to guest = bad user`
* `browseable`
* `guest ok`
* `read only`
* `create mask`

`print$` and `IPC$` exist by default.

### Dangerous Settings

* `browseable = yes` — share listings are visible to employees and to us after access.
* `read only = no` or `writable = yes` — permits creating and modifying files.
* `guest ok = yes` — allows passwordless connection.
* `enable privileges = yes` — honors privileges assigned to a specific SID.
* `create mask = 0777` or `directory mask = 0777`
* `logon script = script.sh` — executed on user login.
* `magic script = script.sh` with `magic output = script.out` — executed when closed and stores output.

Example of a share using all of these settings, as might be left over from testing on an internal subnet:

```
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

After editing `smb.conf`:

```bash
sudo systemctl restart smbd
```

Enumerate shares through a null session. `-N` means no password.

```bash
smbclient -N -L //<TARGET_IP>
smbclient //<TARGET_IP>/notes
```

Use `help` within an SMB session for available commands.

* `get <file>` downloads a file.
* `!<cmd>` runs a local system command without dropping the connection, such as `!ls` or `!cat prep-prod.txt`.

On the server side, `smbstatus` shows who is connected, from which host, and to which share when run as root.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -sC -p139,445
```

Nmap reports the version, NetBIOS name, and signing status. Follow with manual RPC enumeration through a null session:

```bash
rpcclient -U "" <TARGET_IP>
```

Key queries:

```
srvinfo                   server info
enumdomains              all deployed domains
querydominfo              domain/server/user counts, server role
netshareenumall           all shares incl. paths
netsharegetinfo <share>   share details + DACL/ACE/SID
enumdomusers              domain users with RIDs
queryuser <RID>           user details (home drive, profile path, password times, bad pwd count)
querygroup <RID>          group info
```

If `enumdomusers` is blocked, `queryuser <RID>` usually still works. Brute-force RIDs:

```bash
for i in $(seq 500 1100);do rpcclient -N -U "" <TARGET_IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo "";done
```

Alternatives:

```bash
samrdump.py <TARGET_IP>                                  # impacket, users + details
smbmap -H <TARGET_IP>                                    # share permissions per share
crackmapexec smb <TARGET_IP> --shares -u '' -p ''        # shows READ/WRITE + signing status
```

`enum4linux-ng` automates many of these checks:

```bash
git clone https://github.com/cddmp/enum4linux-ng.git && cd enum4linux-ng
pip3 install -r requirements.txt
enum4linux-ng.py <TARGET_IP> -A
```

Important output includes:

* Whether null sessions are allowed.
* Whether random-username sessions are allowed; rerun with that user because it may yield more.
* Supported SMB dialects.
* Whether signing is required.
* Domain or workgroup.
* Users.
* Shares with mapping and listing status.
* Password policy, such as `min_pw_length: 5`, `DOMAIN_PASSWORD_COMPLEX: false`, and `lockout_threshold: None`.

{% hint style="info" %}
Cross-check with more than one tool. Results differ by implementation. Verify important findings manually.
{% endhint %}

## NFS

NFS is a Linux and Unix file-sharing protocol without built-in authentication. RPC options provide authentication, while UID/GID mappings provide authorization. Use NFS only on trusted networks.

### Default Configuration

`/etc/exports` lists shared filesystems, allowed hosts or subnets, and options.

Relevant options:

* `rw`
* `ro`
* `sync`
* `async`
* `secure` — only ports below 1024
* `insecure`
* `no_subtree_check`
* `root_squash` — maps root UID/GID 0 to anonymous and prevents root from accessing the mount

### Dangerous Settings

* `rw` — read and write access.
* `insecure` — permits ports above 1024.
* `nohide` — exports filesystems mounted under an exported directory.
* `no_root_squash` — files created by root retain UID/GID 0.

### Footprinting the Service

Essential ports are TCP 111 and 2049.

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

`ls -l` shows usernames and groups, while `ls -n` shows raw UIDs and GIDs. Recreate a local user with matching UID/GID to view and modify their files.

If `root_squash` is set, root-owned files cannot be edited even as root. An escalation path is to upload a shell with the SUID bit of a target user to the NFS share after obtaining SSH access, then execute it through the SSH user.

```bash
sudo umount ./target-NFS
```

## DNS

Records worth remembering:

* **A/AAAA** → IP address
* **MX** → mail servers
* **NS** → nameservers
* **TXT** → verification keys, SPF/DMARC/DKIM
* **CNAME** → alias
* **PTR** → reverse record
* **SOA** → zone and admin contact

The SOA `rname` converts to an email address by replacing the first `.` with `@`:

```bash
dig soa <TARGET_DOMAIN>
# ... SOA ns-161.awsdns-20.com. awsdns-hostmaster.amazon.com. ...  →  awsdns-hostmaster@amazon.com
```

### Default Configuration

There are three configuration file types:

* Local configuration
* Zone files
* Reverse-resolution files

Bind9 local configuration files include:

* `named.conf.local`
* `named.conf.options`
* `named.conf.log`

Zones are defined there:

```
zone "domain.com" {
    type master;
    file "/etc/bind/db.domain.com";
    allow-update { key rndc-key; };
};
```

The zone file, `/etc/bind/db.domain.com`, holds forward records in BIND format. Exactly one SOA and at least one NS record are required. A syntax error makes the entire zone unusable and the server responds with `SERVFAIL`.

A reverse zone, such as `/etc/bind/db.10.129.14` with `$ORIGIN 14.129.10.in-addr.arpa`, maps the final IP octet to a PTR hostname.

### Dangerous Settings

* `allow-query` — hosts permitted to send requests.
* `allow-recursion` — hosts permitted to send recursive requests.
* `allow-transfer` — hosts permitted to receive zone transfers.
* `zone-statistics` — collects zone statistics.

### Footprinting the Service

```bash
dig ns <TARGET_DOMAIN> @<TARGET_IP>          # other known nameservers — query them too
dig CH TXT version.bind <TARGET_IP>          # BIND version, if the entry exists
dig any <TARGET_DOMAIN> @<TARGET_IP>         # everything the server is willing to disclose
```

Zone transfers use AXFR over TCP 53. If `allow-transfer` is configured for a subnet, `any`, or a leftover workaround, the entire zone file can be retrieved, including other zones containing internal IPs and hostnames:

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

Postfix configuration is in `/etc/postfix/main.cf`.

Relevant commands:

* `HELO` / `EHLO` — start a session
* `AUTH PLAIN`
* `MAIL FROM`
* `RCPT TO`
* `DATA`
* `RSET`
* `VRFY`
* `EXPN`
* `NOOP`
* `QUIT`

```bash
telnet <TARGET_IP> 25
```

`VRFY` checks whether a mailbox exists. Some servers return `252 2.0.0 <anything>` for invalid usernames. Verify results independently.

Through a web proxy:

```
CONNECT <TARGET_IP>:25 HTTP/1.0
```

Sending an email manually:

```
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

Spoofed senders work because senders are not reliably authenticated.

### Dangerous Settings

An open relay occurs when admins do not know which IP ranges to allow and permit all:

```
mynetworks = 0.0.0.0/0
```

The default example is:

```
mynetworks = 127.0.0.0/8 <internal subnet>
```

An open relay enables sending fake emails, initializing communication between parties, or spoofing and reading mail.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sC -sV -p25                    # smtp-commands: EHLO capability list (incl. VRFY)
sudo nmap <TARGET_IP> -p25 --script smtp-open-relay   # 16 relay tests
```

## IMAP / POP3

IMAP manages mail online on the server using folder structures and multi-client synchronization. POP3 only lists, retrieves, and deletes messages.

| Protocol | Plaintext Port | TLS/SSL Port |
| -------- | -------------: | -----------: |
| IMAP     |            143 |          993 |
| POP3     |            110 |          995 |

### Default Configuration

Dovecot provides `dovecot-imapd` and `dovecot-pop3d`.

Core IMAP commands:

```
1 LOGIN <user> <pass>
1 LIST "" *
1 SELECT INBOX
1 FETCH <ID> all
1 LOGOUT
```

POP3 commands:

```
USER
PASS
STAT
LIST
RETR <id>
DELE <id>
CAPA
QUIT
```

### Dangerous Settings

* `auth_debug` — logs all authentication debug output.
* `auth_debug_passwords` — submitted passwords and scheme are logged.
* `auth_verbose` — logs failed authentication attempts and reasons.
* `auth_verbose_passwords` — passwords are logged, possibly truncated.
* `auth_anonymous_username` — username for the ANONYMOUS SASL mechanism.

Misconfigurations can allow reading all sent and received mail, which is often confidential.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -p110,143,993,995 -sC
```

Certificate subjects can reveal hostname, such as `mail1.<domain>`, organization, and location. Capabilities list available commands per port.

With credentials, such as a username reused as a password:

```bash
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD>       # lists mailboxes
curl -k 'imaps://<TARGET_IP>' --user <USERNAME>:<PASSWORD> -v    # adds TLS version, cert, banner (server version)

openssl s_client -connect <TARGET_IP>:pop3s
openssl s_client -connect <TARGET_IP>:imaps
```

## SNMP

SNMP uses UDP 161, with traps on UDP 162. Versions 1 and 2c have no real security: the community string acts as a password and travels in plaintext. Many organizations still run v2c because migration to v3 is complex.

Version 3 adds username/password authentication and encryption.

* **MIB**: a text file describing queryable objects.
* **OID**: a numbered node in the tree hierarchy. A longer chain is more specific.

### Default Configuration

```bash
cat /etc/snmp/snmpd.conf | grep -v "#" | sed -r '/^\s*$/d'
```

Defaults restrict the view:

```
view systemonly included .1.3.6.1.2.1.1
view systemonly included .1.3.6.1.2.1.25.1
rocommunity public
```

### Dangerous Settings

* `rwuser noauth` — full OID tree access without authentication.
* `rwcommunity <community string> <IPv4 address>` — full OID tree access regardless of request source.
* `rwcommunity6` — the same for IPv6.

### Footprinting the Service

```bash
snmpwalk -v2c -c public <TARGET_IP>
```

With the correct string and a no-auth version, SNMP can leak:

* OS version and kernel
* `syscontact` email
* Hostname
* Location
* Boot image
* Installed packages

If the community string is unknown:

```bash
onesixtyone -c snmp.txt <TARGET_IP>      # SecLists snmp.txt
```

Community strings are sometimes restricted to specific IPs and named after the hostname, possibly with symbols. In large networks, patterns may be guessable using custom wordlists with `crunch`.

Once a string is known, brute-force OIDs:

```bash
braa public@<TARGET_IP>:.1.3.6.*
```

## MySQL

MySQL configuration is in `/etc/mysql/mysql.conf.d/mysqld.cnf`, and the default port is 3306. Databases are often exported as single `.sql` files, such as `wordpress.sql`.

### Dangerous Settings

* `user`, `password`, and `admin_address` — plaintext configuration entries. Wrong file permissions plus any file-read or shell access can mean full database access.
* `debug` and `sql_warnings` — verbose errors often surface in web applications, confirm that the app talks to the database differently than intended, and help SQLi manipulation.
* `secure_file_priv` — limits data import and export operations.

### Footprinting the Service

```bash
sudo nmap <TARGET_IP> -sV -sC -p3306 --script mysql*
```

`mysql-info` provides the version reliably. `mysql-empty-password`, `mysql-brute`, and `mysql-enum` can generate false positives, so always verify manually:

```bash
mysql -u root -h <TARGET_IP>                 # no password
mysql -u root -p<PASSWORD> -h <TARGET_IP>    # no space between -p and password
```

Inside MySQL:

```sql
select version();
show databases;
use mysql; show tables;
use sys; show tables; select host, unique_users from host_summary;   # which hosts connect
show columns from <table>;
select * from <table>;
select * from <table> where <column> = "<string>";
```

The `sys` and `information_schema` databases are the interesting system databases because they contain metadata.

## MSSQL

MSSQL is closed source, has native .NET support, and is mostly found on Windows.

Default system databases:

* `master` — all system information.
* `model` — template for new databases.
* `msdb` — SQL Server Agent jobs and alerts.
* `tempdb` — temporary objects.
* `resource` — read-only system objects.

### MSSQL Clients

* SSMS
* `mssql-cli`
* SQL Server PowerShell
* HeidiSQL
* SQLPro
* Impacket's `mssqlclient.py`

SSMS is client-side, so a compromised box with SSMS may hold saved credentials.

### Default Configuration

MSSQL runs as `NT SERVICE\MSSQLSERVER`. Windows Authentication is enabled by default, with login processed through the local SAM or DC/AD.

A compromised account can lead to privilege escalation and lateral movement across the domain.

### Dangerous Settings

* Clients not using encryption.
* Self-signed certificates, which are spoofable when encryption is enabled.
* Named pipes in use.
* Weak and default `sa` credentials, often forgotten to be disabled.

### Footprinting the Service

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 <TARGET_IP>
```

This can reveal hostname, instance name, version, named pipe such as `\\<TARGET_IP>\pipe\sql\query`, and NTLM information including NetBIOS/DNS names and product version.

Metasploit alternative:

```
use auxiliary/scanner/mssql/mssql_ping
set rhosts <TARGET_IP>
run
```

With credentials, connect and enumerate using T-SQL:

```bash
mssqlclient.py <USERNAME>@<TARGET_IP> -windows-auth
```

```sql
select name from sys.databases
```

## Oracle TNS

The TNS listener runs on TCP 1521 by default. It was remotely manageable in Oracle 8i and 9i, but not in 10g or 11g.

Configuration files:

* `tnsnames.ora` — client-side service names mapped to addresses.
* `listener.ora` — server-side listener configuration.

Both are located in `$ORACLE_HOME/network/admin`.

Default passwords to remember:

* Oracle 9: `sys` = `CHANGE_ON_INSTALL`
* Oracle 10: no default password set
* DBSNMP service: `dbsnmp`

### Footprinting the Service

```bash
sudo nmap -p1521 -sV <TARGET_IP> --open
sudo nmap -p1521 -sV <TARGET_IP> --open --script oracle-sid-brute    # SID guessing, e.g. XE
```

The SID identifies a database instance. An incorrect SID prevents connection.

ODAT, Oracle Database Attacking Tool, enumerates and exploits versions, users, vulnerabilities, and misconfigurations. Setup on Pwnbox approximately requires installing `libaio1`, building and installing `cx_Oracle`, cloning https://github.com/quentinhardy/odat.git, and installing its pip dependencies.

Run all modules:

```bash
odat.py all -s <TARGET_IP>       # tries all modules, flags locked accounts, finds valid creds
```

With credentials, for example `scott/tiger`, connect with sqlplus. Install with `apt install oracle-instantclient-sqlplus`. If `libsqlplus.so` is missing:

```bash
echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf && ldconfig
```

Connect:

```bash
sqlplus scott/tiger@<TARGET_IP>/XE
sqlplus scott/tiger@<TARGET_IP>/XE as sysdba    # try logging in as sysdba if privileges allow
```

```sql
select table_name from all_tables;
select * from user_role_privs;      -- current user's roles
select name, password from sys.user$;   -- password hashes → crack offline
```

Web shell upload path requires a web server. Test first with a harmless file because AV or IPS may intervene:

```bash
echo "Oracle File Upload Test" > testing.txt
odat.py utlfile -s <TARGET_IP> -d XE -U scott -P tiger --sysdb --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
curl -X GET http://<TARGET_IP>/testing.txt
```

Default web roots:

* Linux: `/var/www/html`
* Windows: `C:\inetpub\wwwroot`

## IPMI

IPMI uses UDP 623. It is hardware-level host management independent of the OS, working before boot, when powered down, and after system failure.

BMCs, including HP iLO, Dell DRAC, and Supermicro IPMI, commonly expose a web console, Telnet/SSH, and port 623. BMC access is approximately equivalent to physical access: it can monitor, reboot, power off, or reinstall the OS.

{% hint style="warning" %}
IPMI is common on internal tests and should always be checked.
{% endhint %}

### Footprinting the Service

```bash
sudo nmap -sU --script ipmi-version -p 623 <TARGET_HOST>
```

Metasploit equivalent:

```
auxiliary/scanner/ipmi/ipmi_version
```

Default credentials worth trying:

| Platform        | Username        | Password                                                       |
| --------------- | --------------- | -------------------------------------------------------------- |
| Dell iDRAC      | `root`          | `calvin`                                                       |
| HP iLO          | `Administrator` | Randomized 8-character string of numbers and uppercase letters |
| Supermicro IPMI | `ADMIN`         | `ADMIN`                                                        |

### Dangerous Settings

The RAKP flaw in IPMI 2.0 sends a salted SHA1/MD5 hash of a user's password to the client before authentication. Password hashes for any valid account can be retrieved and cracked offline.

There is no direct fix because this behavior is part of the specification. Mitigations are long passwords and network segmentation.

```
use auxiliary/scanner/ipmi/ipmi_dumphashes
set rhosts <TARGET_IP>
run
```

Module options include `CRACK_COMMON`, which automatically cracks common passwords, and output files for Hashcat or John.

Offline cracking example:

```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u    # HP iLO factory-default style: 8 chars, digits+uppercase
```

Cracked BMC passwords are frequently reused elsewhere. In one engagement, an IPMI hash was cracked and then used to SSH as root into many critical servers and web consoles for monitoring tools.

## Linux Remote Management Protocols

Remote management services are high-value. Misconfiguration can mean direct access, and these services are prime targets for weak or default credentials and password reuse.

### SSH

Dangerous `sshd_config` settings:

* `PasswordAuthentication yes`
* `PermitEmptyPasswords yes`
* `PermitRootLogin yes`
* `Protocol 1` — outdated
* `X11Forwarding yes` — had a command injection vulnerability in OpenSSH 7.2p1
* `AllowTcpForwarding yes`
* `PermitTunnel`
* `DebianBanner yes`

Banner interpretation:

* `SSH-1.99-OpenSSH_3.9p1` accepts protocol versions 1 and 2.
* `SSH-2.0-OpenSSH_8.2p1` accepts only version 2.

Old versions can have known vulnerabilities, such as CVE-2020-14145, a MITM issue during the initial connection.

Audit server-side cryptographic configuration:

```bash
ssh-audit.py <TARGET_IP>      # from https://github.com/jtesta/ssh-audit
```

This shows the banner and version and flags weak KEX or host-key algorithms, including weak curves, weak hashing, and weak RNG.

Discover allowed authentication methods and force password authentication for brute-force:

```bash
ssh -v <USERNAME>@<TARGET_IP>
ssh -v <USERNAME>@<TARGET_IP> -o PreferredAuthentications=password
```

### Rsync

Rsync uses port 873 and is often unauthenticated. Test any discovered credentials against it because sensitive files retrieved this way can lead to remote access.

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

R-services are legacy, unencrypted, and occasionally still found internally.

| Service     | Port    |
| ----------- | ------- |
| rexec       | 512/TCP |
| rlogin      | 513/TCP |
| rsh and rcp | 514/TCP |

`rsh` opens a shell without a login procedure. `rcp` copies files in either direction with no overwrite warning.

```bash
sudo nmap -sV -p 512,513,514 <TARGET_IP>
```

Trust is configured in `/etc/hosts.equiv` globally and `~/.rhosts` per user. Syntax is `<username> <ip/hostname>`, and `+` is a wildcard:

```
htb-student     10.0.17.5
+               10.0.17.10    # any user from this host
+               +             # anyone from anywhere
```

Misconfigured trust can allow login without credentials:

```bash
rlogin <TARGET_IP> -l <USERNAME>
rwho                # interactive sessions on the local network (user + host)
rusers -al <TARGET_IP>   # detailed: user, host, TTY, login time, idle, remote host
```

### Final Thoughts

Probe remote management services for everything they disclose, especially after credentials have been compiled elsewhere in the network. Weak and default credentials, as well as password reuse, are recurring wins.

## Windows Remote Management Protocols

RDP, WinRM, and WMI are the main components.

### RDP

RDP uses TCP 3389 and may also use UDP. TLS has been available since Vista, but certificates are self-signed by default. NLA is the default requirement.

```bash
nmap -sV -sC <TARGET_IP> -p3389 --script rdp*
```

`rdp-enum-encryption` shows the security layer, including CredSSP/NLA. `rdp-ntlm-info` leaks `Target_Name`, NetBIOS and DNS computer names, and product version.

{% hint style="warning" %}
`--packet-trace` reveals the RDP cookie `mstshash=nmap`, which can be identified by threat hunters or EDR and can lock you out on hardened networks.
{% endhint %}

```bash
nmap -sV -sC <TARGET_IP> -p3389 --packet-trace --disable-arp-ping -n
```

Unauthenticated security-settings check using the Cisco CX Security tool. It requires `cpan > install Encoding::BER`:

```bash
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git && cd rdp-sec-check
rdp-sec-check.pl <TARGET_IP>
```

Connect with credentials. `rdesktop` and Remmina are alternatives:

```bash
xfreerdp /u:<USERNAME> /p:"<PASSWORD>" /v:<TARGET_IP>
```

### WinRM

WinRM is SOAP-based and uses TCP 5985 for HTTP and TCP 5986 for HTTPS. Often only TCP 5985 is used.

WinRS executes arbitrary remote commands, and PowerShell remote sessions require WinRM.

```bash
nmap -sV -sC <TARGET_IP> -p5985,5986 --disable-arp-ping -n
```

Reachability check from PowerShell:

```
Test-WsMan <host>
```

From Linux:

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>
```

### WMI

WMI has read and write access to nearly all Windows settings. It can be accessed through PowerShell, VBScript, or WMIC. Communication starts on TCP 135 and then moves to a random port.

```bash
wmiexec.py <USERNAME>:"<PASSWORD>"@<TARGET_IP> "hostname"
```
