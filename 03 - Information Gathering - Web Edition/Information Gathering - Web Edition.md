# Web Reconnaissance

## Introduction

Systematic collection of information about a target website, part of the Information Gathering phase.  
Goals: identify assets, find hidden information, analyse the attack surface, gather intelligence.

### Types of Reconnaissance

#### Active Reconnaissance

Direct interaction with the target. Higher risk of detection.

`Port scanning` Nmap, Masscan, Unicornscan, high risk | `Vulnerability scanning` Nessus, OpenVAS, Nikto, high risk | `Network mapping` traceroute, Nmap, medium to high risk | `Banner grabbing` Netcat, curl, low risk | `OS fingerprinting` Nmap -O, Xprobe2, low risk | `Service enumeration` Nmap -sV, low risk | `Web spidering` Burp Suite Spider, OWASP ZAP Spider, Scrapy, low to medium risk

#### Passive Reconnaissance

Uses public sources without touching the target. Very low detection risk, may yield less information.

`Search engine queries` Google, DuckDuckGo, Bing, Shodan | `WHOIS lookups` whois, online WHOIS services | `DNS analysis` dig, nslookup, host, dnsenum, fierce, dnsrecon | `Web archive analysis` Wayback Machine | `Social media analysis` LinkedIn, Twitter, Facebook, OSINT tools | `Code repositories` GitHub, GitLab

## WHOIS

Query and response protocol for databases of registered internet resources, mainly domains but also IP blocks and autonomous systems.

```bash
whois <TARGET_DOMAIN>
```

`Registrar` company where the domain was registered | `Registrant Contact` person or organisation that registered it | `Administrative Contact` manages the domain | `Technical Contact` handles technical issues | `Creation and Expiration Dates` registration and expiry | `Name Servers` servers that resolve the domain

### Why WHOIS Matters for Web Recon

`Identifying Key Personnel` names, emails, phones, useful for social engineering and phishing | `Discovering Network Infrastructure` name servers and IPs show entry points or misconfigurations | `Historical Data Analysis` past records show ownership, contact, and technical changes

WhoisFreaks: https://whoisfreaks.com/

## Utilising WHOIS

WHOIS data helps spot malicious domains and trace infrastructure.

### Scenario 1: Phishing Investigation

Red flags: domain registered days ago, registrant hidden behind a privacy service, name servers of a bulletproof hosting provider. Together they suggest phishing.

### Scenario 2: Malware Analysis

Indicators of a C2 domain on a compromised or bulletproof server: registrant uses a free anonymous email service, registrant address in a country with high cybercrime, registrar with lax abuse policies.

### Scenario 3: Threat Intelligence Report

Patterns across a threat actor's domains: registered in clusters before attacks, aliases and fake registrant identities, shared name servers, many domains taken down after attacks. These feed IOCs.

### Using WHOIS

```bash
sudo apt update
sudo apt install whois -y
```

`Creation Date` and `Registry Expiry Date` long history and distant expiry suggest an established domain | `Registrant Organization` identifies the owner | `Domain Status` client or server Delete, Transfer, Update Prohibited means protected against changes | `Name Server` inside the target domain means the owner runs its own DNS

WHOIS rarely identifies employees or vulnerabilities. Combine it with other recon.

## DNS

Translates human-readable domain names into numerical IP addresses.

### How DNS Works

Lookup order: local cache, DNS resolver (ISP), root name server, TLD name server, authoritative name server. The resolver caches the answer.

### The Hosts File

Local text file mapping hostnames to IP addresses. Bypasses DNS and changes apply immediately.

`C:\Windows\System32\drivers\etc\hosts` Windows | `/etc/hosts` Linux and MacOS

```text
<TARGET_IP>    <HOSTNAME> [<ALIAS> ...]
```

Editing needs admin or root privileges. Mapping a domain to `0.0.0.0` blocks it.

### Key DNS Concepts

Zone: part of the domain namespace managed by one entity, such as a domain and its subdomains. Zone file: text file on a DNS server defining the zone's resource records.

`DNS Resolver` translates names to IPs, ISP or public such as 8.8.8.8 | `Root Name Server` top of the hierarchy, 13 servers named A-M | `TLD Name Server` responsible for one TLD such as .com | `Authoritative Name Server` holds the actual IP of a domain

