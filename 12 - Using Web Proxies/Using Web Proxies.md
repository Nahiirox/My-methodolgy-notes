# Using Web Proxies

## Intro to Web Proxies

MITM tools between browser/app and server that capture, view, and modify HTTP/HTTPS traffic, unlike full packet sniffers like Wireshark.

### Burp Suite

Pro/Enterprise-only: active scanner, fast Intruder, loading certain extensions. Free Community edition covers most pentesting needs.

### OWASP Zed Attack Proxy (ZAP)

Free and open-source, no paid-tier throttling or feature limits, unlike Burp Community edition.

## Setting Up

Install and launch Burp Suite and ZAP. Both are pre-installed on PwnBox, Parrot, and Kali.

### Burp Suite

```bash
burpsuite # launch from terminal
java -jar <FILE> # cross-platform jar launch, needs JRE
```

Community edition: temporary projects only, no saving. Pro/Enterprise: can save or open projects on disk.

### ZAP

```bash
zaproxy # launch from terminal
java -jar <FILE> # cross-platform jar launch, needs JRE
```

Prompts for a new or temporary project on launch, unlike free Burp.

## Proxy Setup

Configure a browser, or use the pre-configured browser, to route traffic through Burp/ZAP for interception.

### Pre-Configured Browser

Comes with proxy settings and CA certificate pre-installed.

Burp: (Proxy>Intercept), Open Browser button.  
ZAP: Firefox icon at the end of the top toolbar.

### Proxy Setup

Default proxy port: 8080 (both tools). Change: Burp (Proxy>Proxy settings>Proxy listeners), ZAP (Tools>Options>Network>Local Servers/Proxies). Firefox proxy port must match.

FoxyProxy Firefox extension for fast proxy switching (pre-installed and pre-configured in PwnBox). Add entry: IP `127.0.0.1`, Port `8080`.

### Installing CA Certificate

Needed so HTTPS traffic routes properly without manual accept prompts.

Burp: browse to `http://burp`, click CA Certificate to download.  
ZAP: (Tools>Options>Network>Server Certificates), Save (Generate creates a new cert).  
Firefox import: `about:preferences#privacy` > View Certificates > Authorities tab > Import > trust the CA for websites and for email users.

## Intercepting Web Requests

Hold an HTTP request before it reaches the server to inspect or edit it, then forward it.

### Intercepting Requests

#### Burp

Default: on. Toggle in (Proxy>Intercept), `Intercept is on/off` button.

Click Forward repeatedly to skip unrelated intercepted requests until the target request appears.

#### ZAP

Default: off (green button = pass-through). Toggle: `CTRL+B` or the top bar button.

HUD: enable via the button at the end of the top bar, controls ZAP features inside the pre-configured browser. Left pane, 2nd button toggles request interception.

`Step` sends the request and keeps breaking on further requests | `Continue` forwards and passes remaining requests through

### Manipulating Intercepted Requests

Tests SQL injection, command injection, upload bypass, auth bypass, XSS, XXE, error handling, deserialization.

Example: a numeric-only front-end field can still accept shell metacharacters if the back end does not validate input.

```text
;ls;
```

## Intercepting Responses

Edit the server's response before it reaches the browser, e.g. to enable or reveal form fields.

### Burp

Enable: (Proxy>Proxy settings), check Intercept Response under Response interception rules. Force a full refresh with `CTRL+SHIFT+R`.

```html
<input type="text" id="ip" name="ip" min="1" max="255" maxlength="100"
    oninput="javascript: if (this.value.length > this.maxLength) this.value = this.value.slice(0, this.maxLength);"
    required>
```

Can persistently unhide disabled/hidden HTML fields via (Proxy>Proxy settings>Response modification rules), e.g. Unhide hidden form fields, without editing each response.

### ZAP

`Step` on an intercepted request also intercepts and shows its response.

HUD light bulb icon (3rd button, left pane): shows/enables hidden or disabled fields without intercepting the response.  
HUD Comments button (add via `+` on left pane): flags positions of HTML comments in the page.

## Automatic Modification

Apply match-and-replace rules to all outgoing requests or incoming responses automatically, instead of editing each one.

