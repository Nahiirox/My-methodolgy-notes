# Certificate Transparency Logs

CT logs are public, append-only records of every SSL/TLS cert issued by any CA. Passive recon — no target contact. Gives historical subdomain coverage that brute force misses, including old/expired certs pointing to legacy infra.

All subdomains, including ones never linked publicly, appear here if a cert was ever issued for them.

---

### crt.sh Queries

All subdomains for a domain, deduplicated:

```bash
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

Filter output — hunt specific patterns (dev, staging, admin, internal):

```bash
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq -r '.[] | select(.name_value | contains("<KEYWORD>")) | .name_value' | sort -u
```

Wildcard certs (`*.target.com`) in the output = that subdomain level exists. Old/expired entries = legacy hosts, outdated software, potentially unpatched.

> [!tip] Run CT log enum before any active brute force. If crt.sh gives you 40 subdomains, half your brute force work is already done passively. Also check expired certs — `*.dev.target.com` from 3 years ago may still resolve to an abandoned host.