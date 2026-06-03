# Password Policies

Useful offensively for building mutation lists and setting spray expectations.

Complexity policies (8 chars, upper/lower/number/special) still pass: `Welcome1`, `Inlanefreight01!`, `Winter2024!` — complexity ≠ strength.

Expiration policies produce predictable increments: `Password01!` → `Password02!` on next cycle. Factor this into mutation rules.

Blacklist what orgs typically ban — meaning users avoid these exact strings but mutate them, so your wordlist should include both:

- Company name + variations
- Month/season names
- "welcome", "password", "123456"
- Common patterns: `<CompanyName><Year>!`, `<Season><Year>@`

# Password Managers

Offensive relevance only:

KeePass `.kdbx` files on a compromised host = high-value target. Crack with hashcat mode 13400:

```bash
keepass2john <FILE>.kdbx > <OUTFILE>
hashcat -m 13400 <OUTFILE> <WORDLIST>
```

Cloud vaults (Bitwarden, 1Password, LastPass) are AES-256 encrypted with a master-password-derived key. No direct offline attack path without the master password or a phishing/keylogger approach.

> [!tip] Hunt for .kdbx during share enum and post-compromise filesystem search — Snaffler flags them as {Red}.