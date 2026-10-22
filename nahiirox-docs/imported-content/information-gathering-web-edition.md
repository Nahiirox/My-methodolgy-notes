# Information Gathering   Web Edition

Recon = the Information Gathering phase before vuln assessment/exploitation. Goals: identify assets (pages, subdomains, IPs, technologies), discover hidden info (backups, configs, internal docs), analyse the attack surface, gather intel for exploitation or social engineering.

## Introduction

Two approaches — active (direct interaction, detectable) vs passive (public sources only, stealthy):

* **Active:** port scanning (Nmap, Masscan, Unicornscan), vulnerability scanning (Nessus, OpenVAS, Nikto), network mapping (traceroute), banner grabbing (netcat, curl), OS fingerprinting (`nmap -O`), service enumeration (`nmap -sV`), web spidering (Burp Spider, OWASP ZAP). Anything active can trip IDS/firewalls; scanners sending exploit payloads are the loudest.
* **Passive:** search engine queries, WHOIS lookups, DNS record analysis (dig, nslookup, host, dnsenum, fierce, dnsrecon), web archive analysis (Wayback Machine), social media, code repositories (GitHub/GitLab). Normal internet activity — very low detection risk.

## WHOIS

Query/response protocol for registered resources — domains, IP blocks, ASNs.

```bash
whois <TARGET_DOMAIN>
```

A record holds: registrar, registrant / administrative / technical contacts, creation + expiration dates, name servers.

Recon value:

* Contacts (names, emails, phones) → social engineering / phishing targets
* Name servers and IPs → hints at network infrastructure and hosting
* Historical WHOIS (e.g. WhoisFreaks) → ownership/contact/infrastructure changes over time

## Utilising WHOIS

```bash
sudo apt install whois -y
whois facebook.com
```

Reading the output:

* Registrar + creation/expiry dates → legitimacy, how established the domain is
* Registrant organization + contact → who's behind it
* Domain statuses (`clientDeleteProhibited`, `clientTransferProhibited`, `clientUpdateProhibited`, server-side equivalents) → domain locked against changes/transfers
* NS records inside the domain itself → org runs its own DNS (common for large orgs)

WHOIS rarely gives individual employees or specific vulns — combine it with other recon techniques.

## DNS

Resolution chain: client cache → resolver (ISP) → root server → TLD server (.com/.org) → authoritative server → answer, cached along the way.

Hosts file = local override of DNS:

* Linux/macOS: `/etc/hosts`
* Windows: `C:\Windows\System32\drivers\etc\hosts`

Format: `<IP> <hostname> [alias]`

Changes apply immediately. Used for dev redirects, connectivity tests, blocking (point a domain at `0.0.0.0`).

A zone is the part of the namespace one entity manages; the zone file on the DNS server defines its records:

```
$TTL 3600
@       IN SOA   ns1.example.com. admin.example.com. (
                2024060401 3600 900 604800 86400 )
@       IN NS    ns1.example.com.
@       IN NS    ns2.example.com.
@       IN MX 10 mail.example.com.
www     IN A     192.0.2.1
mail    IN A     198.51.100.1
ftp     IN CNAME www.example.com.
```

Record types:

* **A / AAAA:** → IPv4/IPv6
* **CNAME:** alias → hostname
* **MX:** mail
* **NS:** delegation
* **TXT:** verification, SPF etc.
* **SOA:** zone admin info
* **SRV:** service host+port
* **PTR:** reverse lookup

Recon angles:

* CNAME pointing at an outdated server → likely vulnerable system
* NS records → hosting provider; an `A` record like `loadbalancer.example.com` → load balancer
* New subdomain appearing (e.g. `vpn.`) → new entry point
* TXT `_1password=...` → org uses 1Password → phishing/social engineering angle

## Digging DNS

Tools: `dig` (most versatile), `nslookup` / `host` (quick checks), `dnsenum` / `fierce` / `dnsrecon` (automated enumeration), `theHarvester` (emails + DNS from many sources), online lookup services.

```bash
dig <TARGET_DOMAIN> A          # default lookup is A anyway
dig <TARGET_DOMAIN> MX|NS|TXT|CNAME|SOA
dig @1.1.1.1 <TARGET_DOMAIN>   # query a specific name server
dig +trace <TARGET_DOMAIN>     # full resolution path
dig -x 192.168.1.1             # reverse lookup
dig +short <TARGET_DOMAIN>     # just the answer
dig +noall +answer <TARGET_DOMAIN>
dig <TARGET_DOMAIN> ANY        # often ignored per RFC 8482
```

Output anatomy: header (status, flags qr/rd/ad, section counts), question, answer (name, TTL, record), footer (server, timing). Some servers detect/block excessive queries — respect rate limits.

## Subdomains

