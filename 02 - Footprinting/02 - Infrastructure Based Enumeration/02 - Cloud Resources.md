# Cloud Resources — Passive Recon

Misconfigured cloud storage (S3/Azure blob/GCP) is auth-optional by default if the admin set it wrong. Treat every cloud hostname you find as potentially open.

S3 buckets appear in subdomain resolution — `s3-website-<region>.amazonaws.com` resolving from a company subdomain = in-scope cloud storage. Check during the host → IP pass.

---

### Finding Cloud Storage

Google dorks — swap company name in:

```
intext:<COMPANY> inurl:amazonaws.com
intext:<COMPANY> inurl:blob.core.windows.net
intext:<COMPANY> inurl:storage.googleapis.com
```

Check page source of the company's main website — images, JS, CSS often load directly from blob/S3 URLs, leaking bucket names without any searching.

GrayHatWarfare (grayhatwarfare.com) — search by company name or abbreviation, filter by file type. Passively lists publicly accessible bucket contents including file names.

> [!tip] Companies often use abbreviations or product names as bucket names, not just the full company name. Try both. If GrayHatWarfare shows `id_rsa` or `id_rsa.pub` in any bucket — download immediately, try against every SSH-exposed host in scope.

domain.glass — paste a domain, get infrastructure fingerprint including Cloudflare status. Confirms Layer 2 (gateway) info passively.

---

### What to Note When You Find a Bucket

Contents visible without auth → document + download everything — config files, keys, documents, source code. Any `id_rsa`, `.pem`, `.key`, `.env`, `.config` file is a critical finding regardless of what else you find. Leaked SSH private key → try against every `22/tcp` host in scope before doing anything else.