# robots.txt

First thing to check on any new web target — passive, zero interaction beyond a single GET:

```bash
curl -s https://<TARGET>/robots.txt
```

---

### What to Extract

`Disallow` entries = paths the owner wants hidden from search engines. These are the interesting ones — admin panels, backup dirs, staging paths, internal APIs, private content.

`Allow` entries — less interesting but confirm public structure.

`Sitemap:` directive → grab the sitemap URL → instant full page inventory without crawling:

```bash
curl -s <SITEMAP_URL>
```

---

### Reading the Output

`Disallow: /admin/` → admin panel exists at that path, try it manually.

`Disallow: /backup/` → obvious target.

`Disallow: /private/` → sensitive content, attempt direct access.

`User-agent: Googlebot` with specific rules → tells you what they're trying to hide from search indexing specifically.

Multiple `Disallow` entries with patterns → map these as a preliminary site structure before running any crawler or fuzzer. Cross-reference with directory brute force results.

> [!tip] robots.txt is a roadmap written by the target. Every `Disallow` path is something they wanted hidden — manually check each one before running any automated tooling. A `/backup/` or `/old/` entry that returns 200 or 403 is immediately worth pursuing.

> [!warning] Honeypot directories sometimes appear in robots.txt to catch aggressive bots. If you hit a path from robots.txt and your IP gets blocked immediately, it was a trap. Note this as an indicator of active monitoring on the target.