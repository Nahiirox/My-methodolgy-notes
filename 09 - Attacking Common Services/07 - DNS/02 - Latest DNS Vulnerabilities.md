# Subdomain Takeover

CNAME record points to a third-party service (AWS S3, GitHub Pages, etc.) that no longer exists → register that external resource yourself → you now control what the subdomain serves. High-value for phishing since it's a legitimate subdomain of the target domain.

Attack chain: find dangling CNAME → register the dead external resource → subdomain now points to your infrastructure → visitors trust it because the parent domain is legitimate.

Also enables: cookie theft, CSRF, CORS abuse, CSP bypass — not just phishing.

---

## Finding Dangling CNAMEs

DNS recon → look for CNAME records returning NXDOMAIN or HTTP 404 on the target resource. Tools like `subjack`, `subzy`, or manual `dig`/`nslookup`:

```bash
dig CNAME <SUBDOMAIN>.<TARGET_DOMAIN>
```

If the CNAME target returns 404 or NXDOMAIN → check if the resource (S3 bucket, GitHub Pages, Heroku app) is claimable on the provider's platform.

---

## Verification

Confirm the subdomain is actually dangling — not just a temporary error:

```bash
curl -s -o /dev/null -w "%{http_code}" https://<SUBDOMAIN>.<TARGET_DOMAIN>
```

`404` or `NoSuchBucket` / `There is no app configured at that hostname` = likely takeable.

> [!tip] This is a legitimate bug bounty finding on most programs. Document the CNAME record, the dead external resource, and proof you could register it. Most programs don't require you to actually take it over — a PoC showing the registration is available is enough.

> [!warning] Actually registering the external resource and serving content from a client's subdomain without explicit written permission is out of scope for most assessments. Confirm scope before doing anything beyond enumeration.