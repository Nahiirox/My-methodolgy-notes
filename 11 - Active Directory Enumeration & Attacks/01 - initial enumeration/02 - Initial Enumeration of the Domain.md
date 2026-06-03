# Initial Domain Enumeration (Unauthenticated)

Passive first — listen before you touch anything.

Capture traffic to pull hosts from ARP/MDNS without sending a packet:

```bash
sudo tcpdump -i <IFACE> -w <OUTFILE>.pcap
```

Responder analyze mode — passive LLMNR/NBT-NS/MDNS listener, no poisoning:

```bash
sudo responder -I <IFACE> -A
```

ICMP sweep to confirm live hosts and build target list:

```bash
fping -asgq <CIDR>
```

Feed live IPs into Nmap — save all formats, always:

```bash
sudo nmap -v -A -iL hosts.txt -oA <OUTFILE>
```

DC fingerprint in nmap output: ports 53/88/389/636/3268/3269 open + LDAP banner leaks domain name + FQDN. RDP NTLM info block leaks NetBIOS name, domain, and OS version. Flag any host showing IIS 7.5 / Server 2008 / MSSQL 2008 → legacy, potential EternalBlue/MS08-067.

> [!warning] `smb2-security-mode: signing enabled but not required` → relay viable. Note it now.

---

## User Enumeration

Kerbrute userenum via Kerberos pre-auth — pre-auth failures don't trigger lockout or logs (mostly):

```bash
kerbrute userenum -d <DOMAIN> --dc <DC_IP> <WORDLIST> -o <OUTFILE>
```

Wordlists: jsmith.txt / jsmith2.txt from Insidetrust. Output feeds directly into spray list.

> [!warning] Failed Kerberos pre-auth still counts as a failed login — will lock accounts. Userenum is fine, but don't confuse it with a spray.

---

## SYSTEM on a domain-joined host = domain user equivalent

Paths to SYSTEM: EternalBlue/MS08-067/BlueKeep on legacy hosts, SeImpersonate abuse (Juicy Potato, pre-2019), local privesc 0-days, local admin + Psexec.

SYSTEM enables: AD enum (BloodHound/PowerView via computer account), Kerberoasting/ASREPRoasting, Net-NTLMv2 capture with Inveigh, token impersonation, ACL attacks.

No creds yet → LLMNR/NBT-NS poisoning or password spray next.