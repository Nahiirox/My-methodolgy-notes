# Attacking Web Applications with Ffuf

Ffuf = the go-to web fuzzer: send wordlist-driven requests, read response codes/sizes, separate real pages from 404 noise. Covers directories, files/extensions, vhosts, parameters, and parameter values. Hundreds of requests per second, so what would take hours manually finishes in seconds.

## Web Fuzzing

Fuzzing = throwing varied input at an interface and studying the reaction (SQLi → special characters, buffer overflow → growing strings; web → wordlists of common paths). Servers don't publish their page tree — we probe with common names and let the HTTP code answer: 404 = no page, 200 = real page.

Wordlists: don't build your own — **SecLists** (`/opt/useful/SecLists` on Pwnbox) covers every fuzzing category, passwords included. The standard directory wordlist:

```text
/opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
```

It starts with copyright comments that pollute results — `-ic` strips them.

## Directory Fuzzing

Core ffuf pattern: `-w` wordlist (optionally `:KEYWORD` to name it) + `-u` URL with the keyword where the fuzz target sits:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ
```

~88k requests in under 10 seconds at the default 40 threads. `-t 200` pushes it faster but can DoS a remote target or choke your own connection — don't. Results show `[Status: 301, Size: 326, ...]` per hit; a 301/200 directory with an empty page still means access (not 404/403) — go fuzz pages under it.

Key flags for the rest of the module: `-H` header, `-X` method, `-b` cookies, `-d` POST data, `-mc/-ms` matchers (default match = 200,204,301,302,307,401,403), `-fc/-fs` filters, `-o` output.

## Page Fuzzing

**Extension fuzzing first** — find what the site runs before fuzzing filenames. Guessing from server headers (apache → php, IIS → asp/aspx) is unreliable; fuzz it instead. Every site has `index.*`, so fuzz the extension on it (the web-extensions wordlist already contains the dot):

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://SERVER_IP:PORT/blog/indexFUZZ
```

`.php` returns 200 → PHP site. (Two wordlists with two keywords — `FUZZ_1.FUZZ_2` — fuzz name and extension together when needed.)

**Page fuzzing** with the known extension:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php
```

Read the Size column: `index` at size 0 = empty page; anything non-zero has content worth visiting.

## Recursive Fuzzing

Automate the directory → subdirectory → files loop:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v
```

`-recursion` re-scans every newly found directory; `-recursion-depth 1` stops at direct subdirectories (deep trees explode the scan time — depth-limit first, then re-run targeted at interesting branches); `-e .php` appends the extension (effectively doubles the wordlist — with and without); `-v` prints full URLs so you can tell which file belongs to which directory. Costs ~6× the requests of the flat scan, but one command yields the whole tree.

## DNS Records

Browsers resolve names via `/etc/hosts` then public DNS. Internal/lab hostnames like `academy.htb` exist in neither → "can't connect". Add them:

```bash
sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'
```

If the named domain serves the same site as the raw IP, the next move is hunting sub-domains under it.

## Sub-domain Fuzzing

Finds **public** sub-domains — ones with DNS records pointing at a working server. SecLists has dedicated lists under `/opt/useful/seclists/Discovery/DNS/`:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.inlanefreight.com/
```

Hits = resolvable, live sub-domains (support, ns3, blog, www...). On a non-public domain (academy.htb), the same scan returns all errors — the main domain only resolves via /etc/hosts, and no other sub-domain exists in public DNS. That's not "no sub-domains"; it's "no public ones" — see vhost fuzzing.

## Vhost Fuzzing

**VHost** = sub-domain served on the same server/IP (one IP, multiple sites). May lack public DNS records entirely — sub-domain fuzzing can't see them. Vhost fuzzing hits the IP we already have and varies the `Host:` header instead:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb'
```

Everything returns 200 — the default page always answers. The tell is response **size**: a recognized VHost serves its own page, which differs from the default. Filter out the default size and the real VHosts are what's left:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs 900
```

Verify by adding the hit to `/etc/hosts` (`admin.academy.htb`) and browsing — different content and a 404 on paths that exist on the main host confirm a separate VHost.

## Filtering Results

Default filtering keeps 200/204/301/302/307/401/403 and drops 404. When everything returns 200 (vhost fuzzing, custom 404 pages), filter on the response body fingerprint instead:

- `-fs` / `-ms` — response size (the workhorse: filter the constant default-page size)
- `-fw` / `-mw` — word count; `-fl` / `-ml` — line count
- `-fr` / `-mr` — regexp on response
- `-fc` / `-mc` — status codes and ranges

Use **matching** when you know what a hit looks like, **filtering** when you know what the misses look like (the usual case — grab the size from the noise responses and exclude it).

## Parameter Fuzzing - GET

Pages like `admin.php` refusing access without a login may accept an undocumented parameter (`?param1=key`). GET params hang off the URL after `?` — put the keyword there:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key -fs xxx
```

(-fs xxx = whatever the default response size turns out to be.) Hits can be dead ends — a deprecated parameter still responds differently but is no longer in use.

Fuzzing parameters regularly exposes unpublished ones: less tested, less secured — prime targets for the vulnerability classes covered elsewhere.

## Parameter Fuzzing - POST

POST data lives in the request body, not the URL. `-d` sets the data, `-X POST` the method, and PHP only accepts `application/x-www-form-urlencoded`:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx
```

Verify hits directly:

```bash
curl http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=key' -H 'Content-Type: application/x-www-form-urlencoded'
```

A changed error (`Invalid id!`) confirms the parameter is live.

## Value Fuzzing

Last hop: fuzz the accepted parameter's value. Pre-made lists work for guessable values (usernames); custom parameters need a custom list. For a numeric `id`, generate one:

```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

Fuzz the value position:

```bash
ffuf -w ids.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx
```

Hit → replay with curl → flag. The whole module's chain in one breath: fuzz directories → fuzz extensions → fuzz pages → recurse → vhost-fuzz past the public DNS boundary → fuzz GET params → fuzz POST params → build a value list → fuzz the value.