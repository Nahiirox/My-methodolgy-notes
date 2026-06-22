# Brute Forcing — Hydra, Medusa & Custom Wordlists

Pre-installed on Parrot/Kali. SecLists: `/usr/share/seclists/` or https://github.com/danielmiessler/SecLists

Key wordlists:

- Passwords: `/usr/share/seclists/Passwords/Common-Credentials/rockyou.txt`, `2023-200_most_used_passwords.txt`, `darkweb2017-top10000.txt`
- Usernames: `/usr/share/seclists/Usernames/top-usernames-shortlist.txt`, `xato-net-10-million-usernames.txt`
- Default creds: `/usr/share/seclists/Passwords/Default-Credentials/default-passwords.txt`

---

## Hydra

Basic syntax:

```bash
hydra [login_options] [password_options] [attack_options] service://target
```

Key flags: `-l` single user / `-L` user list, `-p` single pass / `-P` pass list, `-t` threads, `-f` stop on first hit, `-s` non-default port, `-V` verbose each attempt.

Basic HTTP auth (`http-get`):

```bash
hydra -l <USER> -P <WORDLIST> <TARGET> http-get / -s <PORT>
```

Web login form (`http-post-form`) — inspect HTML first for field names, error string:

```bash
hydra -L <USERLIST> -P <PASSLIST> <TARGET> -s <PORT> -f http-post-form "/<PATH>:<USER_FIELD>=^USER^&<PASS_FIELD>=^PASS^:F=<FAIL_STRING>"
```

`F=<string>` = fail if string present in response. `S=<string>` or `S=302` = succeed if string/code present. Use `S=` when error string isn't consistent; use `F=` (preferred) when error message is reliable.

SSH:

```bash
hydra -l <USER> -P <WORDLIST> ssh://<TARGET> -t 4
```

FTP on non-default port with verbose:

```bash
hydra -L <USERLIST> -P <PASSLIST> -s <PORT> -V ftp://<TARGET>
```

RDP spray:

```bash
hydra -l <USER> -P <WORDLIST> rdp://<TARGET> -t 4 -W 3
```

Generate passwords inline (6-8 chars, lower+upper+digits):

```bash
hydra -l <USER> -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 <TARGET> rdp
```

Multiple targets from file:

```bash
hydra -l root -p toor -M targets.txt ssh
```

---

## Medusa

```bash
medusa -h <TARGET> -u <USER> -P <WORDLIST> -M ssh -t 3
```

Key flags: `-h` single host / `-H` host list, `-u` single user / `-U` user list, `-p` single pass / `-P` pass list, `-M` module, `-t` threads, `-f` stop on hit (current host) / `-F` stop on hit (any host), `-n` non-default port, `-e ns` try empty password + username-as-password.

SSH:

```bash
medusa -h <TARGET> -n <PORT> -u <USER> -P <WORDLIST> -M ssh -t 3
```

FTP (local, after SSH pivot):

```bash
medusa -h 127.0.0.1 -u <USER> -P <WORDLIST> -M ftp -t 5
```

HTTP GET basic auth:

```bash
medusa -H <HOSTLIST> -U <USERLIST> -P <PASSLIST> -M http -m GET
```

Empty/default pass check:

```bash
medusa -h <TARGET> -U <USERLIST> -e ns -M <MODULE>
```

---

## Filtering Wordlists for Password Policies

Chain grep to filter by policy (min length + complexity):

```bash
grep -E '^.{8,}$' <WORDLIST> | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' > filtered.txt
```

Two special chars from set:

```bash
grep -E '([!@#$%^&*].*){2,}' filtered.txt > final.txt
```

---

## Custom Wordlist Generation

**Username Anarchy** — generates username variations from a full name:

```bash
git clone https://github.com/urbanadventurer/username-anarchy.git
./username-anarchy <FIRSTNAME> <LASTNAME> > usernames.txt
```

**CUPP** — builds personalized password list from OSINT (birthdate, pets, partner, company, interests):

```bash
sudo apt install cupp -y
cupp -i
```

Then filter CUPP output against the target's password policy with grep before using in Hydra/Medusa.

> [!tip] Before running Hydra on a login form, manually submit a failed login and check the Network tab — confirms the POST path, field names, and exact failure string to use in `F=`.

> [!warning] High thread counts (`-t`) on SSH/RDP trigger lockouts and IDS alerts. Keep `-t 4` or lower for remote services. Use password spraying (one pass, many users) when lockout policy is unknown.