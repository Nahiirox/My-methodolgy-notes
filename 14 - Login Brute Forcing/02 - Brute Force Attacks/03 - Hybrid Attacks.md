# Hybrid Attacks

Dictionary first, then targeted mutations of those same words (append year, special char, increment). Covers the predictable `Summer2023 → Summer2024 / Summer2023!` pattern without a full brute.

Filter a known wordlist down to only policy-compliant candidates before attacking — chain grep stages, one pass per requirement:

```bash
grep -E '^.{8,}$' <WORDLIST> | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' > <OUTFILE>
```

Adjust the chain to match whatever policy you've enumerated. Common additions: `([!@#$%^&*].*){2,}` for special char count, `^.{6,12}$` for max length.

Check how much you cut before running — if it's still too large, tighten the policy filters:

```bash
wc -l <OUTFILE>
```

> [!tip] Know the policy before filtering — even partial policy info (e.g. "must have a number") halves the keyspace fast

**Credential stuffing** = same workflow but creds come from breach dumps / leaked lists, not generated wordlists. Assume password reuse across services — if you crack one account, try those creds everywhere else you've found that user.