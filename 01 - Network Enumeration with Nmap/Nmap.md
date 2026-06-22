# Nmap — Host Discovery, Scanning & Evasion

Nmap docs: https://nmap.org/book/toc.html — NSE scripts: https://nmap.org/nsedoc/index.html

Always save every scan: `-oA <NAME>` dumps all three formats (.nmap, .gnmap, .xml). Convert XML to HTML: `xsltproc target.xml -o target.html`

---

## Host Discovery

Ping sweep a CIDR — no port scan, just alive check:

```bash
sudo nmap <CIDR> -sn -oA tnet
```

Scan from a host list:

```bash
sudo nmap -sn -oA tnet -iL hosts.lst
```

Force ICMP echo (instead of ARP which is default on LAN) — useful to confirm how host is detected:

```bash
sudo nmap <TARGET> -sn -PE --disable-arp-ping --packet-trace --reason
```

> [!tip] On the same LAN, Nmap defaults to ARP ping not ICMP. Use `--disable-arp-ping -PE` if you specifically need ICMP behavior or want to confirm the host responds to ICMP.

More host discovery strategies: https://nmap.org/book/host-discovery-strategies.html

---

## Port States

`open` = connection established. `closed` = RST received. `filtered` = no response or ICMP error (packet dropped). `unfiltered` = ACK scan only, reachable but open/closed unknown. `open|filtered` = no response, could be firewalled. `closed|filtered` = IP ID idle scan only.

---

## Port Scanning

Default SYN scan (requires root) — top 1000 TCP:

```bash
sudo nmap <TARGET>
```

All ports:

```bash
sudo nmap <TARGET> -p-
```

Specific ports / ranges / top N / fast 100:

```bash
sudo nmap <TARGET> -p 22,80,445
sudo nmap <TARGET> -p 22-445
sudo nmap <TARGET> --top-ports=10
sudo nmap <TARGET> -F
```

TCP Connect scan (full 3-way handshake, no root needed, more detectable, use through proxychains):

```bash
nmap <TARGET> -sT -p <PORT>
```

UDP scan (slow — stateless, no ACK, long timeouts):

```bash
sudo nmap <TARGET> -sU -F
```

UDP state logic: response received = open, ICMP type 3 code 3 = closed, no response = `open|filtered`.

Version detection — banner grab + signature matching:

```bash
sudo nmap <TARGET> -p- -sV
```

Show open ports as discovered (verbose), print progress every 5s:

```bash
sudo nmap <TARGET> -p- -sV -v --stats-every=5s
```

Aggressive scan — `-sV -O --traceroute -sC` combined:

```bash
sudo nmap <TARGET> -p <PORT> -A
```

---

## Grabbing Banners Manually

Nmap sometimes misses banner info. Grab manually and capture with tcpdump:

```bash
sudo tcpdump -i <IFACE> host <ATTACK_IP> and <TARGET>
nc -nv <TARGET> <PORT>
```

PSH flag in tcpdump = server sending data (banner). Contains more info than Nmap reports (e.g. OS from SMTP banner).

---

## NSE Scripts

Run default scripts:

```bash
sudo nmap <TARGET> -sC
```

Run by category:

```bash
sudo nmap <TARGET> --script <CATEGORY>
```

Run specific scripts:

```bash
sudo nmap <TARGET> -p 25 --script banner,smtp-commands
```

Vuln scan on a port:

```bash
sudo nmap <TARGET> -p 80 -sV --script vuln
```

Key categories: `auth`, `brute`, `default` (-sC), `discovery`, `exploit`, `safe`, `version`, `vuln`. NSE script reference: https://nmap.org/nsedoc/index.html

---

## Performance Tuning

Timing templates (0=paranoid → 5=insane, default is T3):

```bash
sudo nmap <CIDR> -F -T5
```

Reduce RTT timeout for faster sweeps (risk missing slower hosts):

```bash
sudo nmap <CIDR> -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms
```

Kill retries entirely (miss more, much faster):

```bash
sudo nmap <CIDR> -F --max-retries 0
```

Set minimum packet rate (whitebox / known bandwidth):

```bash
sudo nmap <CIDR> -F --min-rate 300
```

> [!warning] Aggressive RTT and retry reduction misses hosts. Use T4/T5 or `--max-retries 0` only when speed matters more than completeness. For CPTS exam, use T4 max unless explicitly testing evasion.

Timing template details: https://nmap.org/book/performance-timing-templates.html

---

## Firewall / IDS/IPS Evasion

ACK scan — harder to filter than SYN because firewalls often pass ACK (assumes established session). Returns `unfiltered` for open ports, `filtered` for blocked:

```bash
sudo nmap <TARGET> -p 21,22,25 -sA -Pn -n --disable-arp-ping
```

Decoys — spoof source IPs, real IP buried among fakes (decoys must be live hosts):

```bash
sudo nmap <TARGET> -p 80 -sS -D RND:5 -Pn -n --disable-arp-ping
```

Spoof source IP (pair with `-e <IFACE>` to route correctly):

```bash
sudo nmap <TARGET> -p 445 -O -S <SPOOF_IP> -e tun0 -Pn -n
```

Source port spoofing — many firewalls trust traffic from port 53 (DNS). If a port is filtered, try scanning from port 53:

```bash
sudo nmap <TARGET> -p 50000 -sS --source-port 53 -Pn -n
```

Then connect with ncat from the same source port to confirm:

```bash
ncat -nv --source-port 53 <TARGET> <PORT>
```

DNS proxying — specify internal DNS server for queries (useful from DMZ):

```bash
sudo nmap <TARGET> --dns-server <INTERNAL_DNS>
```

Use `--packet-trace` to verify what's actually being sent/received during evasion attempts.

More scan techniques: https://nmap.org/book/man-port-scanning-techniques.html