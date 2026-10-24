# Web Reconnaissance

## Introduction
![[Pasted image 20261024043530.png]]
Active recon = direct interaction, higher detection risk. Passive = publicly available info, stealthier but less comprehensive.

**Active:** port scanning (`nmap`, `masscan`, `unicornscan`), vuln scanning (Nessus, OpenVAS, Nikto), network mapping (traceroute, nmap), banner grabbing (`nc`, `curl`), OS fingerprinting (nmap `-O`, Xprobe2), service enumeration (nmap `-sV`), web spidering (Burp Spider, ZAP Spider, Scrapy).

**Passive:** search engines, WHOIS, DNS (`dig`, `nslookup`, `host`, `dnsenum`, `fierce`, `dnsrecon`), Wayback Machine, social media, code repos.

## WHOIS

Query the domain registration database for registrant, admin/tech contacts, creation/expiry dates, name servers.

```bash
whois <TARGET_DOMAIN>
```

Fields that matter in recon: name servers (hosting provider), registrant org, creation date (young domain = phishing indicator), privacy service (hidden registrant), registrar abuse history.

Historical WHOIS via WhoisFreaks to track ownership changes.

## Utilising WHOIS

Reading WHOIS is the whole skill:

- **Registration date just days old** → phishing / disposable infra
- **Registrant behind privacy service** → suspicious
- **Name servers at a known bulletproof host** → strong phishing indicator
- **Registrant on free anonymous email, address in high-cybercrime country, lax-abuse registrar** → likely compromised/bulletproof C2
- **Clustered registration dates + shared name servers across many domains** → single threat actor

Established domains (e.g., facebook.com): all six lock statuses set, NS inside the domain, Registrant/Admin/Tech all populated. Hardened. WHOIS alone rarely reveals employees or vulns — combine with other recon.

