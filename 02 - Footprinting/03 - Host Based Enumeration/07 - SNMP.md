# SNMP — Footprinting

Ports: UDP/161 (queries), UDP/162 (traps). SNMPv1/v2c = no auth, community string in cleartext. SNMPv3 = auth + encryption. `public` and `private` are the default community strings — always try them first.

`rwuser noauth` or `rwcommunity public` in config = full read-write OID tree with no auth = critical.

---

### Community String Brute Force

Unknown community string → brute with onesixtyone before anything else:

```bash
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt <TARGET>
```

---

### Enumerate Once You Have a Community String

Full OID walk — dumps OS info, contact email, hostname, installed packages, running processes, network interfaces:

```bash
snmpwalk -v2c -c <COMMUNITY> <TARGET>
```

Fast OID brute force — faster than snmpwalk on large trees:

```bash
braa <COMMUNITY>@<TARGET>:.1.3.6.*
```

---

### What snmpwalk Leaks

From a single walk with `public`: OS version + kernel, admin contact email, hostname, system location, boot parameters, installed package list with versions, running process list.

Admin contact email → username format. Installed packages → software versions → CVE lookup. Boot params → disk UUIDs, kernel flags.

> [!tip] snmpwalk output is noisy. Pipe to grep for the quick wins: `snmpwalk -v2c -c public <TARGET> | grep -E "STRING|INTEGER" | grep -i "user\|pass\|mail\|contact\|name\|host"` to surface high-value fields fast.

> [!warning] SNMPv1/v2c community strings travel in cleartext over UDP — interceptable on LAN. If you capture SNMP traffic during a MITM position, community strings come out in the clear. Wireshark filter: `udp.port == 161`.