### Automatic Request Modification

#### Burp Match and Replace

(Proxy>Proxy settings>HTTP match and replace rules), Add.

Example: Type `Request header`, Match `^User-Agent.*$` (regex), Replace `User-Agent: HackTheBox Agent 1.0`

#### ZAP Replacer

Access: `CTRL+R` or Options>Replacer.

Fields: Match Type (e.g. `Request Header`, adds header if not present), Match String, Replacement String, Enable. Also supports `Request Header String` with a regex pattern.

Initiators tab sets where the rule applies (default: all HTTP(S) messages).

### Automatic Response Modification

(Proxy>Options>Match and Replace), Add. Example: Type `Response body`, Match `type="number"`, Replace `type="text"`, Regex false. Add a second rule: Match `maxlength="3"`, Replace `maxlength="100"`.

Refresh with `CTRL+SHIFT+R`, changes persist across refreshes.

## Repeating Requests

Resend a previously proxied request with edits, without re-intercepting it each time.

### Proxy History

Burp: (Proxy>HTTP History). Shows filtering/sorting, and both Original and Edited request if the request was modified (toggle in the pane header).  
ZAP: bottom History pane (HUD) or History tab (main UI). Shows only the final/modified request.

Both tools also log WebSockets history (async requests).

### Repeating Requests

#### Burp

Send to Repeater: `CTRL+R`. Open Repeater tab: `CTRL+SHIFT+R`. Send button sends the request.

Right-click request > Change Request Method to switch GET/POST without rewriting it.

#### ZAP

Right-click request > Open/Resend with Request Editor. Send button resends; Method dropdown switches HTTP verb.

HUD: click a request in the bottom History pane to open the Request Editor. `Replay in Console` shows the response in the HUD, `Replay in Browser` renders it in the browser.

## Encoding/Decoding

Encode and decode request/response data to interact correctly with the server and inspect encoded values.

### URL Encoding

Required so request data and headers are read correctly; unencoded data can cause server errors.

`space` marks end of data if unencoded | `&` parameter delimiter | `#` fragment identifier

Burp: select text, right-click (Convert Selection>URL>URL-encode key characters) or `CTRL+U`. Can URL-encode as typed (enable via right-click).  
ZAP: auto-encodes request data before sending.

Other types: Full URL-Encoding, Unicode URL encoding.

### Decoding

Decode data a web app has encoded (e.g. a cookie) to inspect the original value.

`HTML` | `Unicode` | `Base64` | `ASCII hex`

Burp: Decoder tab, or Inspector panel (in Proxy/Repeater) for quick encode/decode.  
ZAP: Encoder/Decoder/Hash (`CTRL+E`), auto-decodes known formats in the Decode tab. Add custom tabs via `Add New Tab`.

### Encoding

Re-encode modified data back to its original format (e.g. base64) before sending it.

Burp: chain encodings by selecting a new method in the Decoder output pane.  
ZAP: copy the output text, paste into the input field above to chain.

## Proxying Tools

Route a command-line tool or thick client through a web proxy (`http://127.0.0.1:8080`) to inspect its HTTP traffic.

Proxying slows a tool down. Only proxy when investigating its requests, not for normal use.

### Proxychains

Edit `/etc/proxychains.conf`, comment the last line, add:

```text
#socks4         127.0.0.1 9050
http 127.0.0.1 8080
```

```bash
proxychains -q curl http://<TARGET_IP>:<PORT>
```

`-q` quiet mode, suppresses connection info output.

### Metasploit

```bash
msfconsole
```

Set a proxy on any module with `set PROXIES <TYPE>:<IP>:<PORT>`.

```text
msf6 > use auxiliary/scanner/http/robots_txt
msf6 auxiliary(scanner/http/robots_txt) > set PROXIES HTTP:127.0.0.1:8080
msf6 auxiliary(scanner/http/robots_txt) > set RHOST <TARGET_IP>
msf6 auxiliary(scanner/http/robots_txt) > set RPORT <PORT>
msf6 auxiliary(scanner/http/robots_txt) > run
```

## Burp Intruder