## DNS
![Flowchart showing two main sections: User Database and Data Collection. Includes steps like 'Check User', 'Send to Source Database', 'Store Data', and 'Send to UI System'.](https://mermaid.ink/svg/pako:eNptkk1uwjAQha8y8rpcIItWkAAtUNQmlSrksDDxlEQQT-QfJIS4ex2nNG1arzx-n5-ex3NhBUlkEdtr0ZSwSnMFfhm36w425DTEVDfOou60do15XGJxMBCLosQtjEb3MLk8vcCMnJIP156ceA3WFIiYZ6ikgWSdwatDfQZLkKKh4wn1dnBngyZceuQxKYWFNS39jjtTWfyCvdsgb2t9c-wN4-CU_A7dy8mPjFOeYuG0qU4IK6KDa4bgLdjck9ZpZcC_20e7dWmYbRroGU-JLKxFjZCh7h88C_KCv62Sf9RFUJd87GxJurLCtsH-cssuUlfMu8blit2xGnUtKul_-NKKObMl1pizyG-l0Iec5erqOeEsZWdVsMhqh3dMk9uXLPoQR-Mr10hhMamE73L9fYqysqSfuwEKc3T9BOe0sj4)
**Records worth knowing operationally:**

- **A / AAAA** — IPv4 / IPv6
- **CNAME** — alias; a CNAME pointing at an outdated server (`dev.example.com CNAME oldserver.example.net`) is a lead
- **MX** — mail servers
- **NS** — nameservers; usually reveals hosting provider
- **TXT** — verification keys, SPF, DMARC, service leaks (`_1password=...`)
- **SOA** — zone admin contact, primary NS
- **SRV** — service hostname + port
- **PTR** — reverse lookup

**Hosts file** — local DNS override, bypasses DNS.

```text
# /etc/hosts (Linux/macOS), C:\Windows\System32\drivers\etc\hosts (Windows)
<IP Address>    <Hostname> [<Alias> ...]

127.0.0.1       myapp.local
0.0.0.0         unwanted-site.com
```

## Digging DNS

**dig** — the workhorse. Query any record type against a specific server.

```bash
dig <TARGET_DOMAIN>
dig <TARGET_DOMAIN> A
dig <TARGET_DOMAIN> AAAA
dig <TARGET_DOMAIN> MX
dig <TARGET_DOMAIN> NS
dig <TARGET_DOMAIN> TXT
dig <TARGET_DOMAIN> CNAME
dig <TARGET_DOMAIN> SOA
dig @1.1.1.1 <TARGET_DOMAIN>
dig +trace <TARGET_DOMAIN>
dig -x <TARGET_IP>
dig +short <TARGET_DOMAIN>
dig +noall +answer <TARGET_DOMAIN>
dig <TARGET_DOMAIN> ANY
```

`ANY` unreliable — many servers ignore it per RFC 8482.

Output has four sections: header (status, flags, counts), question, answer, footer (query time, server, timestamp, size). Header flags worth reading: `qr` (response), `rd` (recursion desired), `ad` (authentic data).

`+short` drops everything except the answer:

```bash
dig +short <TARGET_DOMAIN>
```

## Subdomains

Subdomains often host the things not linked from the main site:

- **dev/staging** — relaxed security, sometimes vulnerable
- **hidden login panels** — admin portals meant to be internal
- **legacy apps** — forgotten, outdated, vulnerable
- **sensitive files** — internal docs, configs, exposed by accident

Two enumeration approaches:

**Active** — zone transfer attempt (`dig axfr`), brute force (`dnsenum`, `ffuf`, `gobuster`).

**Passive** — Certificate Transparency logs, search engines (`site:`), aggregated DNS databases.

Active is complete but noisy; passive is quiet but incomplete. Run both.

## Subdomain Bruteforcing

Four steps: pick wordlist → append to domain → DNS lookup (A/AAAA) → keep what resolves.

Wordlist types:
- **General-purpose** — dev, staging, blog, mail, admin, test
- **Targeted** — industry/tech-specific
- **Custom** — from intel gathered elsewhere

```bash
dnsenum --enum <TARGET_DOMAIN> -f subdomains-top1million-20000.txt -r
```

`--enum` tuning shortcut, `-f` wordlist, `-r` recursive (enumerate subdomains of found subdomains).

Other tools: `fierce`, `dnsrecon`, `amass`, `assetfinder`, `puredns`.

## DNS Zone Transfers
![[Pasted image 20261024043633.png]]
**AXFR** pulls the full zone from primary to secondary. If `allow-transfer` is loose, anyone gets it.

```bash
dig axfr @<NAMESERVER> <TARGET_DOMAIN>
```

```bash
dig axfr @nsztm1.digi.ninja zonetransfer.me
```

Success hands you every subdomain, IP, and NS record in the zone — including internal hosts that are never linked publicly. Even a failed attempt tells you something about server config.

## Virtual Hosts
![[Pasted image 20261024043647.png]]

**Subdomain** = DNS record. **VHost** = web server config matched on the `Host` header. A vhost can exist with no DNS record at all — which is why you fuzz for it.

Apache example:

```apacheconf
<VirtualHost *:80>
    ServerName www.example1.com
    DocumentRoot /var/www/example1
</VirtualHost>
<VirtualHost *:80>
    ServerName www.example2.org
    DocumentRoot /var/www/example2
</VirtualHost>
```

Three types: **name-based** (Host header, most common), **IP-based** (unique IP per site), **port-based** (different port per site).

```bash
gobuster vhost -u http://<TARGET_IP> -w <WORDLIST> --append-domain
```

`--append-domain` required in newer Gobuster — without it the full vhost name isn't constructed. Other flags: `-t` threads, `-k` ignore TLS errors, `-o` output file.

Vhost fuzzing is loud — IDS/WAF bait. Confirm scope first.

## Certificate Transparency Logs

**CT logs** — public, append-only ledger of every issued cert. Sanctioned record of subdomains, not guesswork. Includes subdomains from old/expired certs, which may still be live and unpatched.

```bash
curl -s "https://crt.sh/?q=<TARGET_DOMAIN>&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u
```

**crt.sh** — free, no registration, limited filtering. **Censys** — deeper filtering by domain, IP, cert attribute; requires registration.

## Fingerprinting

Extract server tech, version, CMS, WAF from headers, error responses, and page content.

```bash
curl -I <TARGET_DOMAIN>
```

Read `Server:`, `X-Powered-By:`. Redirects often leak more than the destination — `X-Redirect-By: WordPress` is a direct tell. `Link:` headers with `/wp-json/` confirm WordPress.

```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
wafw00f <TARGET_DOMAIN>
```

Tells you whether a WAF sits in front (Wordfence, Cloudflare, etc.) before you start probing.

```bash
nikto -h <TARGET_DOMAIN> -Tuning b
```

`-Tuning b` = software identification modules only. Output worth noting: server version vs current, missing `Strict-Transport-Security`, missing `X-Content-Type-Options`, `x-redirect-by`, exposed `license.txt`, discovered CMS and login paths, cookie flags on login pages.

Other tools: Wappalyzer, BuiltWith, WhatWeb, Netcraft.

## Crawling
![[Pasted image 20261024043727.png]]

**Breadth-first** — wide before deep. Good for structure overview.

**Depth-first** — follow one path to the end before backtracking. Good for reaching deep content.

What to extract:

- **Links** (internal/external) — site structure, hidden pages, external relationships
- **Comments** — users leak internal process, versions, hints
- **Metadata** — titles, descriptions, authors, dates
- **Sensitive files** — `.bak`, `.old`, `web.config`, `settings.php`, `error_log`, `access_log`

Value is in correlation, not single findings. A cluster of URLs pointing at `/files/` is worth visiting by hand — directory listing enabled there turns a mundane link list into exposed backups and internal docs. A comment mentioning a "file server" then reinforces it.

## robots.txt

```text
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://www.example.com/sitemap.xml
```

`Disallow` entries are the interesting part — owners are telling you where the sensitive stuff is. Honeypot directories also get planted here to catch bots that ignore the file.

## Well-Known URIs

Standardised location under `/.well-known/` per RFC 8615. Worth requesting:

- `/.well-known/security.txt` — security contact
- `/.well-known/change-password` — password change page
- `/.well-known/openid-configuration` — OIDC config
- `/.well-known/assetlinks.json` — digital asset ownership
- `/.well-known/mta-sts.txt` — SMTP MTA-STS policy

`openid-configuration` is the most productive:

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "jwks_uri": "https://example.com/oauth2/jwks",
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

Each field is a target: auth/token/userinfo endpoints to test, `jwks_uri` for keys, supported scopes to map what's available, signing algorithms to spot weak config.

## Creepy Crawlies

Tools: Burp Spider, OWASP ZAP Spider, Scrapy (custom crawlers), Apache Nutch (large-scale).

ReconSpider run:

```bash
pip3 install scrapy
wget -O ReconSpider.zip <URL>
unzip ReconSpider.zip
python3 ReconSpider.py http://<TARGET_DOMAIN>
```

Output lands in `results.json` with these keys: `emails`, `links`, `external_files`, `js_files`, `form_fields`, `images`, `videos`, `audio`, `comments`. Emails feed phishing; external files and JS hide paths and endpoints; HTML comments carry dev leftovers.

## Search Engine Discovery

Core operators:

- `site:` — restrict to a domain
- `inurl:` — term in URL
- `filetype:` — file type
- `intitle:` — term in title
- `intext:` / `inbody:` — term in body
- `cache:` — cached copy
- `link:` — pages linking to a URL
- `related:` — similar sites
- `info:` — page summary
- `numrange:` — numeric range
- `allintext:` / `allinurl:` / `allintitle:` — all terms, body/URL/title
- `AND` / `OR` / `NOT` — boolean
- `*` — wildcard
- `..` — numeric range
- `" "` — exact phrase
- `-` — exclude

Common dorks:

```text
site:<TARGET_DOMAIN> inurl:login
site:<TARGET_DOMAIN> (inurl:login OR inurl:admin)
site:<TARGET_DOMAIN> filetype:pdf
site:<TARGET_DOMAIN> (filetype:xls OR filetype:docx)
site:<TARGET_DOMAIN> inurl:config.php
site:<TARGET_DOMAIN> (ext:conf OR ext:cnf)
site:<TARGET_DOMAIN> inurl:backup
site:<TARGET_DOMAIN> filetype:sql
```

Google Hacking Database for more.

## Web Archives

**Wayback Machine** — snapshots since 1996. Crawl → archive → access; frequency varies by site popularity.

For recon: old pages, directories, files, subdomains that no longer exist but may still be live; tech changes over time; content as OSINT. All passive — you never touch the target.

Feed any URL from earlier recon and check the earliest and latest captures.

## Automating Recon

Frameworks: FinalRecon, Recon-ng, theHarvester, SpiderFoot, OSINT Framework.

FinalRecon setup:

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
./finalrecon.py --help
```

Modules and flags:

```text
--headers    header information
--sslinfo    SSL certificate info
--whois      Whois lookup
--crawl      crawl target
--dns        DNS enumeration (40+ record types incl. DMARC)
--sub        subdomain enumeration (crt.sh, AnubisDB, ThreatMiner, CertSpotter, Facebook, VirusTotal, Shodan, BeVigil)
--dir        directory search
--wayback    Wayback URLs (last 5 years)
--ps         fast port scan
--full       full recon
```

Extra: `-nb` hide banner, `-dt` dir threads, `-pt` port threads, `-T` timeout, `-w` wordlist, `-r` allow redirects, `-s` toggle SSL verify, `-sp` SSL port, `-d` custom DNS, `-e` file extensions, `-o` export format, `-cd` export dir, `-k` API key.

```bash
./finalrecon.py --headers --whois --url http://<TARGET_DOMAIN>
./finalrecon.py --full --url http://<TARGET_DOMAIN>
```

Exports land in `~/.local/share/finalrecon/dumps/`.