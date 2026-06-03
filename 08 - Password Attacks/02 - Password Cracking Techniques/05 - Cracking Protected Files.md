# Cracking Protected Files

Hunt encrypted docs by extension — pipe into grep to cut noise:

```bash
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*");do echo -e "\nFile extension: " $ext; find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core";done
```

Hunt SSH private keys by header — no extension needed:

```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

Check if a key is passphrase-protected — prompts if it is, prints pubkey if not:

```bash
ssh-keygen -yf <KEYFILE>
```

Old PEM format tells you directly: `Proc-Type: 4,ENCRYPTED` in the header = encrypted.

Crack SSH key:

```bash
ssh2john.py <KEYFILE> > ssh.hash
john --wordlist=<WORDLIST> ssh.hash
john ssh.hash --show
```

Crack Office doc:

```bash
office2john.py <FILE>.docx > doc.hash
john --wordlist=<WORDLIST> doc.hash
```

Crack PDF:

```bash
pdf2john.py <FILE>.pdf > pdf.hash
john --wordlist=<WORDLIST> pdf.hash
```

> [!tip] `locate *2john*` — full list of JtR extraction scripts on the system, covers zip, rar, keepass, bitlocker, gpg, vnc, and more. Same extract → john pipeline for all of them.