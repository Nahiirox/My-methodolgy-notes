# Using Web Proxies

Web proxies sit between a browser/app and a back-end server as a MITM for HTTP/HTTPS. They capture every request/response, intercept and modify traffic mid-flight, and replay requests with changes.

Beyond capture/replay, they support:

* Vulnerability scanning
* Fuzzing
* Crawling
* Application mapping
* Request analysis
* Configuration testing
* Code review support

The two tools are:

* **Burp Suite** — best UI, built-in Chromium; Pro-only features include active scanner, fast Intruder, and some extensions.
* **OWASP ZAP** — fully free/open-source, no throttling, HUD, and gaining parity.

Learn both: the concepts are the same, and you can choose based on the engagement.

## Setting Up

Both run on Windows, macOS, and Linux. They are pre-installed on Pwnbox, Parrot, and Kali.

Both are Java apps. A JRE is bundled, but the following also works cross-platform:

```bash
java -jar </path/to/burpsuite.jar>
```

* **Burp startup:** temporary project → Use Burp Defaults → Start Burp.
  * Community cannot save projects.
  * Pro can persist projects and active scans.
* **ZAP startup:** run `zaproxy` or launch it from the menu → select “No, do not persist” for a temporary session.

### Dark Theme

* Burp: **Settings > User interface > Display**
* ZAP: **Tools > Options > Display > Flat Dark**

## Proxy Setup

The fastest path is using the pre-configured browser, where the proxy and CA certificate are already configured.

* Burp: **Proxy > Intercept > Open Browser**
* ZAP: click the Firefox icon at the end of the top bar

### Configure a Real Browser

Point the browser at the proxy. Both default to:

```
127.0.0.1:8080
```

Change this under:

* Burp: **Proxy > Proxy settings > Proxy listeners**
* ZAP: **Tools > Options > Network > Local Servers/Proxies**

**FoxyProxy** makes switching one click:

1. Open **Options > Add**.
2. Set the IP to `127.0.0.1`.
3. Set the port to `8080`.
4. Name it Burp or ZAP.

### CA Certificate

Without a CA certificate, HTTPS traffic stalls on trust prompts.

* Burp: browse to `http://burp` → **CA Certificate**
* ZAP: **Tools > Options > Network > Server Certificates > Save**
  * **Generate** creates a new certificate.

Install the certificate in Firefox:

1. Browse to `about:preferences#privacy`.
2. Select **View Certificates**.
3. Open the **Authorities** tab.
4. Select **Import**.
5. Enable “Trust this CA to identify websites” and email users.

## Intercepting Web Requests

### Burp

Open **Proxy > Intercept** and toggle **Intercept is on/off**.

Use:

* **Forward** to send the held request.
* **Drop** to discard it.

### ZAP

Interception is off by default.

* Use the green button or `CTRL+B` to toggle interception.
* Use **Step**, to the right of the break button, to forward traffic.

ZAP’s **HUD** puts controls in the browser:

* The second button on the left pane toggles interception.
* On a caught request:
  * **Step** sends the request and breaks on the next item, letting you inspect each step.
  * **Continue** sends the rest of the traffic through.

### Manipulation Workflow

Intercept → edit the request inline → forward it.

Classic example: a page restricts an IP field to numbers with front-end JavaScript.

1. Intercept the POST request.
2.  Change:

    ```
    ip=1
    ```

    to:

    ```
    ip=;ls;
    ```
3. Forward the request.
4. If the response contains the output of `ls`, you have command injection.

The same mechanism drives SQL injection, XSS, XXE, upload bypasses, authentication bypasses, and deserialization tests.

## Intercepting Responses

Intercept a server response before the browser renders it so you can rewrite the HTML for testing.

### Burp

1. Go to **Proxy > Proxy settings > Response interception rules**.
2. Enable **Intercept Response**.
3. Force refresh with `CTRL+SHIFT+R`.
4. Forward the request.
5.  Edit the held response, for example:

    ```html
    type="number"
    ```

    to:

    ```html
    type="text"
    ```

    Or:

    ```html
    maxlength="3"
    ```

    to:

    ```html
    maxlength="100"
    ```
