# Web Proxies — Burp Suite & ZAP

Both pre-installed on Parrot/Kali. Burp: community (free, throttled Intruder) vs Pro (active scanner, fast Intruder). ZAP: fully free and open-source.

Default proxy port: `127.0.0.1:8080` for both. Change in Burp: `Proxy>Proxy settings>Proxy listeners`. ZAP: `Tools>Options>Network>Local Servers/Proxies`.

---

## Setup

FoxyProxy → add `127.0.0.1:8080`, name it Burp/ZAP, toggle when needed. Pre-configured browser in each tool bypasses this.

Install CA cert — required for HTTPS interception without errors:

Burp → browse to `http://burp` with Burp as proxy → download CA Certificate → Firefox `about:preferences#privacy` → View Certificates → Authorities → Import → trust for websites.

ZAP → `Tools>Options>Network>Server Certificates` → Save → import same way in Firefox.

---

## Intercept

Burp: `Proxy>Intercept` → toggle on/off → Forward / Drop intercepted request.

ZAP: green button top bar = off, click to enable. Or `CTRL+B`. ZAP HUD: second button left pane.

Intercept response in Burp: `Proxy>Proxy settings>Response interception rules` → enable. Then forward the request, response gets intercepted next.

ZAP: click Step after intercepting request → auto-intercepts the response. ZAP HUD lightbulb button = instantly show/enable hidden or disabled form fields without intercepting.

---

## Repeater / Request Editor

Burp: find request in `Proxy>HTTP History` → `CTRL+I` to send to Intruder, `CTRL+R` to send to Repeater → `CTRL+SHIFT+R` to jump there → modify and Send.

ZAP: right-click request in History → `Open/Resend with Request Editor` → Send. Or ZAP HUD → History pane → click request → Replay in Console / Replay in Browser.

Right-click in Burp Repeater → `Change Request Method` to toggle GET/POST without rewriting.

---

## Match & Replace (Auto-Modify)

Burp: `Proxy>Proxy settings>HTTP match and replace rules` → Add.

Replace User-Agent automatically:

- Type: `Request header`
- Match: `^User-Agent.*$`
- Replace: `User-Agent: HackTheBox Agent 1.0`
- Regex: `True`

Auto-fix a number-only input field in responses:

- Type: `Response body` / Match: `type="number"` / Replace: `type="text"`
- Type: `Response body` / Match: `maxlength="3"` / Replace: `maxlength="100"`

ZAP equivalent: `CTRL+R` → Replacer → Add. Same fields: Match Type, Match String, Replacement String.

---

## Encoding & Decoding

Burp Decoder tab: paste encoded string → Decode as Base64 / URL / HTML / etc. Output can be piped to another encoder in the same view.

Burp Inspector (in Repeater/Proxy): inline decode of selected text.

ZAP: `CTRL+E` → Encoder/Decoder/Hash → paste into Decode tab → outputs multiple formats simultaneously.

URL-encode in Burp Repeater: select text → right-click → `Convert Selection>URL>URL-encode key characters` or `CTRL+U`.

Special chars that MUST be URL-encoded in POST data: spaces (→`+` or `%20`), `&` (→`%26`), `#` (→`%23`).

---

## Burp Intruder (Fuzzer)

Community = 1 req/sec (throttled). Pro = unlimited. Use ffuf/ZAP Fuzzer for speed in community version.

Send request to Intruder: `CTRL+I`. Navigate: `CTRL+SHIFT+I`.

Positions tab → wrap target word with `§` markers or select + click `Add §`. Attack type: `Sniper` for single position.

Payloads tab:

- Payload Type: `Simple List` (load wordlist file) or `Runtime file` (large lists, saves memory)
- Load: `/opt/useful/seclists/Discovery/Web-Content/common.txt`
- Payload Processing: Add → `Skip if matches regex` → `^\..*$` (skip dot-files)
- Payload Encoding: URL-encode enabled by default

Settings tab → Grep Match: clear defaults → add `200 OK` → disable "Exclude HTTP Headers" → filter results by success code.

Start Attack → sort results by `200 OK` column or Status column.

---

## ZAP Fuzzer

No throttling. Right-click request in History → `Attack>Fuzz` → Fuzz window.

Locations: select the word to fuzz → Add → sets green marker and opens Payloads window.

Payloads → Add:

- Type: `File` (custom wordlist) or `File Fuzzers` (built-in SecLists/dirbuster/FuzzDB)
- Select wordlist from dropdown

Processors → Add → `URL Encode` (UTF-8) → Generate Preview → Add. Required to avoid server errors on special chars.

Options: set `Concurrent Scanning Threads per Scan` to 20. Strategy: `Depth first` (all passwords per user) vs `Breadth first` (all users per password).

Start Fuzzer → sort results by Response Code → 200 = hit.

Install FuzzDB via ZAP Marketplace (`Manage Add-ons`) for extra payload lists including OS command injection wordlists.

---

## Burp Scanner (Pro Only)

Crawl only = maps site via links/forms, no fuzzing. Crawl and Audit = crawl + passive + active scan.

`Target>Site map` → right-click target → `Add to scope`. Then `Target>Scope` to review/edit.

Dashboard → New Scan → select scope → Crawl configuration: `Crawl strategy - fastest`. Add login creds if authenticated scan needed.

Passive scan: right-click request → `Do passive scan`. Analyzes existing responses, no new requests. Shows potential issues + confidence level.

Active scan: right-click → `Do active scan` or Dashboard → New Scan → `Crawl and Audit`. Select `Audit checks - critical issues only` for high-severity focus.

View results: Dashboard `Issue activity` pane → filter High + Certain. Export: `Target>Site map` → right-click → `Issues>Report issues for this host` → HTML.

---

## ZAP Scanner

Spider (Crawler): right-click request → `Attack>Spider` or HUD right pane second button. Ajax Spider (third button) = catches JS-loaded links, run after normal spider.

Active Scan: HUD right pane Active Scan button. Alerts accumulate in HUD and main UI Alerts tab during scan. High alerts = direct compromise risk.

Report: `Report>Generate HTML Report`. Formats: HTML, XML, Markdown.

---

## Proxying CLI Tools

Route any tool through Burp/ZAP to inspect and manipulate its requests.

proxychains — edit `/etc/proxychains.conf`:

```
# comment out: socks4  127.0.0.1 9050
http 127.0.0.1 8080
```

```bash
proxychains -q curl http://<TARGET>:<PORT>
proxychains -q nmap <TARGET>
```

Metasploit — set proxy per module:

```
set PROXIES HTTP:127.0.0.1:8080
```

> [!tip] Only proxy tools when actively investigating requests. Proxying significantly slows down scans like nmap — disable when not needed.