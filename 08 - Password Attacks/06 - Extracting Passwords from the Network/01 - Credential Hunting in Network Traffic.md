# Credential Hunting in Network Traffic

Protocols worth capturing on sight: HTTP, FTP, SNMP, POP3, IMAP, SMTP, LDAP, old SMB, VNC — any unencrypted service leaks creds in cleartext.

Key Wireshark filters for cred hunting:

```
http.request.method == "POST"
http contains "passw"
ftp
ldap
snmp
tcp.port == 21
```

Follow a TCP stream for full session context: right-click packet → Follow → TCP Stream. Edit → Find Packet → string search for `passw`, `user`, `login` across all packets.

Run Pcredz against a pcap — auto-extracts FTP, HTTP Basic/NTLM/Forms, SNMP community strings, NTLMv1/v2, Kerberos AS-REQ hashes:

```bash
./Pcredz -f <FILE>.pcapng -t -v
```

Live capture instead of pcap:

```bash
./Pcredz -i <IFACE> -t -v
```

NTLMv2 hashes from pcap → hashcat mode 5600. Kerberos AS-REQ etype 23 → mode 18200. Both crack offline the same way as Responder captures.

> [!tip] SNMP community strings from Pcredz output = often reused as passwords on network devices and management interfaces. Try them immediately against SNMP-managed hosts and any web admin panels.