6. Forward the response again.

This also enables disabled buttons persistently for each page load.

### ZAP

Click **Step** on an intercepted request. The response is caught automatically.

Edit it, then click **Continue**.

### ZAP HUD Shortcuts

On the left pane:

* Light-bulb button: show hidden fields and enable disabled fields without intercepting.
* **Comments** button: add comments with `+`; HTML comment positions are flagged inline with hover content.

Burp equivalents are under:

* **Proxy > Proxy settings > Response modification rules**
  * Example: “Unhide hidden form fields”

## Automatic Modification

Rules can rewrite every request or response in flight without manual interception.

### Requests: Match and Replace

#### Burp

Go to **Proxy > Proxy settings > HTTP match and replace > Add**.

| Setting | Value                              |
| ------- | ---------------------------------- |
| Type    | `Request header`                   |
| Match   | `^User-Agent.*$`                   |
| Replace | `User-Agent: HackTheBox Agent 1.0` |
| Regex   | true                               |

#### ZAP

Open Replacer with `CTRL+R`, then select **Add**.

| Setting      | Value                                      |
| ------------ | ------------------------------------------ |
| Match Type   | `Request Header (will add if not present)` |
| Match String | `User-Agent`                               |
| Replacement  | `HackTheBox Agent 1.0`                     |

The **Initiators** tab scopes where the rule applies. By default, it applies to all HTTP(S) traffic.

### Responses

Use the same approach:

| Setting | Value           |
| ------- | --------------- |
| Type    | `Response body` |
| Match   | `type="number"` |
| Replace | `type="text"`   |

No regex is needed for exact strings.

The change survives page refreshes, so the front-end bypass remains active.

## Repeating Requests

Resend a historical request with edits and view the response in the tool, without another interception cycle.

History is available in:

* Burp: **Proxy > HTTP History**
* ZAP: bottom **History** pane or HUD

Both also keep WebSocket history, which is useful for asynchronous updates and data fetching.

Burp displays both **Original** and **Edited** versions of a modified request through the pane header toggle. ZAP shows only the final version.

### Burp

* Press `CTRL+R` on a request to send it to Repeater.
* Open Repeater with `CTRL+SHIFT+R`.
* Edit the request and click **Send**.
* Right-click and choose **Change Request Method** to switch POST ↔ GET without rewriting.

### ZAP

* Right-click a request.
* Select **Open/Resend with Request Editor**.
* Edit the request and click **Send**.
* Use the Method drop-down for any HTTP verb.

### HUD

Click a history entry and select:

* **Replay in Console**
* **Replay in Browser**

The browser option renders the result.

## Encoding and Decoding

URL-encode anything that would break request parsing:

* Space: end-of-data
* `&`: parameter delimiter
* `#`: fragment

### Burp

In Repeater:

* Select text and press `CTRL+U`.
* Or choose **Convert Selection > URL**.
* You can enable encode-as-you-type.

Burp includes:

* **Decoder** tab
  * Input → **Encode as** / **Decode as**
  * The output pane can be re-encoded directly with another scheme.
* **Inspector** in newer versions
  * Inline URL and Base64 decoding in Proxy and Repeater.

### ZAP

ZAP auto-encodes request data behind the scenes.

Open **Encoder/Decoder/Hash** with `CTRL+E`.

* The **Decode** tab automatically tries schemes.
* Custom tabs can mix selected encoders.

Both tools support HTML, Unicode, Base64, and ASCII hex encoding.

### Typical Flow

Find a Base64 cookie:

```
eyJ1c2VybmFtZSI6Imd1ZXN0IiwgImlzX2FkbWluIjpmYWxzZX0=
```

Decode it:

```json
{"username":"guest", "is_admin":false}
```

Change the value to admin/true, re-encode it, and paste it into Repeater or the Request Editor.

## Proxying Tools

Route CLI tools and thick clients through the proxy. This slows them down, so only do it when investigating.

### proxychains

Edit `/etc/proxychains.conf`:

1. Comment out the SOCKS line.
2.  Add:

    ```
    http 127.0.0.1 8080
    ```

Run with `-q` for quiet mode:

