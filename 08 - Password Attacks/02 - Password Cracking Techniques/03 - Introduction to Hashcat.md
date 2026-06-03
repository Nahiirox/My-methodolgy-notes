# Hashcat

Identify hash mode with hashid — `-m` flag gives hashcat mode number directly:

```bash
hashid -m '<HASH>'
```

When hashid is ambiguous, check hashcat example hashes page — format prefix (e.g. `$1$`, `$6$`, `$krb5tgs$`) is usually enough to pin the mode.

Dictionary attack (`-a 0`):

```bash
hashcat -a 0 -m <MODE> <HASH_FILE> <WORDLIST>
```

Dictionary + rules — try `best64.rule` first, `rockyou-30000.rule` if that fails, `dive.rule` as nuclear option:

```bash
hashcat -a 0 -m <MODE> <HASH_FILE> <WORDLIST> -r /usr/share/hashcat/rules/best64.rule
```

Mask attack (`-a 3`) — use when you know the password pattern:

```bash
hashcat -a 3 -m <MODE> <HASH_FILE> '<MASK>'
```

Built-in charsets: `?l` lowercase / `?u` upper / `?d` digit / `?s` special / `?a` all printable / `?h` hex lower / `?H` hex upper

Custom charset — define with `-1` through `-4`, reference as `?1` through `?4`:

```bash
hashcat -a 3 -m <MODE> <HASH_FILE> -1 '<CHARSET>' '?1?1?1?1?d?d'
```

Common exam masks — `?u?l?l?l?l?d?s` (Word + digit + symbol), `?u?l?l?l?l?l?d?d` (corp password pattern), `?d?d?d?d?d?d?d?d` (8-digit PIN).

> [!tip] wordlist alone fails → wordlist + `best64.rule` → wordlist + `rockyou-30000.rule` → mask if you know the pattern. Don't jump straight to mask without trying rules first.