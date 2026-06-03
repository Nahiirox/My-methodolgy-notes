# Subdomain Enumeration

Two modes: **passive** (CT logs, search engines, OSINT — no target contact) and **active** (DNS brute force, zone transfer attempts — generates queries). Run passive first, active to fill gaps.

High-value targets on subdomains: dev/staging environments (relaxed security, exposed configs), admin/login portals not linked from main site, legacy apps on old software versions, internal documents/configs inadvertently exposed.

---

### Passive — CT Logs

crt.sh — pulls all subdomains from certificate SAN fields, no target interaction:

```bash
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

---

### Active — Brute Force

dnsenum — attempts zone transfer first, then brute forces, `-r` recurses into found subdomains:

```bash
dnsenum --enum <DOMAIN> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
```

ffuf — faster, filter by response size/status to cut noise:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -u http://<DOMAIN> -H "Host: FUZZ.<DOMAIN>" -fs <SIZE>
```

Manual bash loop — transparent, no tool dependency:

```bash
for sub in $(cat <WORDLIST>); do dig $sub.<DOMAIN> @<NS_IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt; done
```

---

### Wordlist Selection

General: `subdomains-top1million-20000.txt` or `110000.txt` from SecLists — covers most common names.

Targeted: build custom list from job postings, product names, tech stack, employee names — higher hit rate on less obvious subdomains.

> [!tip] Always try zone transfer before brute forcing — if it works, you get everything instantly. `dig axfr <DOMAIN> @<NS_IP>`. Only brute force after confirming zone transfer is blocked.

> [!warning] Wildcard DNS (`*.target.com → same IP`) makes brute force results unreliable — every guess "resolves." Test with a random string first: `dig randomgarbage1234.<DOMAIN>`. If it resolves, wildcards are set — filter by checking whether all results point to the same IP.