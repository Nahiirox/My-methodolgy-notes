# Password Cracking — Intro

Hash speed varies wildly by algorithm — determines whether brute-force is viable or you stick to wordlist:

MD5 → ~5M/s on a laptop. DCC2 → ~10K/s. Know the mode before deciding how to attack.

<9 char passwords = viable brute-force target. Anything longer → wordlist only unless you have a strong mask.

Generate a hash to verify a cracked result or test a format:

```bash
echo -n '<PASS>' | md5sum
echo -n '<PASS>' | sha256sum
```

Wordlists: `/usr/share/wordlists/rockyou.txt` for general use, SecLists for targeted/curated lists.

> [!tip] Salted hashes won't hit rainbow tables — don't bother. Go straight to hashcat with the right mode and a wordlist.