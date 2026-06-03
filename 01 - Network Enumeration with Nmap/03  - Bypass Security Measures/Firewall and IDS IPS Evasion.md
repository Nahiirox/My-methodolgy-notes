# Nmap — Firewall & IDS/IPS Evasion

Filtered = dropped (no response). Rejected = ICMP type 3 back (fast). Both show `filtered` but behavior differs — `--packet-trace` tells you which.

ACK scan to map firewall rules — firewalls pass ACK packets more readily than SYN, so `unfiltered` in `-sA` output means the port exists and the rule only blocks inbound SYN:

```bash
sudo nmap <TARGET> -p <PORTS> -sA -Pn -n --disable-arp-ping --packet-trace
```

---

### Decoys

Spoof scan origin — your real IP is randomly placed among the decoys. Decoys must be live hosts or SYN-flood protections may kick in:

```bash
sudo nmap <TARGET> -p <PORT> -sS -Pn -n --disable-arp-ping -D RND:5
```

Spoof a specific source IP — use when a subnet is allowed but yours isn't. `-e` sets the outgoing interface:

```bash
sudo nmap <TARGET> -p <PORT> -sS -Pn -n -S <SPOOF_IP> -e <IFACE>
```

---

### DNS Source Port Bypass

Port filtered from normal source → try sourcing from port 53. Many firewalls trust DNS traffic and pass it without inspection:

```bash
sudo nmap <TARGET> -p <PORT> -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
```

Port opens → confirm with ncat using the same source port:

```bash
ncat -nv --source-port 53 <TARGET> <PORT>
```

> [!tip] If `--source-port 53` bypasses the firewall, IDS/IPS rules on that port are probably also weaker — worth interacting with the service directly via ncat before switching to louder tools.

> [!warning] Decoys with `RND` generate random IPs that may not be alive — SYN-flood mitigations can block the scan. Use known live IPs as decoys (`-D <IP1>,<IP2>,ME`) on sensitive targets.

IDS detection heuristic: if your scanning IP gets blocked mid-engagement, IPS is present. Switch to a new VPS/source and scan quieter. `-T1`/`-T2` + decoys + `--source-port 53` is the quiet stack.