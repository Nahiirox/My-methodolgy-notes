# Proxying Tools

Route any CLI tool through Burp/ZAP — inspect, modify, replay its requests like browser traffic.

`/etc/proxychains.conf` — comment out the socks4 line, add:

```
http 127.0.0.1 8080
```

Then prepend any command with `proxychains -q`:

```bash
proxychains -q curl http://<TARGET>:<PORT>
```

MSF module → set proxy before `run`:

```bash
set PROXIES HTTP:127.0.0.1:8080
```

> [!tip] Only proxy tools when you're actively inspecting traffic — it slows them down noticeably, don't leave it on by default.