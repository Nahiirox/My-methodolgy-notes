# Filtering ffuf Results

Filter/match options: `-fc`/`-mc` (status code), `-fs`/`-ms` (size), `-fw`/`-mw` (words), `-fl`/`-ml` (lines), `-fr`/`-mr` (regexp).

Vhost fuzzing returns uniform 200s — get the baseline size from the noise, then filter it out:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://<DOMAIN>:<PORT>/ -H 'Host: FUZZ.<DOMAIN>' -fs <BASELINE_SIZE>
```

Real vhost hit = different size in results. Add it to `/etc/hosts` before visiting:

```bash
sudo sh -c 'echo "<TARGET>  <SUBDOMAIN>.<DOMAIN>" >> /etc/hosts'
```

Confirm it's a separate vhost by checking a path you know exists on the main domain — 404 there means you're on different content.

> [!tip] Use `-fs` when you know the noise size, `-ms` when you know the hit size. For vhosts you almost always know the noise — run unfiltered for 2 seconds, read the uniform size, rerun with `-fs`.