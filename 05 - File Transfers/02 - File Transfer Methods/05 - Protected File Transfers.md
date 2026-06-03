# Protected File Transfers

Encrypt before exfil — NTDS.dit, cred files, scan output with PII all need this. Use a unique key per engagement, never reuse.

## Windows — Invoke-AESEncryption.ps1

Transfer the script to target via any method, then import and encrypt:

```powershell
Import-Module .\Invoke-AESEncryption.ps1
Invoke-AESEncryption -Mode Encrypt -Key "<PASS>" -Path .\<FILE>
```

Output: `<FILE>.aes` in the same directory. Decrypt on the other end:

```powershell
Invoke-AESEncryption -Mode Decrypt -Key "<PASS>" -Path .\<FILE>.aes
```

---

## Linux — OpenSSL

Encrypt with AES-256 + PBKDF2 (stronger key derivation, resists brute-force):

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in <FILE> -out <FILE>.enc
```

Decrypt:

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in <FILE>.enc -out <FILE>
```

> [!tip] `-pbkdf2 -iter 100000` matters — without it OpenSSL uses a weak legacy KDF that cracks fast. Always include both flags.

> [!warning] Don't exfil real PII/financial data/trade secrets unless the client explicitly scoped it. For DLP testing, create dummy files that mimic the format instead.