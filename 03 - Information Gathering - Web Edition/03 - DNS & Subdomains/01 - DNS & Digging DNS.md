# DNS — Web Recon

Record types that matter for recon: `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias → find old/dev servers), `MX` (mail provider), `NS` (hosting provider identity), `TXT` (third-party services, SPF leaks internal IPs), `SOA` (admin email, zone serial), `SRV` (service locations), `PTR` (reverse lookup).

`/etc/hosts` — add entries here to force local resolution when testing vhosts or internal names that don't resolve publicly.

---

### dig — Core Queries

Default A record lookup:

```bash
dig <DOMAIN>
```

Specific record type:

```bash
dig <DOMAIN> <TYPE>
```

Query a specific nameserver directly:

```bash
dig @<NS_IP> <DOMAIN> <TYPE>
```

All records:

```bash
dig <DOMAIN> ANY
```

Reverse lookup — IP to hostname:

```bash
dig -x <IP>
```

Clean output, answer only:

```bash
dig +short <DOMAIN>
dig +noall +answer <DOMAIN>
```

Full DNS resolution path — shows every hop from root to authoritative:

```bash
dig +trace <DOMAIN>
```

---

### Recon Value Per Record Type

`NS` → hosting provider, who manages DNS infra

`MX` → email provider (Google/self-hosted/third-party)

`TXT` → SPF `ip4:` entries leak internal IPs, `include:` entries reveal every third-party service used

`CNAME` → old/dev server aliases — `dev.target.com CNAME oldserver.example.net` = potential vuln target

`SOA` → admin email (dot → @), useful for username format

New subdomain appearing (e.g., `vpn.target.com`) → new entry point. TXT like `_1password=...` → org uses 1Password → social engineering angle.

> [!tip] `dig +trace` is useful when a record resolves differently than expected — shows exactly where in the delegation chain the answer comes from, which helps identify misconfigurations or split-horizon DNS.