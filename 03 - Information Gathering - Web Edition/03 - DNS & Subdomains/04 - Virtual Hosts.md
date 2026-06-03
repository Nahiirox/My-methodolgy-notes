# Virtual Host Enumeration

VHosts ≠ subdomains. Subdomains have DNS records. VHosts are server-side configs that respond to a specific `Host:` header — no DNS record required. Internal/dev VHosts are invisible to subdomain enum unless you fuzz the Host header directly against the IP.

Access a VHost with no DNS record — add it to `/etc/hosts`:

```bash
echo "<TARGET_IP> <VHOST>.<DOMAIN>" >> /etc/hosts
```

---

### Gobuster VHost Brute Force

Sends requests with each wordlist entry as the `Host:` header, looks for responses that differ from the default:

```bash
gobuster vhost -u http://<TARGET_IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain
```

Non-standard port:

```bash
gobuster vhost -u http://<TARGET_IP>:<PORT> -w <WORDLIST> --append-domain
```

Useful flags: `-t` increase threads / `-k` ignore SSL errors / `-o <OUTFILE>` save output.

---

### ffuf VHost Brute Force

Filter by response size to cut noise — run once to see what the default response size is, then add `-fs <SIZE>`:

```bash
ffuf -w <WORDLIST> -u http://<TARGET_IP> -H "Host: FUZZ.<DOMAIN>" -fs <DEFAULT_SIZE>
```

> [!tip] Always check what the server returns for an unknown Host header first (baseline size/status). Then filter that size out — everything left is a real VHost. Without this filter, ffuf output is noise.

> [!warning] VHost fuzzing generates one HTTP request per wordlist entry against the same IP — high request volume, easily spotted by WAF/IDS. Use `-t 10` (default) rather than cranking threads on external targets.