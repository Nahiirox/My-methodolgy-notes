# ffuf — Web Fuzzing

Install: `sudo apt install ffuf -y` | GitHub: https://github.com/ffuf/ffuf

SecLists wordlists on Parrot/Kali: `/opt/useful/seclists/` or `/usr/share/seclists/`

Key wordlists:

- Dirs/pages: `/opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt`
- Extensions: `/opt/useful/seclists/Discovery/Web-Content/web-extensions.txt`
- Subdomains/vhosts: `/opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt`
- Parameters: `/opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt`

`-ic` strips wordlist comments from results.

---

## Directory Fuzzing

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET>:<PORT>/FUZZ
```

---

## Extension Fuzzing

Find what extension the server uses — test `indexFUZZ` on a known directory:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://<TARGET>:<PORT>/blog/indexFUZZ
```

---

## Page Fuzzing

Once extension is known (e.g. `.php`):

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET>:<PORT>/blog/FUZZ.php
```

---

## Recursive Fuzzing

Scans dirs + their immediate subdirs + files with given extension. Use `-v` for full URLs:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET>:<PORT>/FUZZ -recursion -recursion-depth 1 -e .php -v
```

---

## Subdomain Fuzzing (public DNS)

Works only for publicly resolvable subdomains:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.<TARGET>/
```

---

## Vhost Fuzzing (non-public / same IP)

Fuzz the `Host:` header — finds vhosts that don't have public DNS records. All responses return 200 initially → filter out the default response size first:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://<TARGET>:<PORT>/ -H 'Host: FUZZ.<DOMAIN>' -fs <DEFAULT_SIZE>
```

Run once without `-fs` to get the default size, then add `-fs <size>` to filter it.

After finding a vhost, add it to `/etc/hosts`:

```bash
sudo sh -c 'echo "<TARGET_IP>  <VHOST>.<DOMAIN>" >> /etc/hosts'
```

---

## Parameter Fuzzing — GET

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://<TARGET>:<PORT>/admin/admin.php?FUZZ=key -fs <DEFAULT_SIZE>
```

---

## Parameter Fuzzing — POST

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://<TARGET>:<PORT>/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs <DEFAULT_SIZE>
```

Verify found param with curl:

```bash
curl http://<TARGET>:<PORT>/admin/admin.php -X POST -d 'id=key' -H 'Content-Type: application/x-www-form-urlencoded'
```

---

## Value Fuzzing

Generate numeric wordlist (1-1000):

```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

Fuzz the value:

```bash
ffuf -w ids.txt:FUZZ -u http://<TARGET>:<PORT>/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs <DEFAULT_SIZE>
```

---

## Key Flags Reference

`-fc <code>` filter by HTTP code | `-fs <size>` filter by response size | `-fw <words>` filter by word count | `-fl <lines>` filter by line count | `-mc <code>` match code | `-ms <size>` match size | `-H` custom header | `-b` cookie | `-d` POST data | `-X` HTTP method | `-o` output file | `-t` threads (default 40) | `-v` verbose with full URLs | `-ic` ignore wordlist comments

> [!tip] The quickest workflow: run once without filters to see the default response size, then rerun with `-fs <size>` to filter it out. This applies to vhost, parameter, and value fuzzing.

> [!warning] `-recursion` doubles the request count (tries each word with and without extension). Set `-recursion-depth 1` to avoid exploding scan time on deep directory trees.