```bash
proxychains -q curl http://SERVER_IP:PORT
```

### Metasploit

Set a proxy per module:

```
use auxiliary/scanner/http/robots_txt
set PROXIES HTTP:127.0.0.1:8080
set RHOST <TARGET_IP>
set RPORT <PORT>
run
```

For any other tool, find its proxy setting and point it at `127.0.0.1:8080`.

## Burp Intruder

Burp’s fuzzer supports directory, subdomain, parameter, and value fuzzing. It can replace `ffuf`, `gobuster`, and `wfuzz` for many jobs.

{% hint style="warning" %}
Burp Community is throttled to 1 request per second. CLI tools can reach roughly 10k requests per second, so Community is suitable only for short queries. Burp Pro is unlimited and among the best fuzzers available.
{% endhint %}

Press `CTRL+I` to send a Proxy History request to Intruder.

{% stepper %}
{% step %}
## Positions

Wrap the fuzz target with `§`, or select it and use **Add §**.

Directory fuzzing example:

```http
GET /§DIRECTORY§/
```

Keep the two trailing blank lines.
{% endstep %}

{% step %}
## Payloads

Set payload set 1 for each position, tied to the selected attack type:

* **Sniper**: one position
* **Cluster Bomb**: several positions

Payload types include:

* **Simple List**: basic lists
* **Runtime file**: streams wordlists line-by-line, preserving memory for huge lists
* **Character Substitution**: permutations

Load:

```
/opt/useful/seclists/Discovery/Web-Content/common.txt
```

Multiple loads append, allowing you to combine or customize lists.
{% endstep %}

{% step %}
## Payload Processing

Add per-item rules.

For example, skip dot-prefixed items with:

```regex
^\..*$
```

You can also add suffix extensions, filter items, and more.
{% endstep %}

{% step %}
## Payload Encoding

Use the URL-encode toggle per character set. Leave it enabled.
{% endstep %}

{% step %}
## Settings

Set retries to `0` if desired.

For **Grep - Match**:

1. Clear the defaults.
2.  Add:

    ```
    200 OK
    ```
3. Disable **Exclude HTTP Headers**.

The results gain a flag column for sorting.

**Grep - Extract** pulls only a slice of long responses.

Use **Resource Pool** to cap network usage for large attacks.
{% endstep %}
{% endstepper %}

Start the attack and sort by flag, status, or length. An endpoint such as `/admin/` may appear with a `200` response.

The same mechanism supports credential brute-forcing and AD password spraying against OWA, VPN, RDS, Citrix, and custom AD-authenticated portals.

## ZAP Fuzzer

ZAP is the free alternative to Intruder. It has no paid tier and no throttle, though it has slightly fewer features.

Right-click a request and select **Attack > Fuzz**.

{% stepper %}
{% step %}
## Locations

Select text, such as `test` in the path, then click **Add**.

A green marker and payload window appear.
{% endstep %}

{% step %}
## Payloads

Select **Add**, then choose:

* **File**: your own wordlist
* **File Fuzzers**: built-in databases, including dirbuster directory lists
* **Numberzz**: number sequences with increments
{% endstep %}

{% step %}
## Processors

Apply per-payload transforms:

* Base64 encode/decode
* MD5/SHA-1/256/512
* URL encode/decode
* Prefix/Postfix String
* Script

Use URL Encode to prevent server errors with special characters.

**Generate Preview** shows the final payload shape.
{% endstep %}

{% step %}
## Options

Set **Concurrent Scanning Threads**, such as `20`, bounded by CPU and server limits.

Choose an attack order:

* **Depth first**: all words for one position before the next, such as every password for one user.
* **Breadth first**: each word across all positions, such as one password attempted for every user.
{% endstep %}
{% endstepper %}

Start the Fuzzer and sort by:

* Response code, such as `200`
* **Size Resp. Body**, where a different size may indicate a different page
* RTT, useful for time-based SQL injection detection

## Burp Scanner

Burp Scanner is Pro-only. It combines crawling with passive and active scanning.

### Target Scope

Set scope first:

1. Go to **Target > Site map**.
2. Right-click an item Burp has seen.
3. Select **Add to scope**.

