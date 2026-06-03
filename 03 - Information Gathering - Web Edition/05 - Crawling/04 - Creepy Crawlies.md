# Web Crawling — Tools

Run a crawler after robots.txt and sitemap review. Output feeds directory fuzzing, email harvesting, JS analysis, and comment review.

ReconSpider — Scrapy-based, outputs structured JSON, covers emails/links/external files/JS/forms/comments:

```bash
pip3 install scrapy
python3 ReconSpider.py http://<TARGET>
```

Output is `results.json` — parse it:

```bash
cat results.json | jq .emails
cat results.json | jq .links
cat results.json | jq .external_files
cat results.json | jq .js_files
cat results.json | jq .comments
```

---

### Triage Order for results.json

`emails` → username format confirmation, social engineering targets, spray list fodder

`external_files` → PDFs, DOCs → run through `exiftool` for author metadata, internal paths, username format

`js_files` → pull each JS file, grep for API keys, endpoints, hardcoded creds, internal hostnames

`comments` → HTML comments in source often contain TODO notes, version strings, disabled features, internal paths

`links` → filter for `/admin`, `/backup`, `/upload`, `/api`, non-standard paths → add to directory fuzz list

> [!tip] `jq .js_files` output → download each file → `grep -iE "api_key|token|password|secret|endpoint|internal|staging"`. Developers frequently hardcode credentials and internal URLs in frontend JS that gets deployed to prod.