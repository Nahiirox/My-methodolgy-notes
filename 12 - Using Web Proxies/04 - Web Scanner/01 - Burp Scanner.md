# Burp Scanner

Pro only — not available in Community. Covers crawl, passive scan, active scan, and reporting.

**Scope first** — `Target > Site map` → right-click target → `Add to scope`. Exclude logout endpoints and anything dangerous to scan. Scope gates all scanner features and saves resources by dropping out-of-scope traffic.

**Crawl** — `Dashboard > New Scan` → `Crawl` → select preset (`Crawl strategy - fastest` for speed). Builds site map by following links and forms. Does not fuzz for unreferenced pages — that's Intruder/ffuf/Content Discovery.

Add creds in the `Application login` tab if scanning authenticated areas — without it, scanner only sees what an unauth user sees.

**Passive scan** — no new requests sent, analyzes already-visited pages. Right-click request/target → `Do passive scan`. Fast, low noise. Prioritize High severity + Certain/Firm confidence hits.

**Active scan** — right-click → `Do active scan`, or `Dashboard > New Scan > Crawl and Audit`. Runs: crawl → fuzz → passive scan → verify vulns → JS analysis → param fuzzing for SQLi/XSS/CMDi/etc. Use preset `Audit checks - critical issues only` if you want High vulns fast without the noise.

Monitor progress and requests in `Dashboard > Tasks > View details > Logger tab` or the main `Logger` tab.

Filter results: `Issue activity` pane → filter High + Certain/Firm → investigate those first.

**Report** — `Target > Site map` → right-click target → `Issues > Report issues for this host`. Export as HTML, open in browser. Use as appendix data only — never submit raw tool output as a client deliverable.

> [!tip] Crawl + authenticated session covers way more surface — always add login creds or a recorded login sequence before running against anything that has auth.