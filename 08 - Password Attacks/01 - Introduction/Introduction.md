# Password Attacks — Intro

66% password reuse across platforms → valid cred anywhere = try it everywhere. Same username format, same password, all exposed services.

55% of users don't change passwords after a breach → old dehashed creds are still live. Always try breach data before spraying.

Check target email addresses for breach exposure before building lists:

```
haveibeenpwned.com
```

Top spray candidates in order: `123456`, `qwerty`, `password`, `Welcome1`, `<Company><Year>`, `<name>`, `<pet/kid name>`. Custom rules: append `!`, `1`, `123`, current year — covers the "complexity satisfied" crowd.

> [!tip] 36% of users now use password managers → those accounts have strong unique passwords, not worth brute forcing. Prioritize accounts that look like they belong to non-technical users.