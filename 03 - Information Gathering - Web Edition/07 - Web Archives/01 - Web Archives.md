
# Web Archives

Passive — no direct interaction with target infra, nothing logged on their end.

Hunt via [web.archive.org](https://web.archive.org) — enter the target URL, browse historical snapshots.

What to look for across old captures: removed pages/directories still indexed, old subdomains that no longer resolve (may still be live), exposed files pulled from public access, stack/tech changes that reveal older vulns, employee names or internal paths visible in past content.

> [!tip] Old snapshots often expose endpoints that were "removed" but never actually taken offline — try browsing them on the live site after finding them in archives.