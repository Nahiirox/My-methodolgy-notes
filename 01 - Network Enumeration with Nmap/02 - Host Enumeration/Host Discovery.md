# Nmap — Host Discovery

Always save every scan. Different runs produce different results — you want the diff.

ICMP sweep across a subnet, save all formats:

```bash
sudo nmap <CIDR> -sn -oA <OUTFILE>
```

Scan from a host list:

```bash
sudo nmap -sn -oA <OUTFILE> -iL hosts.lst
```

> [!warning] Hosts that don't respond to ICMP aren't necessarily dead — firewall may be dropping pings. Don't remove them from scope yet.

On a local LAN, `-sn` sends ARP first (not ICMP) — ARP reply = host is up. Force ICMP instead:

```bash
sudo nmap <TARGET> -sn -PE --disable-arp-ping -oA <OUTFILE>
```

`--reason` tells you exactly why Nmap marked a host alive — useful when results look wrong:

```bash
sudo nmap <TARGET> -sn -PE --disable-arp-ping --reason
```

---

# Nmap — Port Scanning

Port states: `open` / `closed` (RST back) / `filtered` (no response or ICMP unreachable) / `unfiltered` (ACK scan only) / `open|filtered` (no response on UDP or after firewall drop) / `closed|filtered` (IP ID idle scan only).

Filtered = dropped (scan takes ~2s per port, two packets sent, no reply). vs. rejected = ICMP type 3 code 3 comes back fast. Both show as `filtered` but behavior differs under `--packet-trace`.

Scan specific port selection options: `-p 22,80,445` / `-p 22-445` / `--top-ports=10` / `-p-` (all) / `-F` (top 100).

Quick top-10 sanity check:

```bash
sudo nmap <TARGET> --top-ports=10
```

Clean SYN scan with no noise — use this when tracing behavior:

```bash
sudo nmap <TARGET> -p <PORT> --packet-trace -Pn -n --disable-arp-ping
```

UDP scan — slow, `open|filtered` means no response (normal), ICMP type 3 code 3 = closed:

```bash
sudo nmap <TARGET> -sU -F
```

Version detection — banner grab + signature match:

```bash
sudo nmap <TARGET> -p- -sV
```

> [!tip] `-sT` (full connect) doesn't need root and creates clean logs — use on hosts where you can't run as root or need to confirm `-sS` results that look wrong.

> [!warning] UDP scan with `-p-` will take forever. Stick to `-F` or target specific ports (53, 161, 500, 5353).

---

# Nmap — Saving Results

Always `-oA`. No exceptions.

```bash
sudo nmap <TARGET> -p- -oA <OUTFILE>
```

Produces `<OUTFILE>.nmap` (human) / `<OUTFILE>.gnmap` (greppable) / `<OUTFILE>.xml` (machine).

Convert XML to HTML for reporting:

```bash
xsltproc <OUTFILE>.xml -o <OUTFILE>.html
```

---

# Nmap — Service Enumeration

Workflow: fast scan first → identify open ports → version scan those ports only.

Fast scan to identify ports:

```bash
sudo nmap <TARGET> -p- --min-rate 300 -oA <OUTFILE>
```

Version scan on confirmed open ports:

```bash
sudo nmap <TARGET> -p <PORTS> -sV -oA <OUTFILE>
```

Watch scan progress live — `-v` prints ports as they're found:

```bash
sudo nmap <TARGET> -p- -sV -v
```

Or periodic status output:

```bash
sudo nmap <TARGET> -p- -sV --stats-every=5s
```

> [!tip] Nmap misses banner details sometimes. If `-sV` gives vague output, `nc -nv <TARGET> <PORT>` manually and read the banner yourself — SMTP/FTP/SSH banners often leak OS and exact version that Nmap doesn't surface cleanly.

---

# Nmap — NSE

Default scripts (`-sC`) on a port:

```bash
sudo nmap <TARGET> -p <PORT> -sC
```

Run specific scripts:

```bash
sudo nmap <TARGET> -p <PORT> --script <SCRIPT1>,<SCRIPT2>
```

Run a script category:

```bash
sudo nmap <TARGET> -p <PORT> --script <CATEGORY>
```

Aggressive mode — `-sV` + `-O` + traceroute + `-sC` in one flag:

```bash
sudo nmap <TARGET> -p <PORT> -A
```

Vuln scan against a known open port:

```bash
sudo nmap <TARGET> -p <PORT> -sV --script vuln
```

> [!tip] `--script vuln` on port 80 dumps WordPress users, CVEs, login pages, and exposed readmes automatically. Run it before manually poking a web service.

> [!warning] Categories `exploit`, `dos`, `intrusive` can crash services or trigger IDS. Don't run them without explicit scope approval. `safe`, `default`, `vuln` are fine for normal assessments.

Script categories worth knowing: `auth` / `brute` / `default` / `discovery` / `exploit` / `safe` / `version` / `vuln`.

---

# Nmap — Performance

Tuning levers: `-T<0-5>` / `--min-rate <N>` / `--max-retries <N>` / `--initial-rtt-timeout` / `--max-rtt-timeout`.

Whitelisted / internal scan — crank rate, keep retries:

```bash
sudo nmap <CIDR> -F --min-rate 300 -oA <OUTFILE>
```

Aggressive timing template (use on internal, not external):

```bash
sudo nmap <CIDR> -F -T4 -oA <OUTFILE>
```

Reduce retries to speed up at cost of accuracy:

```bash
sudo nmap <CIDR> -F --max-retries 0
```

> [!warning] `--max-retries 0` and tight RTT timeouts miss hosts. Missed hosts = missed attack surface. On external or slow targets, leave defaults or use `-T3`. Only tune aggressively when you know the network is stable and you're whitelisted.

Timing template reference: `T0` paranoid → `T3` normal (default) → `T4` aggressive → `T5` insane. `T4` is the practical exam-speed option without being reckless. `T5` risks missing results on unstable links.