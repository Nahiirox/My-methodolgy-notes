# Nmap

## Enumeration

The goal is not gaining access — it's identifying all the ways we _could_ attack the target. Tools alone don't get us there; it's about actively interacting with the individual services and knowing what to do with what they give us. When scanning, we're looking for exactly two things:

1. Functions and/or resources that let us interact with the target or provide additional information
2. Information that leads to even more important information for accessing the target

Most findings come from misconfigurations or neglected service security — firewalls, GPOs, and updates alone are often not enough.

Manual enumeration is critical. Scanning tools work with timeouts: a port that doesn't answer in time gets marked closed/filtered/unknown, and we simply never see a service that might have been our way in.

## Introduction to Nmap

```bash
nmap <scan types> <options> <target>
```

Five techniques: host discovery, port scanning, service enumeration/detection, OS detection, scriptable interaction (NSE). Available scan types (`nmap --help`): `-sS/sT/sA/sW/sM` (SYN/Connect/ACK/Window/Maimon), `-sU` (UDP), `-sN/sF/sX` (Null/FIN/Xmas), `--scanflags`, `-sI` (idle), `-sY/sZ` (SCTP), `-sO` (IP protocol), `-b` (FTP bounce).

TCP SYN scan (`-sS`) is the default when running as root (raw sockets) — fast, never completes the three-way handshake. SYN-ACK back = open, RST = closed, no response = filtered (firewall dropping/ignoring).

## Host Discovery

Most effective method: ICMP echo requests — only works if firewalls allow it. Always store every scan (`-oA`) for comparison, documentation, reporting.

```bash
# network range — list live hosts only
sudo nmap <TARGET_CIDR> -sn -oA tnet | grep for | cut -d" " -f5

# from a provided IP list
sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5

# multiple IPs / consecutive range in the last octet
sudo nmap -sn -oA tnet <TARGET_IP1> <TARGET_IP2> <TARGET_IP3> | grep for | cut -d" " -f5
sudo nmap -sn -oA tnet <TARGET_IP_BASE>-20 | grep for | cut -d" " -f5

# single host
sudo nmap <TARGET_IP> -sn -oA host
```

`-sn` = no port scan. Hosts that don't answer may just be dropping ICMP — Nmap marks them inactive.

On a local network, Nmap ARPs before sending anything ICMP. To actually test with ICMP echo requests:

```bash
sudo nmap <TARGET_IP> -sn -oA host -PE --packet-trace --disable-arp-ping
sudo nmap <TARGET_IP> -sn -oA host -PE --reason    # shows WHY a host counts as alive (e.g. arp-response)
```

## Host and Port Scanning

Six port states: `open`, `closed` (RST received — also proves host is alive), `filtered` (no response or error code), `unfiltered` (ACK scan only — reachable, state unknown), `open|filtered` (no response, firewall may protect it), `closed|filtered` (IP ID idle scans only).

Port selection: `-p 21,22,25` (list), `-p 22-445` (range), `--top-ports=10`, `-p-` (all), `-F` (top 100). Default scan = top 1000 TCP ports.

Reading a trace (`--packet-trace -Pn -n --disable-arp-ping` for a clean view):

* SYN out, `RA` back within \~50ms → port closed
* SYN out, silence, resend after \~1s → packets dropped by firewall → filtered (default `--max-retries` is 10)
* SYN out, ICMP `type=3/code=3` (port unreachable) → packets rejected by firewall → filtered, but host alive — look closer at that port later

Connect scan (`-sT`) completes the full handshake: accurate, but loud — logged on most systems, easily caught by IDS/IPS. Behaves like a normal client though, so it's gentle on the service itself. SYN scan (half-open) is the stealthier default.

UDP (`-sU`): stateless, no acks, long timeouts → slow. Nmap sends empty datagrams; a response only comes if the application is configured to send one. ICMP code 3 = closed; other ICMP responses or no response = `open|filtered`.

`-sV` adds version detection: reads banners, falls back to signature-based probing (e.g. SMBProgNeg probe against 445) — significantly longer scans.

## Saving the Results

Three formats: `-oN` (.nmap), `-oG` (.gnmap, grepable), `-oX` (.xml); `-oA target` writes all three. No full path given → saved in the current directory.

```bash
sudo nmap <TARGET_IP> -p- -oA target
xsltproc target.xml -o target.html    # XML → readable HTML report for documentation
```

## Service Enumeration

Exact versions → known vulns, matching exploits, source code review. Workflow: quick top-ports scan first (low traffic, avoids getting blocked), then full `-p-` in the background, then `-sV` against the found ports.

