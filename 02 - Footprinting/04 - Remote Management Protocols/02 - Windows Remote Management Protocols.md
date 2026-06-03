# Windows Remote Management — RDP, WinRM, WMI

---

### RDP

Port TCP/3389 (also UDP/3389). NLA = CredSSP auth before session starts — creds required upfront. NTLM info block leaks hostname, domain, OS version without auth.

Nmap — NLA status, encryption support, NTLM info:

```bash
nmap -sV -sC <TARGET> -p3389 --script rdp*
```

> [!warning] Nmap RDP scripts send `mstshash=nmap` cookie — visible in EDR/threat hunting. On hardened networks, use rdp-sec-check instead for quieter enumeration.

Security layer check — no auth needed, confirms NLA/TLS/legacy RDP support:

```bash
./rdp-sec-check.pl <TARGET>
```

Connect:

```bash
xfreerdp /u:<USER> /p:<PASS> /v:<TARGET>
```

Self-signed cert warning on connect = normal, accept it. CN in cert leaks internal hostname.

---

### WinRM

Ports TCP/5985 (HTTP) / 5986 (HTTPS). Enabled by default on Server 2012+. Requires explicit enable on older versions and clients.

Scan:

```bash
nmap -sV -sC <TARGET> -p5985,5986 --disable-arp-ping -n
```

Connect from Linux:

```bash
evil-winrm -i <TARGET> -u <USER> -p <PASS>
```

PTH with evil-winrm:

```bash
evil-winrm -i <TARGET> -u <USER> -H <NTHASH>
```

---

### WMI

Initiates on TCP/135, moves to random high port after. Semi-interactive shell, runs as the authed user (not SYSTEM), generates event ID 4688 per command.

Remote command execution:

```bash
wmiexec.py <USER>:<PASS>@<TARGET> "hostname"
```

> [!tip] RDP NTLM info block is the highest-yield Nmap output on Windows hosts — leaks NetBIOS name, domain name, and exact OS version (Product_Version) without any auth. Pull this before anything else on TCP/3389.

> [!warning] WinRM access = double hop problem for any AD enum that needs to reach a DC. Use PSCredential objects on every PowerView call, or use RDP as your jump point instead when available.