# Enumeration

## Enumeration

Most of what actually helps comes from misconfigurations or neglected security on a service. When scanning, look for two things: functions/resources that let you interact with the target, and information that leads to further information.

Tools are secondary — knowing how to interact with the service and what's relevant is what matters. Manual enumeration is critical: scanning tools use timeouts, and a port marked closed/filtered because of a slow reply can hide the real entry point. Confirm tool results by hand.

## Introduction to Nmap

Covers host discovery, port scanning, service/version enumeration, OS detection, and scripted interaction (NSE).

```bash
nmap <scan types> <options> <target>
```

Scan techniques:

```text
-sS/sT/sA/sW/sM   TCP SYN/Connect/ACK/Window/Maimon
-sU               UDP
-sN/sF/sX         TCP Null, FIN, Xmas
--scanflags       custom TCP flags
-sI <zombie>      idle scan
-sY/sZ            SCTP INIT/COOKIE-ECHO
-sO               IP protocol
-b <relay>        FTP bounce
```

TCP-SYN (`-sS`) is the default when run as root and sends only the SYN — never completes the handshake:

- SYN-ACK back → open
- RST back → closed
- no packet back → filtered (firewall dropped it)

## Host Discovery

```bash
sudo nmap <TARGET_NETWORK>/24 -sn -oA tnet | grep for | cut -d" " -f5
```

`-sn` disables port scanning (host discovery only), `-oA` saves all output formats.

Hosts from a list:

```bash
sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5
```

Multiple IPs, or an octet range:

```bash
sudo nmap -sn -oA tnet <TARGET_IP> <TARGET_IP> <TARGET_IP>
sudo nmap -sn -oA tnet <TARGET_IP_RANGE>
```

Single host, plus why it was marked up:

```bash
sudo nmap <TARGET_IP> -sn -oA host
sudo nmap <TARGET_IP> -sn -oA host -PE --packet-trace
sudo nmap <TARGET_IP> -sn -oA host -PE --reason
sudo nmap <TARGET_IP> -sn -oA host -PE --packet-trace --disable-arp-ping
```

On a local segment nmap sends an ARP request *before* any ICMP echo, so `-PE` alone doesn't prove ICMP got through — add `--disable-arp-ping` to force the echo request. `--reason` shows the actual answer (`arp-response`, `echo-reply`). A host that doesn't respond may just be filtering ICMP, not offline.

## Host and Port Scanning

Target picture: open ports + services, versions, info the services leak, OS.

Port states worth knowing:

- **open** — connection established
- **closed** — RST received (also confirms the host is alive)
- **filtered** — no reply, or an error code came back
- **unfiltered** — only from ACK scan: reachable, but open/closed can't be determined
- **open|filtered** — no reply, possibly firewalled
- **closed|filtered** — only from IP ID idle scan

### Discovering Open TCP Ports

Default is the top 1000 TCP ports with `-sS` as root, `-sT` otherwise. Port selection: `-p 22,25,80,139,445`, `-p 22-445`, `--top-ports=10`, `-p-` (all), `-F` (top 100).

```bash
sudo nmap <TARGET_IP> --top-ports=10
sudo nmap <TARGET_IP> -p 21 --packet-trace -Pn -n --disable-arp-ping
```

The trace shows `RA` (RST+ACK) coming back → port closed.

### Connect Scan

```bash
sudo nmap <TARGET_IP> -p 443 --packet-trace --disable-arp-ping -Pn -n --reason -sT
```

Full three-way handshake — accurate, but it creates logs and is easily caught by IDS/IPS. Behaves like a normal client connection, so it's the least likely to break or destabilize a service.

### Filtered Ports

```bash
sudo nmap <TARGET_IP> -p 139 --packet-trace -n --disable-arp-ping -Pn
sudo nmap <TARGET_IP> -p 445 --packet-trace -n --disable-arp-ping -Pn
```

Dropped packets → silence, and nmap retries (`--max-retries` defaults to 10), so the scan drags on. Rejected packets → explicit answer, here ICMP type 3 code 3 (port unreachable), and the scan finishes fast. Either way the port deserves a second look later.

### Discovering Open UDP Ports

```bash
sudo nmap <TARGET_IP> -F -sU
sudo nmap <TARGET_IP> -sU -Pn -n --disable-arp-ping --packet-trace -p 137 --reason
```

Stateless, no handshake → much slower than TCP. Nmap sends empty datagrams, so you only get a reply if the application bothers to answer. ICMP type 3 code 3 → closed; nothing back → open|filtered.

### Version Scan

```bash
sudo nmap <TARGET_IP> -p 445 --reason -sV
```

## Saving the Results

Normal `-oN` (.nmap), grepable `-oG` (.gnmap), XML `-oX` (.xml), `-oA` for all three.

```bash
sudo nmap <TARGET_IP> -p- -oA target
xsltproc target.xml -o target.html
```

