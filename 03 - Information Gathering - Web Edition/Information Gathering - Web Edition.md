# Web Reconnaissance

Passive = no direct target interaction (OSINT, CT logs, search engines, Wayback). Active = direct interaction (DNS queries, port scans, crawling). Both needed for full coverage.

---

## WHOIS

```bash
whois <DOMAIN>
whois <IP>
```

Key fields: Registrar, Registrant/Admin/Tech contact, Creation/Expiry dates, Name Servers, Domain Status. Registrant contact → social engineering targets, employee names, emails. NS records → hosting provider. Recent registration + privacy-protected registrant + bulletproof hosting = phishing indicator.

Historical WHOIS: https://whoisfreaks.com — tracks ownership changes over time.

---

## DNS — dig

```bash
dig <DOMAIN>                          # default A record
dig <DOMAIN> A                        # IPv4
dig <DOMAIN> AAAA                     # IPv6
dig <DOMAIN> MX                       # mail servers
dig <DOMAIN> NS                       # nameservers
dig <DOMAIN> TXT                      # SPF, DMARC, verif tokens
dig <DOMAIN> CNAME                    # aliases
dig <DOMAIN> SOA                      # zone admin info
dig <DOMAIN> ANY                      # all available (many servers block this)
dig @<DNS_SERVER> <DOMAIN>            # query specific nameserver
dig +trace <DOMAIN>                   # full resolution path
dig -x <IP>                           # reverse lookup
dig +short <DOMAIN>                   # IP only, no noise
dig +noall +answer <DOMAIN>           # answer section only
dig axfr <DOMAIN> @<DNS_SERVER>       # zone transfer attempt
dig axfr internal.<DOMAIN> @<DNS_SERVER>  # internal zone
```

SOA email: dot replaced with @. SPF TXT record → leaks third-party providers (Mailgun, Atlassian, Google, Outlook, etc.).

---

## Subdomain Enumeration

**Passive — CT logs (crt.sh):**

```bash
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq .
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u

# Filter for specific keyword (e.g. dev subdomains)
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u
```

**Active — brute force:**

```bash
dnsenum --enum <DOMAIN> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
dnsenum --dnsserver <DNS_SERVER> --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt <DOMAIN>

# Manual dig loop
for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do dig $sub.<DOMAIN> @<DNS_SERVER> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt; done
```

Wordlists: `/opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt`, `subdomains-top1million-110000.txt`.

---

## Zone Transfer

Misconfigured `allow-transfer any` leaks full zone: all subdomains, IPs, internal hostnames (DCs, VPNs, WSUS, workstations).

```bash
dig axfr <DOMAIN> @<NS_SERVER>
dig axfr internal.<DOMAIN> @<NS_SERVER>
```

Test target: `dig axfr @nsztm1.digi.ninja zonetransfer.me` (intentionally vulnerable).

---

## Vhost Fuzzing (non-public subdomains)

Fuzzes the `Host:` header to find vhosts that don't have public DNS records. First run without `-fs` to get default response size, then filter it.

