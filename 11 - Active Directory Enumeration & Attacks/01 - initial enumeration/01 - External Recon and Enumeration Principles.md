# External Recon & Enumeration Principles

Passive first, wide → narrow. Exhaust passive before going active. If stuck gaining foothold internally → loop back here, especially breach data.

Hunt for: ASN/netblocks, domain info (subdomains, mail/NS records, exposed portals), email/username schema, public files with metadata or embedded intranet links, breach creds.

---

## IP & Domain

BGP Toolkit (bgp.he.net) → ASN, netblocks, hosting provider. Small orgs usually sit on shared cloud infra (Cloudflare/AWS/Azure) — don't interact with shared infra without explicit written approval from the provider, not just the client.

Validate IP, find co-hosted domains:

```
viewdns.info → reverse IP lookup → <TARGET>
```

Validate NS records, pull new IPs to scope-check:

```bash
nslookup ns1.<DOMAIN>
nslookup ns2.<DOMAIN>
```

> [!warning] IPs from NS lookups may be out of scope. Confirm in writing before touching them.

---

## Public Data & Schema

Job postings (LinkedIn, Glassdoor, Indeed) → stack, software versions, AD schema hints. Contact pages → email naming convention = username format for spraying.

Hunt public files for metadata, intranet links, internal paths:

```
filetype:pdf inurl:<DOMAIN>
filetype:docx inurl:<DOMAIN>
```

Confirm email naming schema:

```
intext:"@<DOMAIN>" inurl:<DOMAIN>
```

Scrape LinkedIn → username list variants (flast, first.last, f.last):

```bash
linkedin2username -c <COMPANY_NAME>
```

Hunt GitHub/cloud storage for hardcoded creds: Trufflehog for repos, Greyhat Warfare for exposed S3/Azure blobs.

---

## Breach Data

Dehashed → cleartext creds or hashes tied to `<DOMAIN>`. Old passwords still hit VPN, OWA, Citrix, RDS, VMware Horizon, or anything running AD auth.

```bash
sudo python3 dehashed.py -q <DOMAIN> -p
```

Got creds → try exposed portals before spraying. Got hashes → crack offline, then same. No hits → fold breach usernames into spray list anyway.

> [!tip] Breach data is the stuck-on-foothold escape hatch. One valid low-priv domain user opens most AD enum.