Burp can then restrict all features to in-scope items only.

Remove dangerous items such as logout endpoints from scope, or use advanced scope controls with regex include/exclude rules.

### Crawler

Go to **Dashboard > New Scan > Crawl**.

The crawler follows links and forms to map the site. It does **not** fuzz unreferenced paths; use Intruder or Content Discovery for that.

Use:

* Custom scan configuration for speed, limits, and login behavior.
* Presets from the library, such as “Crawl strategy - fastest”.
* The **Application login** tab to provide credentials or record a manual login for authenticated coverage.

### Passive Scan

Passive scanning sends no new requests. It analyzes already visited pages for issues such as missing headers and potential DOM XSS.

Run it with:

* Right-click → **Do passive scan**

Results appear under:

* **Dashboard > Issue activity**

Prioritize High/Certain findings.

### Active Scan

Start it through:

* Right-click → **Do active scan**
* **New Scan > Crawl and Audit**

The full sequence is:

1. Crawl and fuzz pages.
2. Passively scan everything.
3. Verify passively discovered issues with requests.
4. Analyze JavaScript.
5. Fuzz insertion points and parameters for XSS, command injection, and SQL injection.

Audit configurations select vulnerability classes and insertion points. A preset example is “critical issues only”.

Watch live requests in the **Logger** tab.

Filter **Issue activity** to High/Certain findings. For example, Burp may identify OS command injection on an `ip` parameter and provide the advisory plus request/response proof.

### Reporting

1. In **Site map**, right-click the host.
2. Select **Issue > Report issues for this host**.
3. Choose the format and contents.

HTML reports open in a browser with severity and confidence summaries, PoCs, and remediation guidance.

Use this as appendix material for the real report, never as a raw-tool deliverable.

## ZAP Scanner

ZAP provides the free equivalent: Spider, passive scanning, and active scanning.

### Spider

* Right-click a request → **Attack > Spider**
* In the HUD, use the second button on the right pane.

ZAP automatically offers to add the site to scope.

Results appear in the **Sites** tab tree or through the HUD **Sites Tree** button.

Use **Ajax Spider**, the third button on the right pane, after the normal Spider to catch links requested via JavaScript AJAX after page load. It adds coverage at the cost of time.

### Passive Scanner

The passive scanner runs automatically on every Spider result and response.

Alerts populate as scanning continues:

* HUD alerts:
  * Left: current page
  * Right: whole application
* Main UI: **Alerts** tab, including details and affected pages

### Active Scan

Use the **Active Scan** button in the right pane.

If no site tree exists, ZAP runs Spider first.

The active scan attacks identified pages and parameters. Watch progress and sent requests in the UI.

High alerts are compromise-capable findings. For example, a Remote OS Command Injection alert may show:

```
127.0.0.1&cat /etc/passwd&
```

With evidence such as:

```
root:x:0:0...
```

Click through to the exact request and response, then replay it.

### Reporting

Use:

* **Report > Generate HTML Report**

XML and Markdown formats are also available.

## Extensions

### Burp BApp Store

Open the **Extensions** tab and sort by Popularity.

Some extensions are Pro-only, and some require Jython or other dependencies first.

Demo:

* **Decoder Improved** adds encoders and hashes.
* Example: **Hash With > MD5**

Extensions worth exploring:

* Active Scan++
* Autorize
* Backslash Powered Scanner
* Java Deserialization Scanner
* JS Link Finder
* Retire.JS
* Software Version Reporter
* CSRF Scanner
* Headers Analyzer
* CSP Auditor
* CMS Scanner
* AWS Security Checks
* .NET Beautifier
* J2EEScan

### ZAP Marketplace

Open:

* **Manage Add-ons > Marketplace**

Add-ons have Release, Beta, and Alpha stability grades.

Demo:

1. Install **FuzzDB Files**.
2. Install **FuzzDB Offensive**.
3.  New File Fuzzers wordlists appear, such as:

    ```
    fuzzdb > attack > os-cmd-execution > command_execution-unix.txt
    ```
4. Run the fuzzer with the list.

Command injection can appear through multiple payload variants, which is useful against WAFs.
