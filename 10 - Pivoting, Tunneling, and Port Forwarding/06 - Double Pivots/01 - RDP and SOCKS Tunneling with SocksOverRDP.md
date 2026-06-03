# SocksOverRDP

SOCKS tunneling over RDP's DVC (Dynamic Virtual Channels) — use when you're locked into Windows-only environment with no SSH. Chains through RDP connections to reach deeper network segments.

Requires: SocksOverRDP binaries + Proxifier portable on Windows attack host.

---

## Setup Chain

On the first Windows foothold — register the DVC plugin:

```cmd
regsvr32.exe SocksOverRDP-Plugin.dll
```

Connect via `mstsc.exe` to the next hop (172.16.5.19) — plugin intercepts the RDP connection and starts SOCKS listener on `127.0.0.1:1080`.

On the second Windows host (172.16.5.19) — run the server component with admin privileges:

```
SocksOverRDP-Server.exe
```

Verify SOCKS listener is up on the second hop:

```cmd
netstat -antb | findstr 1080
```

Configure Proxifier on the first hop to route all traffic through `127.0.0.1:1080`. Then `mstsc.exe` launched through Proxifier will pivot connections through the RDP tunnel into the deeper network.

---

## Flow

`Attack host → RDP → Hop1 (SocksOverRDP plugin) → RDP tunnel → Hop2 (SocksOverRDP-Server + SOCKS) → internal network`

> [!tip] If RDP sessions are sluggish, set Performance to `Modem` in mstsc.exe Experience tab — helps significantly when chaining multiple RDP hops.

> [!warning] `regsvr32.exe` loading a DLL will likely trigger AV/EDR on hardened hosts. Check defenses before attempting. This technique is also very noisy in RDP session logs.