# FTP — Footprinting

Nmap — version, default scripts, aggressive all at once. `ftp-anon` fires automatically and lists dir contents if anonymous is open:

```bash
sudo nmap -sV -p21 -sC -A <TARGET>
```

Add `--script-trace` if you want to see exactly what NSE sends/receives — useful when a script returns nothing but the port is open.

---

### Anonymous Login

```bash
ftp <TARGET>
# user: anonymous / pass: (blank)
```

Inside FTP — recursive listing in one shot if `ls_recurse_enable=YES`:

```bash
ls -R
```

Download everything at once — creates a local dir named after the target IP:

```bash
wget -m --no-passive ftp://anonymous:anonymous@<TARGET>
```

> [!warning] `wget -m` downloads the entire tree in one go — noisy, will look like bulk exfil in logs. Use `get <file>` for targeted pulls if opsec matters.

Test write access — if upload works and the FTP root overlaps with a web root, you have webshell drop potential:

```bash
# inside ftp:
put <LOCALFILE>
```

---

### Manual Interaction

Banner grab — version leaks on connect:

```bash
nc -nv <TARGET> 21
```

TLS-wrapped FTP — `openssl` handles it, also dumps the SSL cert (leaks hostname, email, org, CN):

```bash
openssl s_client -connect <TARGET>:21 -starttls ftp
```

> [!tip] SSL cert CN and emailAddress fields on FTP often leak internal hostnames and admin email addresses that don't appear in public DNS. Note them for username format and internal host discovery.

Upload → web root path traversal chain: FTP write access + web server on same host → drop webshell → RCE. Check if FTP root and web root overlap before assuming write access is low-value.