Burp's fuzzer: fuzzes pages, directories, sub-domains, parameters, and values. Free Community version throttled to 1 request/second.

CLI alternatives: ffuf, dirbuster, gobuster, wfuzz.

### Target

Send request to Intruder: right-click in Proxy History > Send to Intruder, or `CTRL+I`. Open Intruder tab: `CTRL+SHIFT+I`.

### Positions

Mark a payload position: wrap the word with `§` or select it and click `Add §`.

Attack type `Sniper`: one payload position, iterates one payload set through it.

Keep the two trailing blank lines at the end of the request or the server returns an error.

### Payloads

#### Payload Position & Payload Type

Payload set choice depends on attack type and number of payload positions marked.

`Simple list` basic wordlist, iterates each line | `Runtime file` loads line-by-line, less memory, use for large wordlists | `Character substitution` char list + replacements, tries all permutations

#### Payload Configuration

Load wordlist: Load button (file) or Add (manual entries, one at a time). Multiple loads append to the same list, for combining wordlists.

Wordlist example: `common.txt` (seclists Discovery/Web-Content).

#### Payload Processing

Rule `Skip if matches regex` with pattern `^\..*$` skips wordlist lines starting with `.`.

#### Payload Encoding

URL-encodes payloads before sending (enabled by default). Encodes: `./^=<>&+?*:;'{}|^`

### Settings

Grep - Match: flags responses containing a string/regex (e.g. `200 OK`). Clear the default list, Add the target string. Disable `Exclude HTTP Headers` if the match is in headers, not body.

Grep - Extract: pulls out and shows only a specific part of a lengthy response instead of all of it.

Resource Pool tab: controls network resource usage, useful for large attacks.

### Attack

Sort results by the grep-match column, Status, or Length to find hits.

Also usable for AD password spraying (OWA, SSL VPN portals, RDS, Citrix, custom AD-auth apps).

## ZAP Fuzzer

ZAP's fuzzer. Not throttled, unlike free Burp Intruder.

### Fuzz

Open: right-click request > (Attack>Fuzz).

#### Locations

Select the target word, click Add (right pane) to mark the fuzz location (green marker).

#### Payloads

`File` wordlist from a file | `File Fuzzers` built-in wordlist databases | `Numberzz` generates number sequences with custom increments

Example: File Fuzzers > dirbuster > `directory-list-1.0.txt`. More databases installable from the ZAP Marketplace.

#### Processors

`Base64 Decode/Encode` | `MD5 Hash` | `SHA-1/256/512 Hash` | `Prefix String` adds a string before the payload | `Postfix String` adds a string after the payload | `Script` runs a custom script on each payload

Used here: `URL Encode`, to avoid server errors from special characters. Preview with Generate Preview.

#### Options

Concurrent Scanning Threads: set higher (e.g. 20) for faster scans, limited by CPU and server connection limits.

`Depth First` tries all payloads on one position before the next position (e.g. all passwords for one user) | `Breadth First` tries each payload across all positions before the next payload (e.g. one password across all users before the next password)

### Start

Sort results by response code for hits. `Size Resp. Body` differing from others may mean a different page. `RTT` indicates time-based attacks (e.g. time-based SQLi).

## Burp Scanner

Pro-only. Crawler builds the site structure, Scanner runs passive and active vulnerability analysis.

### Target Scope

Start a scan via: right-click a request in Proxy History (Scan to configure, or Passive/Active Scan for defaults); New Scan button (Dashboard) on custom targets; or scan items already in scope.

Site map: (Target>Site map).

Add to scope: right-click item > Add to scope. Remove: right-click an in-scope item > Remove from scope (exclude risky endpoints, e.g. logout).  
Scope detail/config: (Target>Scope). Advanced scope control supports regex include/exclude.

Adding the first item to scope prompts the option to restrict all Burp features to in-scope items only.

### Crawler

New Scan (Dashboard) auto-populates in-scope items.

`Crawl` maps linked pages only, no fuzzing for unreferenced pages (use Intruder/Content Discovery for that) | `Crawl and Audit` runs the crawler then the passive and active scanner

