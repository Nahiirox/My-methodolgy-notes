# sshuttle

No proxychains config needed — sshuttle handles iptables routing automatically. Traffic to the specified subnet routes through the SSH tunnel transparently. Any tool runs normally without `proxychains` prefix.

Install:

```bash
sudo apt-get install sshuttle
```

Route a subnet through a pivot via SSH:

```bash
sudo sshuttle -r <USER>@<PIVOT_IP> <CIDR> -v
```

After running, scan or connect directly — no proxychains, no special flags:

```bash
sudo nmap -v -A -sT -p3389 <TARGET_IP> -Pn
xfreerdp /v:<TARGET_IP> /u:<USER> /p:<PASS>
```

sshuttle adds iptables `REDIRECT` rules on your attack host that intercept any traffic destined for the specified CIDR and send it through the SSH tunnel. Removed automatically when you kill sshuttle.

> [!warning] Requires `sudo` — needs root to modify iptables. Also requires Python3 on the pivot host (runs a server component remotely over SSH).

> [!tip] sshuttle is the cleanest pivot method for interactive work — RDP, web browsing, curl, everything just works without prefixing `proxychains`. Use SSH `-D` + proxychains when you need to proxy non-TCP or need finer control. Use sshuttle when you just want to "be on" the pivot network.

Limitation: SSH only. No TOR, no HTTPS proxy. If the pivot doesn't have SSH exposed or you need a non-SSH tunnel, fall back to proxychains + SSH `-D` or Chisel.