`A` hostname to IPv4 | `AAAA` hostname to IPv6 | `CNAME` alias pointing to another hostname | `MX` mail servers for the domain | `NS` delegates the zone to an authoritative name server | `TXT` arbitrary text, domain verification, SPF | `SOA` zone admin info, primary name server, admin email | `SRV` hostname and port of a service | `PTR` reverse lookup, IP to hostname

`IN` in a record is the class field, meaning Internet.

### Why DNS Matters for Web Recon

`CNAME` pointing to an outdated server may lead to a vulnerable system | `NS` records reveal the hosting provider | `A` record such as loadbalancer pinpoints a load balancer | `New subdomain` such as vpn may be a new entry point | `TXT` value `_1password=...` shows the organisation uses 1Password, useful for social engineering and phishing

## Digging DNS

Tools and techniques for using DNS in web reconnaissance.

### DNS Tools

`dig` flexible lookups, zone transfers, in-depth record analysis | `nslookup` simple A, AAAA, MX lookups | `host` concise A, AAAA, MX lookups | `dnsenum` automated enumeration, dictionary attacks, brute-forcing, zone transfers | `fierce` recursive subdomain search, wildcard detection | `dnsrecon` combines multiple DNS techniques, many output formats | `theHarvester` OSINT, collects emails and employee info from many sources | Online DNS Lookup Services

### The Domain Information Groper

dig queries DNS servers for any record type with detailed, customisable output.

#### Common dig Commands

```bash
dig <TARGET_DOMAIN> # default A lookup
dig <TARGET_DOMAIN> A # IPv4
dig <TARGET_DOMAIN> AAAA # IPv6
dig <TARGET_DOMAIN> MX # mail servers
dig <TARGET_DOMAIN> NS # authoritative name servers
dig <TARGET_DOMAIN> TXT
dig <TARGET_DOMAIN> CNAME
dig <TARGET_DOMAIN> SOA
dig @1.1.1.1 <TARGET_DOMAIN> # specific name server
dig +trace <TARGET_DOMAIN> # full resolution path
dig -x <TARGET_IP> # reverse lookup
dig +short <TARGET_DOMAIN> # short answer only
dig +noall +answer <TARGET_DOMAIN> # answer section only
dig <TARGET_DOMAIN> ANY # all records
```

Many servers ignore ANY queries (RFC 8482). `-x` may need a name server specified. Some servers block excessive queries.

#### Groping DNS

`status: NOERROR` successful query | `qr` response | `rd` recursion desired | `ad` resolver considers data authentic | `ANSWER SECTION` the record, TTL is how long it can be cached | `SERVER` server and protocol that answered | `OPT PSEUDOSECTION` from EDNS, larger messages and DNSSEC support | `recursion requested but not available` server does not support recursion

## Subdomains

Extensions of the main domain that often host resources not linked from the main site.

### Why is this important for web reconnaissance?

`Development and Staging Environments` relaxed security, may hold vulnerabilities or sensitive data | `Hidden Login Portals` admin panels not meant to be public | `Legacy Applications` forgotten apps with outdated, vulnerable software | `Sensitive Information` confidential documents, internal data, configuration files

### Subdomain Enumeration

Subdomains appear as `A` or `AAAA` records, and `CNAME` records can alias them.

Active gives more control and coverage but is more detectable. Passive is stealthier but may miss subdomains. Combine both.

#### Active Subdomain Enumeration

Interacts directly with the target's DNS servers. Zone transfer can leak all subdomains but rarely works. Brute-force tests a wordlist of names with `dnsenum`, `ffuf`, `gobuster`.

#### Passive Subdomain Enumeration

Uses external sources without querying the target's DNS. Certificate Transparency logs list subdomains in the SAN field of certificates. Search engines with `site:` filter subdomains. Online databases aggregate DNS data.

## Subdomain Bruteforcing

Active discovery that tests a wordlist of names against the target domain.

`General-purpose` common names such as dev, staging, blog, mail, admin, test, use when naming is unknown | `Targeted` industry or technology specific, more efficient, fewer false positives | `Custom` built from keywords, patterns, or gathered intel

Each word is appended to the domain and resolved via A or AAAA. Resolving names are kept and validated.

`dnsenum` dictionary and brute-force attacks | `fierce` recursive discovery, wildcard detection | `dnsrecon` combines multiple DNS techniques, custom output | `amass` actively maintained, many data sources | `assetfinder` quick and light scans | `puredns` brute-forcing, resolves and filters results

