# Custom Wordlists & Rules

Generate mutated variants from a wordlist using a custom rule file — good for testing policy-compliant password patterns:

```bash
hashcat --force <WORDLIST> -r <RULE_FILE> --stdout | sort -u > <OUTFILE>
```

Common rule functions worth knowing: `:` (no change) / `c` (capitalize first) / `u` (all upper) / `l` (all lower) / `sXY` (replace X with Y) / `$X` (append X)

Example rule file for typical corporate policy (upper + leet + special):

```
:
c
so0
c so0
sa@
c sa@
$!
$! c so0 sa@
```

Scrape a company's website for likely password terms — spider depth 4, min 6 chars, lowercase:

```bash
cewl https://<TARGET_DOMAIN> -d 4 -m 6 --lowercase -w <OUTFILE>
```

Then combine CeWL output with rules and crack:

```bash
hashcat --force <CEWL_WORDLIST> -r /usr/share/hashcat/rules/best64.rule --stdout | sort -u > <MUT_WORDLIST>
hashcat -a 0 -m <MODE> <HASH_FILE> <MUT_WORDLIST>
```

OSINT → wordlist pipeline: names (pet, spouse, kids), company name, city, interests, birth year/month → base wordlist → apply rules for policy compliance (upper, leet substitutions, trailing digit/symbol, year append).

> [!tip] Policy known → build mask or rules to match it exactly. 12-char min + upper + lower + digit + symbol → `c so0 sa@ $! $2024` style rules on base words, or mask `?u?l?l?l?l?l?d?d?s` variants. Targeted beats generic every time.