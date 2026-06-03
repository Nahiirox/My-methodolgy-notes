# DNS — Footprinting

Ports: UDP/53 (queries), TCP/53 (zone transfers + large responses). Key record types to pull: A, NS, MX, TXT, CNAME, PTR, SOA.

---

### Query Sequence

NS records — find all nameservers, query each one separately (different servers may have different zones):

```bash
dig ns <DOMAIN> @<TARGET>
```

Server version — CHAOS query, works if the server exposes it:

```bash
dig CH TXT version.bind <TARGET>
```

All records at once:

```bash
dig any <DOMAIN> @<TARGET>
```

SOA record — admin email (dot → @), serial number, zone owner:

```bash
dig soa <DOMAIN> @<TARGET>
```

---

### Zone Transfer — Highest Value Move

Try AXFR first, before any brute force — dumps the entire zone in one shot:

```bash
dig axfr <DOMAIN> @<TARGET>
```

Internal zone AXFR — if the ANY or NS query reveals an `internal.` subdomain, try it explicitly:

```bash
dig axfr internal.<DOMAIN> @<TARGET>
```

> [!tip] Internal zone AXFR leaks DC hostnames, mail servers, VPN hosts, WSUS servers, and workstation names with IPs — complete internal network map. Try every zone you discover, not just the primary.

> [!warning] Zone transfer working = `allow-transfer` is misconfigured (`any` or your subnet allowed). This is a critical finding — document the full output as evidence.

---

### Subdomain Brute Force

Bash loop — slow but transparent:

```bash
for sub in $(cat <WORDLIST>); do dig $sub.<DOMAIN> @<TARGET> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt; done
```

dnsenum — tries AXFR first, then brute forces, outputs to file:

```bash
dnsenum --dnsserver <TARGET> --enum -p 0 -s 0 -o <OUTFILE>.txt -f <WORDLIST> <DOMAIN>
```