Save every scan — different tools give different results, and comparisons later are worth it. The HTML conversion is what you hand over in reporting.

## Service Enumeration

Quick port scan first (less traffic, less chance of getting blocked), then a full `-p-` version scan in the background.

```bash
sudo nmap <TARGET_IP> -p- -sV
sudo nmap <TARGET_IP> -p- -sV --stats-every=5s
sudo nmap <TARGET_IP> -p- -sV -v
```

Space bar prints progress on a running scan; `-v`/`-vv` prints open ports as they're discovered.

Banner grabbing — nmap can truncate what the service actually sent:

```bash
nc -nv <TARGET_IP> 25
sudo tcpdump -i eth0 host <ATTACKER_IP> and <TARGET_IP>
```

Here the banner reveals the distro (`220 inlane ESMTP Postfix (Ubuntu)`) which nmap's output dropped. Services can also hide or manipulate banners, so connect manually when something looks off.

## Nmap Scripting Engine

Lua scripts in 14 categories: auth, broadcast, brute, default (`-sC`), discovery, dos, exploit, external, fuzzer, intrusive, malware, safe, version, vuln.

```bash
sudo nmap <TARGET_IP> -sC
sudo nmap <TARGET_IP> --script <category>
sudo nmap <TARGET_IP> --script <script-name>,<script-name>
sudo nmap <TARGET_IP> -p 25 --script banner,smtp-commands
```

`smtp-commands` lists the commands the server supports (VRFY etc.) — that's a route into user enumeration.

`-A` = `-sV` + `-O` + `--traceroute` + `-sC`:

```bash
sudo nmap <TARGET_IP> -p 80 -A
```

Vuln assessment:

```bash
sudo nmap <TARGET_IP> -p 80 -sV --script vuln
```

`http-enum` surfaces interesting paths, `http-wordpress-users` finds usernames, `vulners` maps the detected version to CVEs.

## Performance

```bash
sudo nmap <TARGET_NETWORK>/24 -F
sudo nmap <TARGET_NETWORK>/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms
sudo nmap <TARGET_NETWORK>/24 -F --max-retries 0
sudo nmap <TARGET_NETWORK>/24 -F -oN tnet.minrate300 --min-rate 300
sudo nmap <TARGET_NETWORK>/24 -F -oN tnet.T5 -T 5
```

Timing templates: `-T 0` paranoid, `1` sneaky, `2` polite, `3` normal (default), `4` aggressive, `5` insane.

Speed costs accuracy — in the module's comparison the tightened RTT timeout lost 2 hosts and `--max-retries 0` lost 2 ports. On a whitelisted white-box test, `--min-rate` gives the big gain without dropping results.

https://nmap.org/book/man-performance.html

## Firewall and IDS/IPS Evasion

Filtered means dropped or rejected. Rejected packets answer explicitly: TCP RST, or ICMP errors — net unreachable, net prohibited, host unreachable, host prohibited, port unreachable, proto unreachable.

The ACK scan (`-sA`) slips past firewalls better than SYN/connect because the firewall can't tell whether the connection was originally established from inside:

```bash
sudo nmap <TARGET_IP> -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace
sudo nmap <TARGET_IP> -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

SYN scan: SA → open, ICMP unreachable → filtered, silence → filtered. ACK scan: RST → unfiltered (reachable, state unknown), silence → filtered. Compare the two and you know whether you're looking at a drop rule or a reject rule.

### Detect IDS/IPS

Scan from a VPS; if it stops reaching the target network, an admin took action — switch to another VPS and go quieter. Aggressively scanning one port is a cheap way to provoke a response and find out whether monitoring exists at all.

### Decoys

```bash
sudo nmap <TARGET_IP> -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
```

Random source IPs with yours dropped in among them. Decoys must be alive — otherwise SYN-flood protection on the target can lock you out. Spoofed packets are often filtered by ISPs and routers anyway. Works for SYN, ACK, ICMP scans and OS detection.

Test whether a filter is subnet-specific by changing source IP:

```bash
sudo nmap <TARGET_IP> -n -Pn -p 445 -O
sudo nmap <TARGET_IP> -n -Pn -p 445 -O -S <SOURCE_IP> -e tun0
```

### DNS Proxying

Reverse DNS queries usually pass firewalls (UDP 53, increasingly TCP 53). `--dns-server <ns>,<ns>` lets you use the internal DNS servers in a DMZ. `--source-port 53` makes your scan packets look like DNS traffic:

```bash
sudo nmap <TARGET_IP> -p 50000 -sS -Pn -n --disable-arp-ping --packet-trace
sudo nmap <TARGET_IP> -p 50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
ncat -nv --source-port 53 <TARGET_IP> 50000
```

If the firewall trusts port 53 through, IDS/IPS rules on that port are usually much weaker too.