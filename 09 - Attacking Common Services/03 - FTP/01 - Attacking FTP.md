# Attacking FTP

FTP listens on TCP/21. Default Nmap scripts catch anonymous login automatically.

Enumerate — `ftp-anon` script fires automatically with `-sC`, banner reveals version:

```bash
sudo nmap -sC -sV -p 21 <TARGET>
```

---

## Anonymous Auth

Try anonymous login first, always — no password needed:

```bash
ftp <TARGET>
# user: anonymous
# pass: (blank)
```

Inside FTP: `ls` / `cd` / `get <file>` / `mget *` / `put <file>`. If the `incoming` directory is writable, you can upload — check for web server overlap → potential RCE via path traversal.

---

## Brute Force

No anonymous access → brute with Medusa. Prefer spraying if lockout policy unknown:

```bash
medusa -u <USER> -P <WORDLIST> -h <TARGET> -M ftp
```

`-U` for user list, `-P` for password list, `-M ftp` specifies protocol.

---

## FTP Bounce Attack

Use an exposed FTP server to port scan internal hosts that aren't reachable directly. Nmap `-b` flag triggers it:

```bash
nmap -Pn -v -n -p<PORT> -b anonymous:password@<FTP_DMZ_IP> <INTERNAL_TARGET_IP>
```

> [!warning] Most modern FTP servers block bounce attacks by default. Only works on misconfigured servers. Worth trying if you have FTP access to a DMZ host and suspect internal targets behind it.