Subdomains often host what the main site doesn't link to: dev/staging environments (relaxed security), hidden login portals/admin panels, legacy applications (outdated software), accidentally exposed documents and configs.

Two enumeration approaches:

1. **Active** — direct queries against the target's DNS: zone transfer attempts (rarely work), brute-forcing wordlists against the domain (dnsenum, ffuf, gobuster).
2. **Passive** — external sources only: Certificate Transparency logs (subdomains in the SAN field), search engines (`site:` operator), DNS aggregation databases.

Combine both — active is comprehensive but detectable, passive is stealthier but incomplete.

## Subdomain Bruteforcing

Process: pick a wordlist (general-purpose, targeted at the industry/tech, or custom from gathered intel) → iterate `word.domain` → A/AAAA lookup for each → validate what resolves.

Tools: dnsenum, fierce, dnsrecon, amass (extensive data sources), assetfinder (quick/lightweight), puredns (powerful resolving + filtering).

```bash
dnsenum --enum <TARGET_DOMAIN> -f subdomains-top1million-20000.txt -r
```

`--enum` = tuning shortcut, `-f` = wordlist, `-r` = recursive brute-forcing (found subdomain → enumerate its subdomains). dnsenum also tries zone transfers, scrapes Google, does reverse lookups and WHOIS on its own.

## DNS Zone Transfers

AXFR = full copy of a zone between name servers. If access controls are misconfigured, anyone can pull the whole zone file: complete subdomain list, their IPs, NS records — a full map of DNS infrastructure.

```bash
dig axfr @<NS> <TARGET_DOMAIN>
dig axfr @nsztm1.digi.ninja zonetransfer.me    # public demo target that allows it
```

Mostly fixed in modern setups (transfers restricted to trusted secondaries), but the attempt costs nothing and even a refusal tells you something about the server's configuration.

## Virtual Hosts

Subdomain = DNS record. Virtual host = web server (Apache/Nginx/IIS) config serving many sites on one IP, selected by the HTTP Host header — the header acts as a switch into the matching DocumentRoot. VHosts can exist without any DNS record: map the name in `/etc/hosts` and the server will still serve it.

That's why VHost fuzzing matters: it finds public AND non-public subdomains/vhosts by throwing hostnames at a known IP. VHosts aren't limited to subdomains — entirely different domains can share the server.

Hosting types:

* **Name-based:** Host header, most common, cheap
* **IP-based:** one IP per site, better isolation
* **Port-based:** different ports on one IP

```bash
gobuster vhost -u http://<TARGET_IP> -w <wordlist_file> --append-domain
```

`--append-domain` is required in newer gobuster versions (builds full hostnames from the wordlist). Useful extras: `-t` threads, `-k` ignore TLS cert errors, `-o` save output. Example hit: `forum.<TARGET_DOMAIN>` Status 200. VHost scanning is loud — IDS/WAF may notice.

## Certificate Transparency Logs

CT logs are public, append-only ledgers of every issued SSL/TLS cert. Every CA must submit to multiple logs; anyone can inspect them. For recon: certs list subdomains in their SAN field — a definitive, historical record no wordlist can match, including subdomains behind old/expired certs that may still run outdated software.

Search options: crt.sh (free web UI + API, simple) and Censys (advanced filtering, API, free registration).

```bash
curl -s "https://crt.sh/?q=<TARGET_DOMAIN>&output=json" | jq -r '.[]
 | select(.name_value | contains("dev")) | .name_value' | sort -u
```

## Fingerprinting

Identify the tech stack so exploits can be targeted and misconfigurations spotted. Techniques: banner grabbing, HTTP headers (`Server`, `X-Powered-By`), crafted probe responses, page content/scripts. Tools: Wappalyzer (browser extension), BuiltWith, WhatWeb, Nmap, Netcraft, wafw00f (WAF detection).

Banner chasing through redirects — each hop adds info:

```bash
curl -I http://<TARGET_DOMAIN>       # Server: Apache/2.4.41 (Ubuntu), redirect to https
curl -I https://<TARGET_DOMAIN>      # X-Redirect-By: WordPress
curl -I https://www.<TARGET_DOMAIN>  # Link: .../wp-json/... → WordPress REST API exposed
```

Check for a WAF before fuzzing anything:

```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
wafw00f <TARGET_DOMAIN>              # e.g. "behind Wordfence (Defiant) WAF"
```

Nikto, software-identification modules only:

```bash
nikto -h <TARGET_DOMAIN> -Tuning b
```

Example findings: Apache 2.4.41 outdated, WordPress installation + `/wp-login.php`, `/license.txt` (software identification), missing HSTS / X-Content-Type-Options, `Content-Encoding: deflate` → potential BREACH.

## Crawling

Automated browsing: fetch seed URL → parse → queue links → repeat.

* **Breadth-first:** map the whole width, good overview
* **Depth-first:** follow one path deep, good for specific content

