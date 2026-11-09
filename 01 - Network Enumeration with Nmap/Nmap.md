## Enumeration

Collecting as much information as possible to find every way to attack a target.

Two kinds of findings give access: functions or resources that let us interact or give extra information, and information that leads to more important information. Most come from misconfiguration.

Manual enumeration is critical. Tools mark slow ports closed, filtered, or unknown, and Nmap does not show closed ports, so a service can be missed.

## Introduction to Nmap

Open-source network scanner. Finds hosts, services with names and versions, and operating systems, and checks packet filters, firewalls, and IDS.

### Syntax

```bash
nmap <SCAN_TYPES> <OPTIONS> <TARGET_IP>
```

### Scan Techniques

`-sS` TCP SYN | `-sT` TCP Connect | `-sA` TCP ACK | `-sW` TCP Window | `-sM` TCP Maimon | `-sU` UDP | `-sN` TCP Null | `-sF` TCP FIN | `-sX` TCP Xmas | `--scanflags <FLAGS>` custom TCP flags | `-sI <ZOMBIE_HOST>` idle scan | `-sY` SCTP INIT | `-sZ` SCTP COOKIE-ECHO | `-sO` IP protocol scan | `-b <FTP_RELAY_HOST>` FTP bounce scan

SYN scan (`-sS`) is a default unless another scan is set and one of the most popular. It scans thousands of ports per second and never completes the three-way handshake.

```bash
sudo nmap -sS <TARGET_IP>
```

`SYN-ACK` response, port open | `RST` response, port closed | no response, port filtered (packets dropped or ignored by firewall)

## Host Discovery

Finds which hosts are online in a network. ICMP echo requests are the most effective method.

Works only if host firewalls allow it, hosts that ignore echo requests are marked inactive. Other techniques are in Firewall and IDS/IPS Evasion. Save every scan for comparison, documentation, and reporting.

### Scan Network Range

```bash
sudo nmap <TARGET_RANGE> -sn -oA <OUTPUT_FILE> | grep for | cut -d" " -f5 # live host IPs only
```

`-sn` disables port scan | `-oA` saves all output formats

### Scan IP List

Reads targets from a provided list.

```bash
sudo nmap -sn -oA <OUTPUT_FILE> -iL <FILE> | grep for | cut -d" " -f5
```

`-iL` read targets from list file

### Scan Multiple IPs

```bash
sudo nmap -sn -oA <OUTPUT_FILE> <TARGET_IP> <TARGET_IP_2> <TARGET_IP_3>| grep for | cut -d" " -f5 # list of IPs
sudo nmap -sn -oA <OUTPUT_FILE> <TARGET_IP>-<LAST_OCTET>| grep for | cut -d" " -f5 # range in last octet
```

### Scan Single IP

Check that a host is alive before scanning its ports.

```bash
sudo nmap <TARGET_IP> -sn -oA <OUTPUT_FILE> # alive check
sudo nmap <TARGET_IP> -sn -oA <OUTPUT_FILE> -PE --packet-trace # force ICMP echo, show packets
sudo nmap <TARGET_IP> -sn -oA <OUTPUT_FILE> -PE --reason # why host is up
sudo nmap <TARGET_IP> -sn -oA <OUTPUT_FILE> -PE --packet-trace --disable-arp-ping # ICMP echo without ARP
```

`-PE` ICMP echo request ping | `--packet-trace` shows all packets sent and received | `--reason` shows reason for the result | `--disable-arp-ping` disables ARP ping

On a local network Nmap sends an ARP ping first, so no ICMP echo is sent unless `--disable-arp-ping` is set.

Host discovery strategies: https://nmap.org/book/host-discovery-strategies.html

## Host and Port Scanning

Find open ports and services, service versions, information services provide, and the operating system of a live host.