### DNSEnum

Perl tool. Enumerates A, AAAA, NS, MX, TXT records, tries zone transfers automatically, brute-forces subdomains, scrapes Google, does reverse lookups and WHOIS. Wordlist from SecLists with the top 20000 subdomains.

```bash
dnsenum --enum <TARGET_DOMAIN> -f subdomains-top1million-20000.txt -r
```

`--enum` shortcut for some tuning options | `-f` wordlist | `-r` recursive brute-forcing of found subdomains

## DNS Zone Transfers

Full copy of a zone's DNS records from one name server to another. A misconfigured server leaks all subdomains.

### What is a Zone Transfer

Flow: AXFR request from secondary, SOA record from primary, all records sent, end of transfer, ACK.

### The Zone Transfer Vulnerability

If any client may request a transfer, the whole zone file is exposed. Reveals all subdomains including hidden dev, staging, and admin hosts, their IPs, and name server records.

### Remediation

Servers usually allow transfers only to trusted secondaries. Misconfigurations still occur, so try it. A failed attempt can still reveal server configuration.

### Exploiting Zone Transfers

```bash
dig axfr @<NAME_SERVER> <TARGET_DOMAIN>
```

Returns all DNS records of the zone if the server allows the transfer.

## Virtual Hosts

Web server feature hosting several sites on one IP, told apart by the HTTP Host header.

### How Virtual Hosts Work: Understanding VHosts and Subdomains

Subdomains have DNS records. VHosts are web server configurations, for example Apache `<VirtualHost>` with `ServerName` and `DocumentRoot`. A vhost without a DNS record can be reached by adding it to the local hosts file. Non-public subdomains may not appear in DNS. VHost fuzzing finds public and non-public subdomains and vhosts by testing hostnames against a known IP.

### Server VHost Lookup

The browser sends the domain in the Host header. The server matches it to a vhost config and serves files from that document root.

### Types of Virtual Hosting

`Name-based` Host header only, most common, limits with SSL/TLS | `IP-based` unique IP per site, no Host header needed | `Port-based` sites on different ports of one IP, such as 80 and 8080

### Virtual Host Discovery Tools

`gobuster` directory brute-forcing, also vhost discovery | `Feroxbuster` Rust, fast, recursion, wildcard discovery, filters | `ffuf` fuzzes the Host header, custom wordlists and filters

### gobuster

Needs the target IP and a wordlist such as SecLists or a custom list.

```bash
gobuster vhost -u http://<TARGET_IP> -w <WORDLIST> --append-domain
gobuster vhost -u http://<TARGET_DOMAIN>:<PORT> -w subdomains-top1million-110000.txt --append-domain
```

`-u` target URL | `-w` wordlist | `--append-domain` appends base domain to each word | `-t` more threads | `-k` ignore SSL/TLS errors | `-o` save output to file

Older Gobuster versions have no `--append-domain` and append the domain by default. Heavy traffic can be caught by IDS or WAF.

## Certificate Transparency Logs

Public, append-only ledgers of issued SSL/TLS certificates. Passive source for subdomains.

### What are Certificate Transparency Logs?

CAs must submit every new certificate to multiple CT logs, open for anyone to inspect.

### CT Logs and Web Recon

Gives a historical record of certificates for a domain and its subdomains, no wordlist needed. Also exposes subdomains of old or expired certificates that may run outdated software.

### Searching CT Logs

`crt.sh` free web interface, no registration, limited filtering | `Censys` advanced filtering by domain, IP, certificate attributes, API access, free registration

### crt.sh lookup

```bash
curl -s "https://crt.sh/?q=<TARGET_DOMAIN>&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u # dev subdomains
```

## Fingerprinting

Extracts technical details about the technologies behind a site, such as web server, OS, and software.

### Fingerprinting Techniques

`Banner Grabbing` banners show server software and version | `Analysing HTTP Headers` `Server` shows the web server, `X-Powered-By` shows languages or frameworks | `Probing for Specific Responses` crafted requests and error messages reveal technology | `Analysing Page Content` structure, scripts, copyright header

`Wappalyzer` browser extension, technology profiling | `BuiltWith` technology stack reports | `WhatWeb` command-line, signature database | `Nmap` NSE scripts for service and OS fingerprinting | `Netcraft` hosting and security reports | `wafw00f` identifies WAFs

