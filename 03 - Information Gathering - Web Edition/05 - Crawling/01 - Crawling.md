# Web Crawling

Crawlers follow links systematically from a seed URL — map structure, find hidden pages, pull sensitive files. Two strategies: **breadth-first** (wide overview, all links at each level before going deeper) vs **depth-first** (follows one path as far as possible, good for reaching deep structure). Pick based on goal.

---

### What to Hunt in Crawl Output

Links → internal paths reveal structure, hidden dirs, API endpoints. External links reveal third-party dependencies.

Comments in source → devs leaving TODO notes, version hints, internal paths, credentials in commented-out code.

Metadata → page titles, author names, keywords, dates → confirm CMS version, username format.

Sensitive files to flag immediately: `.bak`, `.old`, `web.config`, `settings.php`, `config.php`, `error_log`, `access_log`, `.env`, `database.yml` — any of these exposed = creds, API keys, or source code.

---

### Context Is the Point

A single finding is noise. Patterns are signal. `/files/` directory in links → manually visit → directory listing enabled → backup archives exposed. A comment mentioning "file server" + the `/files/` path = confirmed public file exposure.

Hunt for: repeated directory patterns in URLs, any path with `backup`, `old`, `archive`, `temp`, `test`, `dev` in the name, admin paths not linked from the main nav.

> [!tip] When a crawler returns a long list of URLs, filter immediately for high-value patterns: `grep -iE "backup|config|admin|upload|temp|old|dev|test|\.bak|\.sql|\.env" urls.txt`. These are the paths worth manually verifying before anything else.