Scan config: New (custom) or Select from library (presets, e.g. `Crawl strategy - fastest`).  
Application login tab: provide credentials, or record a manual login sequence, for authenticated crawl coverage.

Progress: Dashboard > Tasks. Results: (Target>Site map), updated after the crawl.

### Passive Scanner

Analyzes already-visited page source for potential vulnerabilities, sends no new requests. Only suggests unverified findings, with a confidence rating per finding.

Start: right-click a target/request > Do passive scan.  
Review: Dashboard > View Details > Issue activity tab, or the Dashboard Issue activity pane. Prioritize High severity + Certain confidence; review all levels for sensitive apps.

### Active Scanner

- Crawls and fuzzes (like dirbuster/ffuf) to find all pages
- Runs a Passive Scan on identified pages
- Sends requests to verify Passive Scan findings
- Runs JavaScript analysis for further vulnerabilities
- Fuzzes insertion points/parameters for XSS, Command Injection, SQL Injection, and other common vulnerabilities

Start: right-click a request > Do active scan, or New Scan > Crawl and Audit.

Audit config: vuln types to test (default all), payload insertion points. Preset example: `Audit checks - critical issues only`. Login details optional, same as Crawl config.

Requests log: View details > Logger tab, or the main Logger tab.  
Filter results by severity/confidence (e.g. High + Certain) in the Issue activity pane.

### Reporting

Export: (Target>Site map), right-click target > (Issue>Report issues for this host). Choose export format and which severity/confidence levels to include.

Don't submit raw scanner output as the final client deliverable. Use it as appendix/raw data only.

## ZAP Scanner

Spider builds the site map, then passive and active scans find vulnerabilities.

### Spider

Builds the site map, similar to Burp's Crawler.

Start: right-click request > (Attack>Spider), or HUD right pane 2nd button (Spider Start).

Prompts to add the site to scope automatically if it is not already in scope.

Progress: HUD Spider button, or main UI Spider tab (auto-switches).  
Results: main UI Sites tab, or HUD right pane 1st button (Sites Tree).

Ajax Spider (HUD right pane, 3rd button): also finds links loaded via JS/AJAX requests, slower but can catch links the normal Spider misses. Run after the normal Spider finishes.

### Passive Scanner

Runs automatically on every response during Spider/browsing, before any Active Scan. Finds issues from source, e.g. missing security headers, DOM-based XSS.

HUD: left pane = alerts on the current page, right pane = all alerts found app-wide. Main UI Alerts tab for full detail.

### Active Scanner

Start: HUD right pane Active Scan button (auto-runs Spider first if not already done), or right-click request > Attack menu.

High alerts usually mean direct compromise potential. Click the alert's URL to see the request/response; replay via HUD or Request Editor.

```text
127.0.0.1&cat /etc/passwd&
```

### Reporting

Generate: (Report>Generate HTML Report), choose save location. Also exports as XML or Markdown.

## Extensions

Add community-built features: Burp via the BApp Store, ZAP via the Marketplace.

### BApp Store

Access: Extensions tab > BApp Store sub-tab. Sort by Popularity.

Some extensions are Pro-only; some need extra dependencies (e.g. `Jython`) installed first.

Example: Decoder Improved adds extra encoders/hashing (e.g. `Hash With>MD5`) in its own tab.

.NET Beautifier, J2EEScan, Software Vulnerability Scanner, Software Version Reporter, Active Scan++, Additional Scanner Checks, AWS Security Checks, Backslash Powered Scanner, Wsdler, Java Deserialization Scanner, C02, Cloud Storage Tester, CMS Scanner, Error Message Checks, Detect Dynamic JS, Headers Analyzer, HTML5 Auditor, PHP Object Injection Check, JavaScript Security, Retire.JS, CSP Auditor, Random IP Address Header, Autorize, CSRF Scanner, JS Link Finder

### ZAP Marketplace

Access: Manage Add-ons button > Marketplace tab.

`Release` stable | `Beta` / `Alpha` may be unstable

Example: FuzzDB Files + FuzzDB Offensive add-ons add wordlists to File Fuzzers, e.g. an OS command injection list at `fuzzdb>attack>os-cmd-execution` (`command_execution-unix.txt`).