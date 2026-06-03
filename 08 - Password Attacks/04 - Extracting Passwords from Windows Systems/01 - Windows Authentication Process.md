# Windows Authentication — Credential Storage

Key locations to know for dumping:

SAM — local account hashes, requires SYSTEM: `%SystemRoot%\system32\config\SAM` → `HKLM\SAM`

NTDS.dit — domain account hashes + all AD objects, DC only: `%SystemRoot%\ntds.dit`

Credential Manager — per-user stored creds (network, web, app), encrypted: `C:\Users\<USER>\AppData\Local\Microsoft\[Vault/Credentials]\`

LSASS (`%SystemRoot%\System32\Lsass.exe`) holds auth material in memory — this is the dump target for live creds, hashes, and Kerberos tickets.

Auth flow that matters operationally: local non-domain login → Msv1_0.dll → SAM. Domain login → Kerberos.dll or NTLM → NTDS.dit on DC.

> [!tip] SAM is partially encrypted on disk via SYSKEY on any modern Windows — raw file copy isn't enough, use reg save or VSS methods to pull usable hashes.