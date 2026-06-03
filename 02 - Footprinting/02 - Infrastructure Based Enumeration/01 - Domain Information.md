# Domain Information — Passive Recon

All of this is passive — no direct contact with the target. Stay in customer/visitor mode. Third-party providers (AWS, Cloudflare, Google) are out of scope without their explicit written permission.

---

### Certificate Transparency — crt.sh

Pull all subdomains from CT logs, deduplicated:

```bash
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

---

### Resolve Subdomains → IP List

Filter to company-owned IPs only — third-party hostnames (amazonaws.com etc.) drop out naturally:

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep <DOMAIN> | cut -d" " -f1,4; done
```

Collect IPs for Shodan:

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep <DOMAIN> | cut -d" " -f4 >> ip-addresses.txt; done
for i in $(cat ip-addresses.txt); do shodan host $i; done
```

---

### DNS Records

Pull everything — A, MX, NS, TXT, SOA:

```bash
dig any <DOMAIN>
```

What each record type gives you:

`A` → IPs, scope confirmation. `MX` → mail provider (Google = skip, self-hosted = target). `NS` → hosting provider identity. `TXT` → third-party services in use, internal IPs in SPF records.

> [!tip] SPF TXT record is the highest-yield field — `ip4:` entries leak internal IPs directly, and `include:` entries expose every third-party service the company uses for email (Mailgun, Outlook, Atlassian, etc.).

---

### What TXT Records Leak — Third-Party Intel

`atlassian-domain-verification` → Jira/Confluence in use → software dev platform, potential creds reuse target

`logmein-verification-code` → centralized remote access platform → if you get admin creds here, you may own everything it manages

`mailgun` in SPF → email API + webhooks → hunt for exposed API keys, test endpoints for IDOR/SSRF

`spf.protection.outlook.com` → Office 365 → implies OneDrive, potentially Azure blob/file storage (SMB) — check for exposed cloud storage

`MS=ms...` in TXT → Microsoft domain verification → usually matches a login username or ID on the management portal

> [!tip] NS records pointing to a hosting provider's nameservers reveal who hosts the infrastructure. Cross-reference with Shodan results to confirm and identify additional exposed ports/services.