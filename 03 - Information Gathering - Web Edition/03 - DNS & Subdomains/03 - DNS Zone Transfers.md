## DNS Zone Transfers

Try AXFR before any brute force — misconfigured server gives you everything in one query. Query each NS server separately — different servers may have different `allow-transfer` configs.

Find NS servers first:

```bash
dig ns <DOMAIN>
```

Attempt AXFR against each NS:

```bash
dig axfr @<NS_SERVER> <DOMAIN>
```

Successful transfer dumps: all subdomains + IPs, all record types (A, MX, CNAME, SRV, PTR, TXT), internal hostnames, admin email from SOA.

If the main domain blocks it, try subzones — if `ANY` query or NS enum revealed `internal.<DOMAIN>` or similar:

```bash
dig axfr @<NS_SERVER> internal.<DOMAIN>
```

> [!tip] zonetransfer.me is a live intentionally-misconfigured domain you can use to test AXFR tooling and syntax without touching a target.

> [!warning] AXFR uses TCP/53. If TCP/53 is filtered but UDP/53 is open, zone transfer is already blocked at the network level — don't waste time. Move straight to brute force.