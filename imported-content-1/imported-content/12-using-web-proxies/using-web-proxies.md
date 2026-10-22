# Web Proxies

Web proxies sit between browser/app and back-end server as a MITM for HTTP/HTTPS — capture every request/response, intercept and modify mid-flight, replay with changes. Beyond capture/replay: vuln scanning, fuzzing, crawling, app mapping, request analysis, config testing, code review support. The two tools: **Burp Suite** (best UI, built-in Chromium; Pro-only = active scanner, fast Intruder, some extensions) and **OWASP ZAP** (fully free/open-source, no throttling, HUD, gaining parity). Learn both — same concepts, pick per engagement.

## Setting Up

Both run on Windows/macOS/Linux, pre-installed on Pwnbox/Parrot/Kali. Both are Java apps (JRE bundled; `java -jar </path/to/burpsuite.jar>` works cross-platform).

- **Burp startup:** temporary project (community can't save; Pro can persist projects/active scans) → Use Burp Defaults → Start Burp.
- **ZAP startup:** `zaproxy` or menu → "No, do not persist" for a temp session.

Dark theme: Burp > Settings > User interface > Display; ZAP > Tools > Options > Display > Flat Dark.

## Proxy Setup

Fastest path: the pre-configured browser (proxy + CA cert already set).

- Burp: Proxy > Intercept > **Open Browser**.
- ZAP: Firefox icon at the end of the top bar.

Real browser instead: point it at the proxy — both default to `127.0.0.1:8080` (change under Burp Proxy > Proxy settings > Proxy listeners, or ZAP Tools > Options > Network > Local Servers/Proxies). **FoxyProxy** extension makes switching one click: Options > Add > IP 127.0.0.1, port 8080, name it Burp/ZAP.

**CA certificate** — without it HTTPS traffic stalls on trust prompts:

- Burp: browse to `http://burp` → CA Certificate.
- ZAP: Tools > Options > Network > Server Certificates > Save (Generate makes a new one).

Install in Firefox: `about:preferences#privacy` → View Certificates → Authorities → Import → "Trust this CA to identify websites" + email users.

## Intercepting Web Requests

Burp: Proxy > Intercept, toggle "Intercept is on/off", Forward/Drop the held request.

ZAP: interception off by default — green button or CTRL+B toggles; "step" (right of the break button) forwards. ZAP's **HUD** (button at end of top bar) puts the controls in the browser: second button on left pane = intercept; on a caught request: **Step** (send + break on next, examine each step) vs **Continue** (send the rest through).

Manipulation workflow: intercept → edit request inline → forward. Classic demo: page restricts an IP field to numbers via front-end JS; intercept the POST, change `ip=1` to `ip=;ls;` → response comes back as `ls` output = command injection. Same mechanism drives SQLi, XSS, XXE, upload bypasses, auth bypass, deserialization tests.

## Intercepting Responses

Intercept the server's response before the browser renders it — rewrite HTML to suit the test:

Burp: Proxy > Proxy settings > Response interception rules > enable Intercept Response. Force refresh (CTRL+SHIFT+R), forward the request, the response is held — edit `type="number"` → `type="text"`, `maxlength="3"` → `maxlength="100"`, forward again. Works for enabling disabled buttons persistently (per page load).

ZAP: click **Step** on the intercepted request and the response gets caught automatically; edit + Continue.

ZAP HUD shortcuts (left pane): light-bulb button = show hidden fields / enable disabled fields without intercepting; **Comments** button (add via +) flags HTML comment positions inline with hover content. Burp's equivalents live under Proxy > Proxy settings > Response modification rules (e.g. "Unhide hidden form fields").

## Automatic Modification

Rules that rewrite every request/response in flight — no manual interception.

**Requests (match & replace):**

- Burp: Proxy > Proxy settings > HTTP match and replace > Add — Type `Request header`, Match `^User-Agent.*$`, Replace `User-Agent: HackTheBox Agent 1.0`, Regex true.
- ZAP: CTRL+R (Replacer) > Add — Match Type `Request Header (will add if not present)`, Match String `User-Agent`, Replacement `HackTheBox Agent 1.0`. The Initiators tab scopes where the rule applies (default: all HTTP(S)).

**Responses:** same idea — Type `Response body`, Match `type="number"`, Replace `type="text"`, no regex needed for exact strings. Survives page refreshes, so the front-end bypass sticks.

## Repeating Requests

Resend any historical request with edits, response shown in-tool — no re-intercept cycle.

History lives in Burp Proxy > HTTP History (or ZAP's bottom History pane / HUD). Both keep WebSockets history too (async updates/data-fetching — advanced testing). Burp shows both Original and Edited versions of a modified request (pane header toggle); ZAP shows only the final.

- Burp: CTRL+R on a request → Repeater (CTRL+SHIFT+R) → edit → **Send**. Right-click > Change Request Method flips POST↔GET without rewriting.
- ZAP: right-click > Open/Resend with Request Editor → Send; Method drop-down for any verb.
- HUD: click a history entry → **Replay in Console** or **Replay in Browser** (rendered).

## Encoding/Decoding

URL-encode anything that would break request parsing: space (end-of-data), `&` (param delimiter), `#` (fragment). Burp Repeater: select text → CTRL+U or Convert Selection > URL; can enable encode-as-you-type. ZAP auto-encodes request data behind the scenes.

Both ship full encoders for HTML / Unicode / Base64 / ASCII hex:

- Burp: **Decoder** tab (input → Encode as/Decode as; output pane can be re-encoded with another scheme directly). Newer Burp also has the **Inspector** in Proxy/Repeater for inline URL/Base64 decode.
- ZAP: CTRL+E = Encoder/Decoder/Hash; Decode tab auto-tries schemes; custom tabs can mix chosen encoders.

Typical flow: find base64 cookie `eyJ1c2VybmFtZSI6Imd1ZXN0IiwgImlzX2FkbWluIjpmYWxzZX0=` → decode → `{"username":"guest", "is_admin":false}` → flip to admin/true → re-encode → paste into Repeater/Request Editor.

## Proxying Tools

Route CLI tools and thick clients through the proxy (slows them down — only when investigating):

- **proxychains** — universal. In `/etc/proxychains.conf`, comment the socks line and add `http 127.0.0.1 8080`; run with `-q` for quiet mode:

```bash
proxychains -q curl http://SERVER_IP:PORT
```

- **Metasploit** — per-module `set PROXIES HTTP:127.0.0.1:8080`:

```text
use auxiliary/scanner/http/robots_txt
set PROXIES HTTP:127.0.0.1:8080
set RHOST <TARGET_IP>
set RPORT <PORT>
run
```

Any other tool: same pattern — find its proxy setting, point it at 127.0.0.1:8080.

## Burp Intruder

Burp's fuzzer — fuzz directories, subdomains, params, values; replaces ffuf/gobuster/wfuzz for most jobs. **Community is throttled to 1 req/sec** (CLI tools do ~10k/s) — short queries only; Pro is unlimited and among the best fuzzers available.

Workflow: CTRL+I sends a Proxy History request to Intruder.

1. **Positions** — wrap the fuzz target with `§` (or select + Add §). Directory fuzzing: `GET /§DIRECTORY§/`. Keep the two trailing blank lines.
2. **Payloads** — set 1 (per position, tied to attack type — Sniper = one position; Cluster Bomb = several). Payload Types: Simple List (basic), Runtime file (streamed line-by-line — use for huge wordlists to spare memory), Character Substitution (permutations). Load `/opt/useful/seclists/Discovery/Web-Content/common.txt`; multiple loads append (combine/customize).
3. **Payload Processing** — rules per item: e.g. Skip if matches regex `^\..*$` to drop dot-prefixed entries; can also suffix extensions, filter, etc.
4. **Payload Encoding** — URL-encode toggle per character set (leave on).
5. **Settings** — retries to 0 if desired; **Grep - Match**: clear defaults, add `200 OK`, disable "Exclude HTTP Headers" → results get a flag column to sort by. Grep - Extract pulls just a slice of long responses. Resource Pool caps network use for big attacks.

Start Attack → sort by flag/status/length → `/admin/` pops with 200. Same machinery does credential brute-forcing and AD password spraying against OWA/VPN/RDS/Citrix/custom AD-auth portals.

## ZAP Fuzzer

No paid tier, no throttle — the free alternative to Intruder, slightly fewer features.

Right-click request > Attack > Fuzz:

1. **Locations** — select text (e.g. `test` in the path) > Add → green marker + payload window.
2. **Payloads** — Add → type File (own wordlist), **File Fuzzers** (built-in databases — dirbuster directory-list etc.), Numberzz (number sequences with increments).
3. **Processors** — per-payload transforms: Base64 encode/decode, MD5/SHA-1/256/512, URL encode/decode, Prefix/Postfix String, Script (custom). Use URL Encode to avoid server errors on special chars; Generate Preview shows the final payload shape.
4. **Options** — Concurrent Scanning Threads (e.g. 20 — bounded by CPU and server limits); **Depth first** (all words on one position before next: every password for one user) vs **Breadth first** (each word across all positions: one password tried for every user).

Start Fuzzer → sort by Response code (200s), Size Resp. Body (a different-sized page = different page), or RTT (time-based SQLi detection).

## Burp Scanner

Pro-only. Crawler + passive/active scanning.

**Target scope first** — Target > Site map (everything the proxy has seen) > right-click > Add to scope (Burp offers to restrict all features to in-scope only). Remove dangerous items from scope (logout endpoints), or use advanced scope control with regex includes/excludes.

**Crawler** — Dashboard > New Scan > Crawl. Follows links/forms to map the site; does NOT fuzz for unreferenced paths (that's Intruder/Content Discovery). Scan configuration: custom (speed/limits/login behavior) or preset library (e.g. "Crawl strategy - fastest"). Application login tab: give creds or record a manual login so the crawl covers authenticated content.

**Passive scan** — no new requests; analyzes already-visited pages (missing headers, potential DOM XSS). Run via right-click > Do passive scan. Results in Dashboard > Issue activity with severity + confidence (hunt High/Certain first).

**Active scan** — right-click > Do active scan, or New Scan > Crawl and Audit. Full sequence: crawl + fuzz for pages → passive scan everything → verify passively-found issues with requests → JavaScript analysis → fuzz insertion points/params for XSS, command injection, SQLi. Audit configurations select vuln classes and insertion points (preset: "critical issues only"). Watch live requests in the Logger tab. Filter Issue activity to High/Certain → e.g. OS command injection on the `ip` parameter, with advisory + request/response proof.

**Reporting** — Site map > right-click host > Issue > Report issues for this host. Pick format and contents; HTML opens in a browser with severity/confidence summary, PoCs, remediation. Appendix material for the real report — never a raw-tool deliverable.

## ZAP Scanner

Free equivalent: Spider + passive + active scanning.

- **Spider** — right-click request > Attack > Spider (HUD: second button on right pane). Auto-offers to add the site to scope. Results in Sites tab tree / HUD Sites Tree button. **Ajax Spider** (third button on right pane) additionally catches links requested via JavaScript AJAX after page load — run it after the normal Spider for extra coverage, at the cost of time.
- **Passive scanner** — runs automatically on every Spider/response; alerts populate as you go (HUD alerts: left = current page, right = whole app). Alerts tab in the main UI shows details + affected pages.
- **Active scan** — Active Scan button (right pane); auto-runs Spider first if no site tree. Attacks all identified pages/params; watch progress and sent requests in the UI. High alerts = the compromise-capable ones (module demo: Remote OS Command Injection, PoC `127.0.0.1&cat /etc/passwd&`, evidence `root:x:0:0...` — click through to the exact request/response and replay it).
- **Reporting** — Report > Generate HTML Report (also XML/Markdown).

## Extensions

- **Burp BApp Store** (Extensions tab): sort by Popularity. Some Pro-only; some need Jython etc. installed first. Demo: Decoder Improved → extra encoders/hashes (Hash With > MD5). Worth exploring: Active Scan++, Autorize, Backslash Powered Scanner, Java Deserialization Scanner, JS Link Finder, Retire.JS, Software Version Reporter, CSRF Scanner, Headers Analyzer, CSP Auditor, CMS Scanner, AWS Security Checks, .NET Beautifier, J2EEScan...
- **ZAP Marketplace** (Manage Add-ons > Marketplace): Release/Beta/Alpha stability grades. Demo: install **FuzzDB Files** + **FuzzDB Offensive** → new File Fuzzers wordlists appear, e.g. `fuzzdb > attack > os-cmd-execution > command_execution-unix.txt` — run the fuzzer with it and the command injection pops multiple payload variants (handy against WAFs).