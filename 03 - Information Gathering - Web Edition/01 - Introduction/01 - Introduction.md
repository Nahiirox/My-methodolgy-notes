# Web Reconnaissance — Overview

Two modes: **passive** (no direct target contact, OSINT/public data) and **active** (direct interaction, generates logs). Exhaust passive before going active.

---

### Active Techniques — Detection Risk

`High` — port scanning, vuln scanning (sends payloads, trips IDS)

`Medium` — network mapping, web spidering (tunable — mimic legit traffic to lower risk)

`Low` — banner grabbing, OS fingerprinting, service version enum (logged but rarely alerted)

---

### Passive Techniques — All Very Low Detection Risk

Search engine queries / Google dorks → exposed content, employee info, subdomains

WHOIS → registrant, contact, name servers, registration dates

DNS enum → subdomains, MX, NS, TXT, zone transfer attempts

Web archive (Wayback Machine) → old versions of pages, removed content, historical configs

Social media / LinkedIn → org structure, stack, usernames, email format

GitHub/GitLab → hardcoded creds, internal tooling, leaked keys, API endpoints

---

### Recon Goals (Filter Everything Through These)

Find all assets (subdomains, IPs, tech stack, cloud resources) → discover hidden/exposed content (configs, backups, keys) → map attack surface (entry points, versions, misconfigs) → gather intel for exploitation or social engineering.

> [!tip] Passive recon feeds active recon. Don't port scan until you've mapped the full scope passively — you may find assets that would otherwise be missed, and you'll know what to prioritize.