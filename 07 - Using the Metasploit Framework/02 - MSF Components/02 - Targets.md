# MSF — Targets

List targets for the loaded exploit:

```
msf6 exploit(...)> show targets
msf6 exploit(...)> info
```

Set a specific target when you know the exact OS/version — more reliable than Automatic:

```
msf6 exploit(...)> set target <INDEX>
```

Automatic = MSF fingerprints the service first then picks. Use it when you're unsure. Set a specific target when Nmap or banner already gave you the exact version — faster and avoids detection from extra probing.

> [!tip] Return address varies by OS version, SP, and language pack — if Automatic fails but the vuln is confirmed, manually enumerate target options and try each relevant one. Comments in the module source often explain what differentiates them.