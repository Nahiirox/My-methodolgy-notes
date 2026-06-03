# Subdomain Fuzzing (ffuf)

Public domains — FUZZ goes in the subdomain position:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.<DOMAIN>/
```

Private/local domains (HTB, internal) — public DNS has no record of subdomains, so this returns zero hits even if subdomains exist. That's not "no subdomains," that's "no public DNS record." → vhost fuzzing instead (next section).

> [!warning] Adding `<DOMAIN>` to `/etc/hosts` only resolves that exact entry — subdomains still go to public DNS and fail. Don't mistake 0 results for no attack surface.

Wordlists: `/opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt` for fast pass, larger lists in the same dir for thorough enum.