Crawling follows real links — unlike fuzzing, which guesses them.

Extract: internal/external links, HTML comments, metadata, and sensitive files (`.bak`, `.old`, `web.config`, `settings.php`, `error_log`, `access_log` — credentials, API keys, source snippets).

Context is where the value is: several links pointing into `/files/` → check it manually → directory listing enabled with backup archives and internal documents. A comment mentioning a "file server" plus that `/files/` find = exposed file server. Connect the dots instead of reading each item alone.

## robots.txt

`/robots.txt` in the site root, per the Robots Exclusion Standard. Records of `User-agent:` (`*` = all bots) plus directives:

```
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://www.example.com/sitemap.xml
```

For recon: Disallow entries are a free list of directories the owner wants hidden — admin panels, private content, backups. Allowed/disallowed paths sketch the site structure beyond the visible navigation. Watch for honeypot directories planted to trap crawlers.

## Well-Known URIs

RFC 8615 standardizes `/.well-known/` as the place for a site's metadata and service configs; IANA maintains the registry.

Notable entries:

* `security.txt` (vuln reporting contacts, RFC 9116)
* `change-password`
* `openid-configuration`
* `assetlinks.json`
* `mta-sts.txt`

`/.well-known/openid-configuration` (OpenID Connect Discovery) returns JSON worth grabbing:

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "jwks_uri": "https://example.com/oauth2/jwks",
  "response_types_supported": ["code", "token", "id_token"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

Gives auth/token/userinfo endpoints, the JWKS key set location, supported scopes, response types and signing algorithms — a full map of the IdP implementation.

## Creepy Crawlies

Crawlers: Burp Suite Spider, OWASP ZAP (spider + scanner), Scrapy (Python framework for custom spiders), Apache Nutch (large-scale, Java).

ReconSpider — HTB's custom Scrapy spider:

```bash
pip3 install scrapy
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
python3 ReconSpider.py http://<TARGET_DOMAIN>
```

Output lands in `results.json`, keyed by data type:

* `emails`
* `links`
* `external_files` (PDFs etc.)
* `js_files`
* `form_fields`
* `images`
* `videos`
* `audio`
* `comments` (HTML comments from source)

## Search Engine Discovery

Using search engines as OSINT tools.

Core operators:

```
site:example.com                only this domain
inurl:login / allinurl:         term(s) in the URL
intitle:"confidential report" / allintitle:   term(s) in the title
intext:"password reset" / allintext:          term(s) in the body
filetype:pdf                    file type
cache:example.com               cached version
link: / related: / info:        backlinks, similar sites, page summary
numrange:1000-2000              numbers in a range (also 100..500)
"exact phrase"
AND / OR / NOT                  boolean
* wildcard, -excluded_term
```

Google Dorking = combining operators to surface sensitive content. Standard patterns (more in the Google Hacking Database):

```
site:example.com (inurl:login OR inurl:admin)             # login pages
site:example.com (filetype:xls OR filetype:docx)          # exposed documents
site:example.com inurl:config.php                          # config files
site:example.com (ext:conf OR ext:cnf)                     # config extensions
site:example.com inurl:backup
site:example.com filetype:sql                              # database backups
```

## Web Archives

The Wayback Machine (Internet Archive, archiving since 1996) stores dated snapshots of full page content.

Recon uses:

* Old pages, directories, files and subdomains no longer live but possibly still reachable or exposing sensitive info/flaws
* Diffing snapshots over time → structural, content and technology changes
* OSINT on past activities, employees, tech choices
* Fully passive — no touch on the target's infrastructure

Usage: enter the URL, pick a capture date.

## Automating Recon

Frameworks: FinalRecon (modular all-in-one), Recon-ng (module framework), theHarvester (emails, subdomains, hosts, names, ports, banners from public sources), SpiderFoot (OSINT automation across many data sources), OSINT Framework (tool/resource collection).

FinalRecon:

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git && cd FinalRecon
pip3 install -r requirements.txt
chmod +x finalrecon.py

finalrecon.py --headers --whois --url http://<TARGET_DOMAIN>
```

Modules:

* `--headers`
* `--sslinfo`
* `--whois`
* `--crawl` (HTML/CSS/JS, links, images, robots.txt, sitemap.xml, JS links, Wayback)
* `--dns` (40+ record types incl. DMARC)
* `--sub` (crt.sh, AnubisDB, ThreatMiner, CertSpotter, Facebook/VirusTotal/Shodan/BeVigil APIs)
* `--dir` (custom wordlists + extensions)
* `--wayback` (last 5 years of URLs)
* `--ps` (fast port scan)
* `--full` (everything)

Extras: `-w` wordlist, `-e` extensions, `-d` custom DNS, `-T` timeout, `-o` export format. Results auto-export to `~/.local/share/finalrecon/dumps/`.