During long scans: `[Space bar]` prints status, `--stats-every=5s` does it periodically, `-v`/`-vv` shows ports the moment they're found.

Banner grabbing: after the handshake the server typically sends a banner (TCP packet with PSH+ACK). Nmap may drop details the banner actually contained — the SMTP banner `220 inlane ESMTP Postfix (Ubuntu)` reveals the distro, Nmap showed only "Postfix smtpd". Banners can also be removed/faked manually. Grab them yourself:

```bash
nc -nv <TARGET_IP> 25
sudo tcpdump -i eth0 host <YOUR_IP> and <TARGET_IP>   # watch the handshake + PSH-ACK banner
```

## Nmap Scripting Engine

Lua scripts in 14 categories: `auth`, `broadcast`, `brute`, `default` (used by `-sC`), `discovery`, `dos`, `exploit`, `external`, `fuzzer`, `intrusive`, `malware`, `safe`, `version`, `vuln`.

```bash
sudo nmap <TARGET_IP> -sC                                  # default scripts
sudo nmap <TARGET_IP> --script <category>
sudo nmap <TARGET_IP> --script <script1>,<script2>

sudo nmap <TARGET_IP> -p 25 --script banner,smtp-commands
```

`-A` (aggressive) = `-sV` + OS detection (`-O`) + traceroute + default scripts. On a web port it pulled Apache 2.4.29, WordPress 5.3.4 via `http-generator`, the page title, and Linux OS guesses.

Vuln assessment:

```bash
sudo nmap <TARGET_IP> -p 80 -sV --script vuln
```

`http-enum` fingerprints the app and interesting paths (`/wp-login.php`, `/readme.html`), `http-wordpress-users` enumerates usernames (raise `http-wordpress-users.limit` if it stops early), `vulners` matches CPE against CVEs with links.

## Performance

Matters on big networks or low bandwidth. Every speedup trades completeness:

```bash
# RTT tuning — 39s → 12s on a /24, but 2 hosts fewer
sudo nmap <TARGET_CIDR> -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms

# retries (default 10) — 0 skips silent ports entirely, loses results
sudo nmap <TARGET_CIDR> -F --max-retries 0

# min rate — 30s → 9s, same results (best when whitelisted and bandwidth is known)
sudo nmap <TARGET_CIDR> -F --min-rate 300

# timing templates
sudo nmap <TARGET_CIDR> -F -T 5
```

`-T 0`–`-T 5` = paranoid / sneaky / polite / normal (default) / aggressive / insane. Too aggressive → security systems block us from the traffic alone.

## Firewall and IDS/IPS Evasion

Dropped packets: no response at all, Nmap retries, scan drags. Rejected packets: explicit answer — RST for TCP, or ICMP errors (Net/Host Unreachable, Net/Host/Port/Proto Prohibited, Port Unreachable).

ACK scan (`-sA`) is harder for firewalls/IDS to filter than `-sS`/`-sT`: ACK-only packets are often passed because the firewall can't tell which side initiated the connection. Open ports then show as `unfiltered`:

```bash
sudo nmap <TARGET_IP> -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

IDS vs IPS: IDS reports to the admin, IPS blocks on its own. Detection: work from several VPS with different IPs — if one host suddenly loses access, security measures exist and we continue from another VPS, quieter. Aggressively scanning a single port/service can also trigger a reaction, which tells us monitoring is in place.

Decoys — disguise origin with random source IPs, real IP randomly mixed in. Decoys must be alive, otherwise SYN-flood protections on the target may make the service unreachable:

```bash
sudo nmap <TARGET_IP> -p 80 -sS -Pn -f --disable-arp-ping --packet-trace -D RND:5
```

Spoofed packets often get filtered by ISPs/routers — alternatively use own VPS IPs combined with IP ID manipulation. If only certain subnets are blocked from a service, set the source IP manually:

```bash
sudo nmap <TARGET_IP> -n -Pn -p 445 -O -S <SOURCE_IP> -e tun0
```

DNS: queries (UDP 53) are usually passed. In a DMZ, `--dns-server <ns>,<ns>` lets us use the company's more-trusted internal resolvers. TCP 53 as source port is often trusted and weakly filtered by IDS/IPS:

```bash
sudo nmap <TARGET_IP> -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53

# once confirmed open, interact through the same trick
ncat -nv --source-port 53 <TARGET_IP> <PORT>
```

Example outcome: port 50000 filtered without it, open with `--source-port 53`, and ncat returned the `220 ProFTPd` banner.