`open` connection established (TCP, UDP, SCTP) | `closed` RST flag received, also shows host is alive | `filtered` no response or error code, open or closed unknown | `unfiltered` only in TCP ACK scan, accessible but open or closed unknown | `open|filtered` no response, firewall or packet filter may protect the port | `closed|filtered` only in IP ID idle scans, closed or filtered unknown

### Discovering Open TCP Ports

Default is SYN scan (`-sS`) on the top 1000 TCP ports when run as root, since raw packets need socket permissions. Otherwise Nmap runs a TCP connect scan (`-sT`).

`-p <PORT>,<PORT_2>` ports one by one | `-p <PORT_START>-<PORT_END>` range | `--top-ports=10` top ports from Nmap database | `-p-` all ports | `-F` top 100 ports

```bash
sudo nmap <TARGET_IP> --top-ports=10
```

#### Nmap - Trace the Packets

```bash
sudo nmap <TARGET_IP> -p <PORT> --packet-trace -Pn -n --disable-arp-ping
```

`-Pn` disables ICMP echo requests | `-n` disables DNS resolution

`SENT` with `S` is our SYN packet. `RCVD` with `RA` is RST and ACK from the target, port closed.

### Connect Scan

`-sT` completes the three-way handshake, so it is accurate but one of the least stealthy. It creates logs and is easily detected by IDS/IPS. It behaves like a normal client, so it has minimal impact on services. Use it when accuracy matters and the goal is to map without disrupting services.

SYN scan is half-open and more stealthy, but advanced IDS/IPS detect it too.

```bash
sudo nmap <TARGET_IP> -p <PORT> --packet-trace --disable-arp-ping -Pn -n --reason -sT
```

`SYN-ACK` response, port open | `RST` response, port closed

### Filtered Ports

Packets are either dropped or rejected by firewall rules.

```bash
sudo nmap <TARGET_IP> -p <PORT> --packet-trace -n --disable-arp-ping -Pn
```

Dropped: no response, Nmap resends the SYN (`--max-retries` default 10) and the scan takes much longer. Rejected: ICMP type 3 code 3 (port unreachable) returns, so the firewall is rejecting the port. If the host is known alive, look at that port later.

### Discovering Open UDP Ports

UDP is stateless with no handshake, so there is no acknowledgment and timeouts are long. `-sU` is much slower than a TCP scan. Admins sometimes forget to filter UDP.

```bash
sudo nmap <TARGET_IP> -F -sU # top 100 UDP ports
sudo nmap <TARGET_IP> -sU -Pn -n --disable-arp-ping --packet-trace -p <PORT> --reason # trace one UDP port
```

Nmap sends empty datagrams, so an open port answers only if the application is configured to.

`udp-response` port open | `port-unreach` ICMP type 3 code 3, port closed | `no-response` or any other ICMP response, port `open|filtered`

### Version Scan

Gets service names, versions, and details from open ports.

```bash
sudo nmap <TARGET_IP> -Pn -n --disable-arp-ping --packet-trace -p <PORT> --reason -sV
```

Port scanning techniques: https://nmap.org/book/man-port-scanning-techniques.html

## Saving the Results

Always save scan results to compare scans and tools and for documentation and reporting.

### Different Formats

`-oN` normal, .nmap | `-oG` grepable, .gnmap | `-oX` XML, .xml | `-oA` all formats

```bash
sudo nmap <TARGET_IP> -p- -oA <OUTPUT_FILE>
```

Without a full path, results save in the current directory.

### Style Sheets

Converts XML output to an HTML report, readable by non-technical people and useful for documentation.

```bash
xsltproc <OUTPUT_FILE>.xml -o <OUTPUT_FILE>.html
```

Output formats: https://nmap.org/book/output.html

## Service Enumeration

Find the exact application and version to search known vulnerabilities, source code, and a precise exploit for the service and OS.

### Service Version Detection

Run a quick port scan first, since less traffic lowers the chance of being detected and blocked. Run the full port scan (`-p-`) in the background, then version scan the specific ports with `-sV`.