### Fingerprinting `<TARGET_DOMAIN>`

#### Banner Grabbing

```bash
curl -I <TARGET_DOMAIN> # headers only
curl -I https://<TARGET_DOMAIN> # grab each redirect target
curl -I https://www.<TARGET_DOMAIN>
```

`Server` web server software and version | `X-Redirect-By` shows what redirects, such as WordPress | `Link` header with `wp-json` path indicates WordPress

#### Wafw00f

Check for a WAF first, it can block or filter probes.

```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
wafw00f <TARGET_DOMAIN>
```

#### Nikto

```bash
sudo apt update && sudo apt install -y perl
git clone https://github.com/sullo/nikto
chmod +x ./nikto.pl # run inside nikto/program
nikto -h <TARGET_DOMAIN> -Tuning b
```

`-h` target host | `-Tuning b` only Software Identification modules

Findings to read: WordPress install and login (`/wp-login.php`), `/license.txt` exposing software details, outdated server version, missing security headers, uncommon headers such as `x-redirect-by`.

## Crawling

Automated browsing that follows links from a seed URL to map a site and collect data.

### How Web Crawlers Work

Starts at a seed URL, extracts links, queues them, repeats. Crawling follows links, fuzzing guesses them.

### Breadth-First Crawling

Crawls all links on a page before going deeper. Gives a broad overview of structure and content.

### Depth-First Crawling

Follows one path as far as possible before backtracking. Good for specific or deep content.

### Extracting Valuable Information

`Links` internal and external, map structure and find hidden pages | `Comments` can leak sensitive details or hints | `Metadata` titles, descriptions, keywords, authors, dates | `Sensitive Files` backups `.bak` `.old`, configs `web.config` `settings.php`, logs `error_log` `access_log`, may hold credentials, API keys, source

### The Importance of Context

Correlate findings. Repeated `/files/` links led to a directory with browsing enabled exposing backups and internal documents, and a comment about a file server supported it.

## robots.txt

Text file in the web root following the Robots Exclusion Standard, telling bots which paths to crawl or avoid.

### What is robots.txt?

Located at `<TARGET_DOMAIN>/robots.txt`.

### How robots.txt Works

Directives target user-agents. `*` is a wildcard for all bots.

### Understanding robots.txt Structure

`User-agent` bot the rules apply to, `*` or a name such as `Googlebot`, `Bingbot` | `Disallow` paths not to crawl | `Allow` permitted paths inside a broader Disallow | `Crawl-delay` seconds between requests | `Sitemap` URL of the XML sitemap

### robots.txt in Web Reconnaissance

`Disallowed paths` point to hidden directories or files such as admin panels and backups | `Allowed and disallowed paths` map the structure and reveal pages not in navigation | `Honeypot directories` can be crawler traps

### Analyzing robots.txt

`Disallow` entries such as `/admin/` and `/private/` suggest an admin panel and private content.

## Well-Known URIs

Standard `/.well-known/` path in the web root holding site metadata and configuration files (RFC 8615). IANA keeps a registry of these URIs.

`security.txt` contact for security researchers, RFC 9116 | `change-password` standard password change page | `openid-configuration` OpenID Connect configuration | `assetlinks.json` verifies ownership of digital assets such as apps | `mta-sts.txt` SMTP MTA-STS policy, RFC 8461

change-password: https://w3c.github.io/webappsec-change-password-url/#the-change-password-well-known-uri  
openid-configuration: http://openid.net/specs/openid-connect-discovery-1_0.html  
assetlinks.json: https://github.com/google/digitalassetlinks/blob/master/well-known/specification.md

### Web Recon and .well-known

Endpoint: `https://<TARGET_DOMAIN>/.well-known/openid-configuration`. Returns JSON metadata about the OpenID Connect provider.

`authorization_endpoint` user authorization requests | `token_endpoint` where tokens are issued | `userinfo_endpoint` user information | `jwks_uri` JSON Web Key Set, the cryptographic keys | `scopes_supported` and `response_types_supported` map functionality and limits | `id_token_signing_alg_values_supported` signing algorithms

## Creepy Crawlies

Tools that automate crawling.

### Popular Web Crawlers

`Burp Suite Spider` active crawler in Burp Suite | `OWASP ZAP` free, open-source scanner with a spider | `Scrapy` Python framework for custom crawlers | `Apache Nutch` scalable Java crawler for large crawls

