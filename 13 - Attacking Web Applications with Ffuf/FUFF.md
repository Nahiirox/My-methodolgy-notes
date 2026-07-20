# WEB FUZZING (FFUF)

## INTRODUCTION

Tool: ffuf — https://github.com/ffuf/ffuf. Covers directory fuzzing, file/extension fuzzing, vhost fuzzing, and parameter (name + value) fuzzing, GET and POST both.

## WEB FUZZING

Fuzzing = throwing structured input at an interface and reading how it reacts — special chars for SQLi, long incrementing strings for buffer overflows, wordlists of common terms for directory/page discovery. This module's flavor is the last one: since a webserver won't hand you a sitemap, you brute-force plausible paths and read the status code back. 404 = doesn't exist, 200 = it does, go look manually.

Wordlist source: SecLists (https://github.com/danielmiessler/SecLists), mounted at `/opt/useful/seclists` on Pwnbox. Main one for this module:

```
/opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
```

Not exhaustive — randomly-named or unique pages won't show up regardless of wordlist size — but gets the majority on most targets. Wordlist files sometimes carry copyright header comments that pollute results; strip with `-ic` (ignore comments).

## DIRECTORY FUZZING

```bash
sudo apt install ffuf -y   # or pre-installed on Pwnbox
```

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ
```

`:FUZZ` binds a keyword to the wordlist, then that keyword marks the injection point in `-u`. Default matcher already excludes 404s (`200,204,301,302,307,401,403`), so a clean run only shows real hits.

Default threads = 40. Cranking to `-t 200` speeds things up but risks tripping rate limits or actually DoSing a remote target — don't do it against anything you don't control outright.

A hit with an empty body isn't necessarily a dead end — it just means no index page at that path. Worth a follow-up fuzz underneath it before writing it off.

## PAGE & EXTENSION FUZZING

Guessing the extension off the server header (Apache → `.php`, IIS → `.asp`/`.aspx`) is unreliable — fuzz it directly instead. `index` exists on nearly everything, so fuzz the extension against that:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://<TARGET_IP>:<PORT>/<DIR>/indexFUZZ
```

(wordlist already includes the leading dot — don't add another). A `200` confirms the real extension in play; other extensions may still return `403`/etc. and aren't it.

Extension confirmed, reuse the directory wordlist against it to find actual pages:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/<DIR>/FUZZ.php
```

Size-0 hits are still real, accessible pages — just empty output, not a false positive.

## RECURSIVE FUZZING

Automates re-running the scan inside every newly discovered directory instead of doing it by hand per-directory.

```
-recursion              enable
-recursion-depth N      cap how deep — 1 = fuzz direct subdirs of any hit, don't go further
-e .php                 extension appended during the recursive pass (safe to assume site-wide)
-v                      verbose, full URLs — without this you can't tell which .php belongs to which directory
```

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET_IP>:<PORT>/FUZZ -recursion -recursion-depth 1 -e .php -v
```

Request count multiplies fast — the wordlist effectively runs twice per directory found (with and without the extension) — so always set a depth cap rather than letting it run unbounded on a deep tree.

## DNS / HOSTS RESOLUTION

A discovered page can reference a domain name (e.g. a message pointing to `academy.htb`) that resolves to nothing in a browser — `academy.htb` isn't a public site, so there's no public DNS record for it. Browsers only resolve a name via `/etc/hosts` or public DNS; if it's in neither, connection just fails.

Fix by mapping it manually:

```bash
sudo sh -c 'echo "<TARGET_IP>  academy.htb" >> /etc/hosts'
```

If the content's identical to what you already had at the bare IP, the domain's just an alias, not a separate target. If a recursive scan through the named domain still doesn't surface something you know should exist (e.g. an admin panel mentioned in a message), the next move is subdomains or vhosts — see below.

## SUB-DOMAIN FUZZING

Finds subdomains via **public DNS only** — irrelevant for internal/lab targets or anything not publicly resolvable, no matter how good the wordlist is.

```
/opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.<TARGET_DOMAIN>/
```

Zero hits on something like `*.academy.htb` doesn't mean no subdomains exist — it means none are publicly resolvable. Adding the base domain to `/etc/hosts` doesn't help here either; ffuf's still asking public DNS for each `FUZZ`'d name, not checking the hosts file.

## VHOST FUZZING

The actual technique for private/internal subdomains. A vhost can share the exact same IP as the main site with zero public DNS trace — DNS-based fuzzing is structurally blind to it. Instead of fuzzing the URL, fuzz the `Host` header while hitting the same IP every request:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/ -H 'Host: FUZZ.<TARGET_DOMAIN>'
```

Every single request returns `200` here — you're hitting one real server regardless of the header value — so status-code matching is useless. Note the response size for a known-wrong value first; that's your baseline noise to filter out (next section).

## FILTERING RESULTS

Match and filter are opposite sides of the same knob:

```
-mc   match status code(s)     -fc   filter OUT status code(s)
-ms   match response size      -fs   filter OUT response size
-mw   match word count         -fw   filter OUT word count
-ml   match line count         -fl   filter OUT line count
-mr   match regex              -fr   filter OUT regex
```

Default matcher (`200,204,301,302,307,401,403`) already strips 404s automatically — that's why an unfiltered directory scan still looks clean.

Vhost fuzzing is the case where matching breaks down (everything's `200`) — filter by size instead:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/ -H 'Host: FUZZ.<TARGET_DOMAIN>' -fs <BASELINE_SIZE>
```

A real vhost returns a genuinely different page → different size than the fallback default → survives the filter and stands out.

## PARAMETER FUZZING — GET

Applies when a page is clearly gated behind something — blank/restricted content, no visible form, no session cookie — and a hidden parameter is the likely missing piece.

```
/opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt
```

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/admin/admin.php?FUZZ=key -fs <BASELINE_SIZE>
```

A hit confirms the parameter name is real and being read server-side — not that the value you sent is correct, just that the param itself exists.

Unpublished parameters found this way tend to be less tested and less secured than anything exposed through the actual UI — worth revisiting later with other vuln classes (SQLi, XSS, etc.) once found.

## PARAMETER FUZZING — POST

POST data doesn't live in the URL, so it needs `-d` plus `-X POST` instead of a `?param=` in `-u`. PHP backends specifically expect `application/x-www-form-urlencoded` unless told otherwise:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs <BASELINE_SIZE>
```

Confirm a hit manually before building anything further on it:

```bash
curl http://<TARGET_DOMAIN>:<PORT>/admin/admin.php -X POST -d '<PARAM>=key' -H 'Content-Type: application/x-www-form-urlencoded'
```

A response body change (e.g. an "Invalid id!" message instead of silence) confirms the param's actually being processed, not just ignored.

## VALUE FUZZING

Parameter name confirmed, now the value itself needs discovery — no universal wordlist fits every param type, since expected format depends entirely on what the param represents. Match the wordlist to the guess: usernames → a SecLists username list or a custom OSINT-built one; numeric IDs → generate a sequential range.

```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

Same POST fuzz pattern, wordlist now sits at the value position instead of the name:

```bash
ffuf -w ids.txt:FUZZ -u http://<TARGET_DOMAIN>:<PORT>/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs <BASELINE_SIZE>
```

Hit = working value — confirm with `curl` the same way as the parameter-name step and pull whatever the endpoint returns.