# WHOIS

Query WHOIS for a domain:

```bash
whois <DOMAIN>
```

---

### What to Extract

`Creation Date` — recently registered (days/weeks ago) + hidden registrant + suspicious hosting = phishing infrastructure. Old domain = likely legit.

`Registrant/Admin/Tech` — if not privacy-protected: real names, email addresses, phone numbers → username format, social engineering targets, org structure hints.

`Name Servers` — self-hosted NS (e.g., `ns1.target.com`) = they manage their own DNS infra. Third-party NS → reveals hosting provider. Shared NS across multiple domains = same infrastructure, pivot to related assets.

`Registrar` — lax abuse policy registrars (often paired with bulletproof hosting) = indicator of malicious infra during threat intel work.

`Expiry Date` — domain expiring soon = potential for domain takeover if it lapses.

---

### Historical WHOIS

WhoisFreaks (whoisfreaks.com) — historical records show ownership changes, old contact info, previous name servers. Old registrant email before privacy service was added = real contact detail.

> [!tip] Always cross-reference WHOIS name servers against the IPs you find in DNS — if NS records point to a different org than the registrant, it may indicate a hosting provider, CDN (Cloudflare/Akamai), or third-party DNS management. Each is a separate entity that may be out of scope.