ffuf (from prior note):

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://<TARGET>:<PORT>/ -H 'Host: FUZZ.<DOMAIN>' -fs <DEFAULT_SIZE>
```

gobuster:

```bash
gobuster vhost -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain
gobuster vhost -u http://<TARGET_IP> -w <WORDLIST> --append-domain -t 50 -k -o vhosts.txt
```

`--append-domain` required in newer gobuster versions. `-k` = ignore TLS errors. Add discovered vhosts to `/etc/hosts`.

---

## Fingerprinting

Banner grab — check server version, redirect chain, CMS headers:

```bash
curl -I http://<TARGET>
curl -I https://<TARGET>
curl -I https://www.<TARGET>
```

Look for: `Server:` (Apache/nginx/IIS + version), `X-Powered-By:` (PHP/ASP.NET), `X-Redirect-By:` (WordPress), `Link:` headers with `wp-json` = WordPress.

WAF detection:

```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
wafw00f <DOMAIN>
```

Nikto fingerprinting only:

```bash
nikto -h <TARGET> -Tuning b
```

`-Tuning b` = software identification modules only. Reveals: server version, WordPress presence, `/wp-login.php`, `/license.txt`, missing security headers.

Other tools: Wappalyzer (browser extension), WhatWeb (`whatweb <DOMAIN>`), BuiltWith.

---

## robots.txt & Well-Known URIs

```bash
curl https://<TARGET>/robots.txt
curl https://<TARGET>/.well-known/security.txt
curl https://<TARGET>/.well-known/openid-configuration    # OAuth/OIDC endpoints
curl https://<TARGET>/.well-known/mta-sts.txt
```

`robots.txt` Disallow entries = directories owner wants hidden (admin panels, private dirs, backup locations, legacy apps). `Allow:` entries confirm accessible paths. `Sitemap:` → full site structure.

`openid-configuration` returns JSON with: authorization_endpoint, token_endpoint, userinfo_endpoint, jwks_uri, supported scopes — maps authentication infrastructure.

IANA .well-known registry: https://www.iana.org/assignments/well-known-uris/

---

## Web Crawling

ReconSpider (Scrapy-based):

```bash
pip3 install scrapy
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
python3 ReconSpider.py http://<TARGET>
```

Output `results.json` contains: emails, links (internal+external), external_files, js_files, form_fields, images, videos, audio, HTML comments.

Comments in source often leak: software versions, internal paths, developer notes, disabled functionality.

---

## Google Dorks

```
site:<DOMAIN>                                    # all indexed pages
site:<DOMAIN> inurl:admin
site:<DOMAIN> inurl:login
site:<DOMAIN> inurl:config.php
site:<DOMAIN> (inurl:login OR inurl:admin)
site:<DOMAIN> filetype:pdf
site:<DOMAIN> (filetype:xls OR filetype:docx)
site:<DOMAIN> (ext:conf OR ext:cnf)
site:<DOMAIN> inurl:backup
site:<DOMAIN> filetype:sql
intitle:"confidential report"
intext:"password reset"
inurl:login inurl:<DOMAIN>
"<DOMAIN>" inurl:amazonaws.com                   # S3 buckets
"<DOMAIN>" inurl:blob.core.windows.net           # Azure blobs
```

Google Hacking Database (GHDB): https://www.exploit-db.com/google-hacking-database

---

## Wayback Machine

https://web.archive.org

Useful for: old subdomains, removed pages, legacy admin panels, exposed credentials in old configs, tracking technology changes over time. Passive — no target interaction.

```bash
# Via API
curl "https://web.archive.org/cdx/search/cdx?url=<DOMAIN>/*&output=text&fl=original&collapse=urlkey"
```

FinalRecon also pulls Wayback data: `./finalrecon.py --wayback --url http://<TARGET>`.

---

## Automated Recon — FinalRecon

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py

./finalrecon.py --headers --url http://<TARGET>
./finalrecon.py --whois --url http://<TARGET>
./finalrecon.py --sslinfo --url http://<TARGET>
./finalrecon.py --dns --url http://<TARGET>
./finalrecon.py --sub --url http://<TARGET>
./finalrecon.py --crawl --url http://<TARGET>
./finalrecon.py --wayback --url http://<TARGET>
./finalrecon.py --dir -w <WORDLIST> --url http://<TARGET>
./finalrecon.py --full --url http://<TARGET>
```

Output saved to `~/.local/share/finalrecon/dumps/`. Add API keys with `-k shodan@<KEY>` for extended subdomain sources.

Other frameworks: Recon-ng, theHarvester, SpiderFoot, amass, OSINT Framework.

> [!tip] Reconnaissance workflow order: WHOIS → DNS (dig any + axfr attempts) → CT logs (crt.sh) → subdomain brute (dnsenum) → vhost fuzz (ffuf/gobuster) → fingerprint (curl -I + wafw00f + nikto) → crawl (robots.txt + well-known + ReconSpider) → Google dorks + Wayback for historical intel.