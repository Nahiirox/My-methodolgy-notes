# John the Ripper

Identify unknown hash format — `-j` adds JtR format name to output:

```bash
hashid -j <HASH>
```

When hashid is ambiguous, context tells you more than the tool. Also check PentestMonkey hash examples list.

Single crack mode — use when you have a `/etc/passwd`-style file with username + GECOS info, targets Linux creds:

```bash
john --single <HASH_FILE>
```

Wordlist mode:

```bash
john --wordlist=<WORDLIST> <HASH_FILE>
john --wordlist=<WORDLIST> --rules <HASH_FILE>
```

Incremental mode (Markov-chain brute-force, last resort):

```bash
john --incremental <HASH_FILE>
```

Specify format explicitly when auto-detect fails:

```bash
john --format=<FORMAT> --wordlist=<WORDLIST> <HASH_FILE>
```

Show already-cracked results from a session:

```bash
john --show <HASH_FILE>
```

File cracking — convert first, then crack:

```bash
<tool>2john <FILE> > <HASH_FILE>
john --wordlist=<WORDLIST> <HASH_FILE>
```

Find all converters on the box:

```bash
locate *2john*
```

Key converters to remember: `ssh2john`, `keepass2john`, `zip2john`, `rar2john`, `pdf2john`, `office2john`, `pfx2john`, `putty2john`.

> [!tip] JtR format names differ from hashcat mode numbers — `hashid -j` gives you the JtR name directly. For hashcat, use `hashid` without `-j` and cross-reference the hashcat example hashes page.