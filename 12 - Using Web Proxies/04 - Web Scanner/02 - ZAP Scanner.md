# ZAP Scanner

Free, no throttle, full feature set.

**Spider** — right-click request in History → `Attack > Spider`. Or HUD right pane → Spider button (second from top). ZAP will prompt to add the site to scope — say yes. Builds site tree by following links.

After Spider finishes, run **Ajax Spider** (third HUD right-pane button) — catches links loaded via JS/AJAX that normal Spider misses. Takes longer, worth running on JS-heavy apps.

**Passive scan runs automatically** as Spider makes requests — no extra step needed. Check HUD alerts (left pane = current page, right pane = whole app) or `Alerts` tab in main UI.

**Active scan** — HUD right pane → Active Scan button. If Spider hasn't run yet, ZAP runs it first automatically. Fuzzes all identified pages and params for SQLi/XSS/CMDi/etc. Watch `Alerts` tab populate as it runs.

Triage by severity — High alerts first. Click any alert for details including the exact request/payload used and evidence in the response. Replay directly from the alert window via HUD or Request Editor.

**Report** — `Report > Generate HTML Report` (or XML/Markdown). Open in browser.

> [!tip] Passive scan is free while Spider runs — by the time Spider finishes you already have a first-pass vuln list before active scan even starts.