```bash
sudo nmap <TARGET_IP> -p- -sV
sudo nmap <TARGET_IP> -p- -sV --stats-every=5s # status every 5 seconds
sudo nmap <TARGET_IP> -p- -sV -v # open ports shown when found
```

`[Space Bar]` shows scan status during a scan | `--stats-every` takes seconds `s` or minutes `m` | `-vv` more verbose

### Banner Grabbing

Nmap reads banners first. If it cannot identify the version from the banner, it falls back to signature matching, which makes the scan much longer. Servers often send a banner after the handshake, but banners can be removed or manipulated. Nmap can miss banner details, so compare with the raw banner.

```bash
sudo nmap <TARGET_IP> -p- -sV -Pn -n --disable-arp-ping --packet-trace
```

`NSOCK` READ SUCCESS lines in the trace show the full banner, which can reveal more than the summary, such as the OS.

#### Tcpdump

```bash
sudo tcpdump -i <INTERFACE> host <ATTACKER_IP> and <TARGET_IP>
```

#### Nc

```bash
nc -nv <TARGET_IP> <PORT>
```

#### Tcpdump - Intercepted Traffic

`[S]` SYN | `[S.]` SYN-ACK | `[.]` ACK | `[P.]` PSH-ACK, server sends data (banner)

## Nmap Scripting Engine

NSE runs Lua scripts that interact with services. Scripts are split into 14 categories.

`auth` determine credentials | `broadcast` host discovery by broadcast, found hosts added to remaining scans | `brute` brute-force login | `default` run by `-sC` | `discovery` evaluate accessible services | `dos` check DoS vulnerabilities, harms services | `exploit` exploit known vulnerabilities | `external` use external services | `fuzzer` send varied fields to find vulnerabilities, slow | `intrusive` can negatively affect target | `malware` check for malware infection | `safe` defensive, non-intrusive | `version` extends service detection | `vuln` identify specific vulnerabilities

### Default Scripts

```bash
sudo nmap <TARGET_IP> -sC
```

### Specific Scripts Category

```bash
sudo nmap <TARGET_IP> --script <CATEGORY>
```

### Defined Scripts

```bash
sudo nmap <TARGET_IP> --script <SCRIPT_NAME>,<SCRIPT_NAME_2>
sudo nmap <TARGET_IP> -p <PORT> --script banner,smtp-commands
```

`banner` shows service banner, can reveal the Linux distribution | `smtp-commands` lists commands the SMTP server accepts, may help find existing users

### Nmap - Aggressive Scan

```bash
sudo nmap <TARGET_IP> -p <PORT> -A
```

`-A` runs `-sV` service detection, `-O` OS detection, `--traceroute`, and `-sC` default scripts. Shows web server, web application and version, page title, and OS guess with percentage.

### Vulnerability Assessment

The `vuln` category checks the web server and application for versions and known vulnerabilities in databases.

```bash
sudo nmap <TARGET_IP> -p <PORT> -sV --script vuln
```

NSE scripts and categories: https://nmap.org/nsedoc/index.html

## Performance

Options that control scan speed on large networks or low bandwidth. Faster settings can miss hosts and ports.

`-T <0-5>` timing template | `--min-parallelism <NUMBER>` parallel probes | `--max-rtt-timeout <TIME>` max RTT timeout | `--min-rate <NUMBER>` packets sent per second | `--max-retries <NUMBER>` retries per port

### Timeouts

RTT is the time to get a response from a port. Nmap starts with a high timeout of 100ms (`--min-RTT-timeout`).

```bash
sudo nmap <TARGET_RANGE> -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms # shorter RTT timeouts
```

A too short initial RTT timeout can make Nmap overlook hosts.

### Max Retries

Default is 10. Setting 0 skips a port that does not answer.

```bash
sudo nmap <TARGET_RANGE> -F --max-retries 0 | grep "/tcp" | wc -l # count open ports
```

Fewer retries can miss open ports.

### Rates

Use when the bandwidth is known, for example white-box tests where we are whitelisted. Nmap tries to keep the set rate.

