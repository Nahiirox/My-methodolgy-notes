# SSH Port Forwarding & SOCKS Tunneling

---

## Local Port Forwarding

Forward a specific remote port to your local machine — use when you know the exact service and port:

```bash
ssh -L <LPORT>:localhost:<RPORT> <USER>@<PIVOT_IP>
```

Multiple ports in one command:

```bash
ssh -L <LPORT1>:localhost:<RPORT1> -L <LPORT2>:localhost:<RPORT2> <USER>@<PIVOT_IP>
```

Verify the forward is active:

```bash
netstat -antp | grep <LPORT>
nmap -v -sV -p<LPORT> localhost
```

---

## Dynamic Port Forwarding (SOCKS)

Use when you don't know which ports/hosts you'll need — creates a full SOCKS proxy that routes all traffic through the pivot:

```bash
ssh -D 9050 <USER>@<PIVOT_IP>
```

Configure proxychains to use it — add to `/etc/proxychains.conf`:

```
socks4  127.0.0.1 9050
```

Scan through the proxy — must use `-sT` (full TCP connect), `-Pn` (no ICMP ping, especially against Windows):

```bash
proxychains nmap -v -Pn -sT <TARGET_IP>
```

Host discovery scan through proxy:

```bash
proxychains nmap -v -sn <CIDR>
```

Run any tool through the proxy by prepending `proxychains`:

```bash
proxychains msfconsole
proxychains xfreerdp /v:<TARGET_IP> /u:<USER> /p:<PASS>
```

> [!warning] Only full TCP connect scans work through proxychains — no SYN/half-open scans, no UDP. Partial packets return garbage results. Use `-sT` always.

> [!warning] ICMP ping doesn't work through proxychains against Windows hosts (Defender blocks it by default). Always use `-Pn` when scanning Windows targets through a proxy.

> [!tip] `socks5` in proxychains.conf gives you auth + UDP support. `socks4` is enough for most TCP-only scan scenarios. Match the type to what your tunnel supports.