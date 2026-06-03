# Password Spraying — Overview & Considerations

One password → full user list → delay → repeat. Never multiple passwords per user in quick succession.

**Get the password policy before spraying.** Enumerate it once you have any foothold. Without it: assume 5 bad attempts → lockout, wait 2-3 hours between rounds minimum. One spray attempt with one common password is the safe "hail mary" if you're flying blind.

> [!warning] Spraying without knowing the lockout threshold can lock hundreds of production accounts. If in doubt, ask the client for the policy — that's always an option.

---

## Building the User List

Kerbrute + jsmith.txt from statistically-likely-usernames repo covers ~40-60% of valid accounts in most domains. Supplement with LinkedIn scraping.

If standard lists yield nothing → hunt PDF metadata. Author field often leaks internal username format:

```bash
exiftool <FILE>.pdf | grep -i author
```

Got a pattern like `F9L8` (fixed-length GUID, A-Z + 0-9) → generate the full space:

```bash
for x in {{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}; do echo $x; done > users.txt
```

Feed generated list into Kerbrute userenum to validate before spraying — don't spray against guesses.

> [!tip] PDF metadata → username schema is a high-yield move when AD enum hits a wall. Check `filetype:pdf inurl:<DOMAIN>` before giving up on user list building.