### Scrapy

#### Installing Scrapy

```bash
pip3 install scrapy
```

#### ReconSpider

Custom Scrapy spider. Output is saved to `results.json`.

```bash
wget -O ReconSpider.zip https://cdn.services-k8s.prod.aws.htb.systems/content/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
python3 ReconSpider.py http://<TARGET_DOMAIN>
```

#### results.json

`emails` addresses found | `links` URLs of links | `external_files` files such as PDFs | `js_files` JavaScript files | `form_fields` form fields | `images` | `videos` | `audio` | `comments` HTML comments in source

## Search Engine Discovery

Using search engines and operators to find information about a target (OSINT).

### Search Operators

Syntax varies slightly between search engines.

`site:` limit to a domain | `inurl:` term in URL | `filetype:` file type | `intitle:` term in title | `intext:` or `inbody:` term in body | `cache:` cached version of a page | `link:` pages linking to a page | `related:` similar sites | `info:` page summary | `define:` definitions | `numrange:` numbers in a range | `allintext:` all words in body | `allinurl:` all words in URL | `allintitle:` all words in title | `AND` require all terms | `OR` any of the terms | `NOT` exclude a term | `*` wildcard for any word | `..` numeric range | `" "` exact phrase | `-` exclude a term

Examples of combining operators, wildcard, range, and exclusion.

```text
site:<TARGET_DOMAIN> numrange:1000-2000
site:<TARGET_DOMAIN> AND (inurl:admin OR inurl:login)
"linux" OR "ubuntu" OR "debian"
site:<TARGET_DOMAIN> NOT inurl:login
site:<TARGET_DOMAIN> filetype:pdf user* manual
site:<TARGET_DOMAIN> "price" 100..500
site:<TARGET_DOMAIN> -inurl:sports
```

### Google Dorking

Search operators used to find sensitive information and hidden content. Dorks for login pages, exposed files, configuration files, database backups. More in the Google Hacking Database.

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

## Web Archives

Wayback Machine snapshots of past site versions, a passive recon source.

### What is the Wayback Machine?

Digital archive of the web run by the non-profit Internet Archive since 1996. Snapshots are called captures or archives.

### How Does the Wayback Machine Work?

Stores HTML, CSS, JavaScript, and images of each capture. Browse by URL and date, search terms in archived content, or download archived sites for offline analysis. Not every page is archived and owners can request exclusion.

### Why the Wayback Machine Matters for Web Reconnaissance

`Hidden assets and vulnerabilities` old pages, directories, files, subdomains no longer on the current site | `Tracking changes` compare snapshots for changes in structure, content, technologies | `Gathering intelligence` OSINT on past activities, employees, technology choices | `Stealthy reconnaissance` passive, does not touch the target

### Going Wayback on HTB

Enter the page URL, then select a capture date to view the snapshot.

## Automating Recon

Automating recon tasks for speed and consistent coverage.

### Reconnaissance Frameworks

`FinalRecon` Python tool with modules for SSL certificate checks, Whois, header analysis, crawling | `Recon-ng` modular Python framework for DNS enumeration, subdomain discovery, port scanning, web crawling | `theHarvester` collects emails, subdomains, hosts, employee names, open ports, banners from search engines, PGP key servers, SHODAN | `SpiderFoot` OSINT automation integrating many data sources | `OSINT Framework` collection of OSINT tools and resources

### FinalRecon

Covers headers, Whois, SSL certificate, crawler (HTML, CSS, JavaScript, links, images, robots.txt, sitemap.xml, JavaScript links, Wayback), DNS enumeration (40+ record types incl. DMARC), subdomain enumeration (crt.sh, AnubisDB, ThreatMiner, CertSpotter, Facebook API, VirusTotal API, Shodan API, BeVigil API), directory enumeration with custom wordlists and extensions, Wayback URLs of the last five years.

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
pip3 install -r requirements.txt # inside FinalRecon directory
chmod +x ./finalrecon.py
./finalrecon.py --headers --whois --url http://<TARGET_DOMAIN>
```

`--url` target URL | `--headers` header info | `--sslinfo` SSL certificate info | `--whois` Whois lookup | `--crawl` crawl target | `--dns` DNS enumeration | `--sub` subdomain enumeration | `--dir` directory search | `--wayback` Wayback URLs | `--ps` fast port scan | `--full` full recon