```bash
sudo nmap <TARGET_RANGE> -F -oN <OUTPUT_FILE> --min-rate 300
```

### Timing

Six templates set scan aggressiveness. Too aggressive can get us blocked by security systems.

`-T 0` paranoid | `-T 1` sneaky | `-T 2` polite | `-T 3` normal, default | `-T 4` aggressive | `-T 5` insane

```bash
sudo nmap <TARGET_RANGE> -F -oN <OUTPUT_FILE> -T 5
```

Options each template sets: https://nmap.org/book/performance-timing-templates.html

Performance: https://nmap.org/book/man-performance.html

## Firewall and IDS/IPS Evasion

Nmap options to bypass firewall rules and IDS/IPS.

### Firewalls

Monitors traffic and passes, drops, or blocks packets based on rules.

### IDS/IPS

IDS scans the network for attacks and reports them. IPS adds defensive measures when it detects an attack. Both match patterns and signatures, so a service detection scan can be blocked.

### Determine Firewalls and Their Rules

Dropped packets get no response. Rejected packets get an explicit response: RST for TCP, or an ICMP error such as Net Unreachable, Net Prohibited, Host Unreachable, Host Prohibited, Port Unreachable, Proto Unreachable.

ACK scan (`-sA`) is harder for firewalls and IDS/IPS to filter than SYN or Connect scans. It sends only the ACK flag and open or closed ports both answer with RST. Firewalls usually block incoming SYN but often pass ACK, since they cannot tell which side opened the connection.

```bash
sudo nmap <TARGET_IP> -p <PORT>,<PORT_2>,<PORT_3> -sS -Pn -n --disable-arp-ping --packet-trace # SYN scan
sudo nmap <TARGET_IP> -p <PORT>,<PORT_2>,<PORT_3> -sA -Pn -n --disable-arp-ping --packet-trace # ACK scan
```

`SA` reply to SYN, port open | `R` reply to ACK, port `unfiltered` | no reply, packets dropped, port `filtered`

### Detect IDS/IPS

Hard to detect because they are passive monitors. On detection the admin blocks the source IP and may contact the ISP.

Use several VPS with different IPs. Scan aggressively from one VPS. If it gets blocked, defensive measures exist, so continue from another VPS and scan quieter.

### Decoys

Use when subnets or regions are blocked or an IPS blocks us. Nmap inserts random IPs into the IP header to disguise the packet origin, and our real IP is placed randomly among them. Decoys work with SYN, ACK, ICMP scans, and OS detection.

Decoys must be alive, otherwise SYN-flood protection can make the target service unreachable. ISPs and routers often filter spoofed packets, so use our own VPS IPs with IP ID manipulation.

`-D RND:5` five random decoy IPs | `-S <SOURCE_IP>` set source IP, helps when only some subnets are denied | `-e <INTERFACE>` send through interface | `-O` OS detection

```bash
sudo nmap <TARGET_IP> -p <PORT> -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
sudo nmap <TARGET_IP> -n -Pn -p <PORT> -O # baseline, port filtered
sudo nmap <TARGET_IP> -n -Pn -p <PORT> -O -S <SOURCE_IP> -e <INTERFACE> # test other source IP
```

### DNS Proxying

Nmap does reverse DNS resolution by default. Firewalls usually pass DNS traffic (UDP 53, and TCP 53 for zone transfers and data over 512 bytes). Company DNS servers are more trusted than Internet ones, which helps in a DMZ to reach internal hosts.

`--dns-server <DNS_SERVER>,<DNS_SERVER_2>` set DNS servers | `--source-port 53` scan from DNS port

```bash
sudo nmap <TARGET_IP> -p <PORT> -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
```

A port shown filtered can show open from source port 53. That means the firewall trusts port 53 and IDS/IPS may be weaker there too, so connect with Netcat from the same source port.

```bash
ncat -nv --source-port 53 <TARGET_IP> <PORT>
```