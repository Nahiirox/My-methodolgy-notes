# Credentialed Enumeration — Linux

Minimum required: cleartext creds, NTLM hash, or SYSTEM on a domain-joined host.

---

## CrackMapExec

Users with `badpwdcount` — filter before spraying:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users
```

Groups with member counts — flag Administrators, Domain Admins, privileged IT groups:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --groups
```

Who's logged on to a target host — `(Pwn3d!)` means your user is local admin there:

```bash
sudo crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --loggedon-users
```

Share enum — READ/WRITE access per share:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --shares
```

Spider a share — output JSON to `/tmp/cme_spider_plus/<ip>.json`, hunt for scripts/configs with creds:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> -M spider_plus --share '<SHARE>'
```

---

## SMBMap

Check access across shares:

```bash
smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP>
```

Recursive dir listing, dirs only:

```bash
smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP> -R '<SHARE>' --dir-only
```

---

## rpcclient

Enumerate all users + RIDs:

```bash
rpcclient -U "<USER>%<PASS>" <DC_IP>
# then:
enumdomusers
```

Pull full user detail by RID (hex):

```
queryuser <RID>
```

> [!tip] RID `0x1f4` (500) = built-in Administrator always. Use `queryuser` on interesting RIDs from `enumdomusers` output to pull account detail, bad password count, last logon, etc.

---

## Impacket

psexec.py — drops a service via ADMIN$, lands as SYSTEM, noisy:

```bash
psexec.py <DOMAIN>/<USER>:'<PASS>'@<TARGET_IP>
```

wmiexec.py — semi-interactive, no file drop, runs as the authed user not SYSTEM, generates event ID 4688 per command:

```bash
wmiexec.py <DOMAIN>/<USER>:'<PASS>'@<TARGET_IP>
```

> [!warning] psexec.py writes a randomly-named executable to ADMIN$ — louder than wmiexec. Use wmiexec for stealth, psexec when you need SYSTEM.

---

## Windapsearch

Domain Admins group members:

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u <USER>@<DOMAIN> -p <PASS> --da
```

All privileged users including nested group membership — good for finding over-privileged accounts:

```bash
python3 windapsearch.py --dc-ip <DC_IP> -u <USER>@<DOMAIN> -p <PASS> -PU
```

---

## BloodHound (Python ingestor)

Collect everything — outputs `<date>_users/groups/computers/domains.json`:

```bash
sudo bloodhound-python -u '<USER>' -p '<PASS>' -ns <DC_IP> -d <DOMAIN> -c all
```

Zip and upload to GUI:

```bash
zip -r <OUTFILE>.zip *.json
```

Start neo4j before launching GUI:

```bash
sudo neo4j start
bloodhound
```

> [!tip] First query to run after upload: **Find Shortest Paths To Domain Admins**. Immediately shows exploitable ACL/group/session chains without manual tracing.