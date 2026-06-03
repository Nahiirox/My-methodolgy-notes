# Web Fingerprinting

Goal: identify server software, version, framework, CMS, WAF — then match against known CVEs and misconfigs for that specific stack.

---

### Banner Grabbing

HTTP headers — follow redirects manually, each hop may reveal more:

```bash
curl -I <TARGET>
curl -I https://<TARGET>
curl -I https://www.<TARGET>
```

Key headers to read: `Server` (web server + version), `X-Powered-By` (language/framework), `X-Redirect-By` (CMS doing the redirect), `Link` (wp-json path = WordPress), `Set-Cookie` (framework fingerprint from cookie names).

---

### WAF Detection

Detect WAF type before scanning further — changes how you approach everything:

```bash
wafw00f <TARGET>
```

WAF present → adjust tooling, consider bypass techniques, expect some probes to be blocked or filtered.

---

### Nikto — Software Identification

Fingerprinting only, no vuln scanning noise (`-Tuning b`):

```bash
nikto -h <TARGET> -Tuning b
```

Output to look for: server version (compare against current = flag if outdated), CMS indicators (`/wp-login.php`, `license.txt`, `/wp-json/`), missing security headers (HSTS, X-Content-Type-Options, CSP), exposed config/license files.

---

### What Fingerprinting Tells You

`Apache 2.4.41` → compare against current version → CVE lookup for that specific version

`X-Redirect-By: WordPress` → WordPress installed → check `/wp-login.php`, `/wp-json/`, `xmlrpc.php`

`Wordfence WAF` → WordPress-specific WAF → bypass techniques differ from generic WAFs

`license.txt` accessible → may reveal exact CMS version

Missing `Strict-Transport-Security` → HTTPS not enforced → potential downgrade

> [!tip] Always follow redirect chains manually with `curl -I` — each intermediate response may expose a different header set. The redirect from HTTP to HTTPS to www often leaks the most info at different hops.

> [!warning] Nikto `-Tuning b` still generates 1000+ requests. Wordfence and similar WAFs will log and potentially block this. Run wafw00f first to know what you're dealing with before running any scanner.