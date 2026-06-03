# Linux Remote Management — SSH, Rsync, R-Services

---

### SSH

Port TCP/22. Banner format: `SSH-2.0-OpenSSH_8.2p1` = SSH-2 only. `SSH-1.99-OpenSSH_3.9p1` = both versions supported (SSH-1 = MITM vulnerable).

Fingerprint version, weak ciphers, supported auth methods:

```bash
./ssh-audit.py <TARGET>
```

Check auth methods via verbose connect:

```bash
ssh -v <USER>@<TARGET>
```

Force password auth for brute — bypasses publickey preference:

```bash
ssh -v <USER>@<TARGET> -o PreferredAuthentications=password
```

Dangerous settings to flag: `PermitRootLogin yes` / `PermitEmptyPasswords yes` / `PasswordAuthentication yes` / `Protocol 1` / `PermitTunnel yes`.

---

### Rsync

Port TCP/873. Often requires no auth. Check for password reuse from other creds found elsewhere.

Scan:

```bash
sudo nmap -sV -p873 <TARGET>
```

List available shares — no creds needed if open:

```bash
nc -nv <TARGET> 873
#list
```

List share contents:

```bash
rsync -av --list-only rsync://<TARGET>/<SHARE>
```

Sync everything down:

```bash
rsync -av rsync://<TARGET>/<SHARE> ./
```

Rsync over SSH (non-standard port):

```bash
rsync -av -e "ssh -p <PORT>" <USER>@<TARGET>:<REMOTEPATH> ./
```

> [!tip] `.ssh` directory accessible via rsync → pull it down immediately. `authorized_keys`, `id_rsa`, `known_hosts` — all useful. `secrets.yaml` / `build.sh` / any config files → hunt for hardcoded creds.

---

### R-Services

Ports TCP/512 (rexec) / 513 (rlogin) / 514 (rsh/rcp). Unencrypted. Trust-based auth via `/etc/hosts.equiv` and `~/.rhosts` — if misconfigured, no password needed.

Scan:

```bash
sudo nmap -sV -p512,513,514 <TARGET>
```

Login without password if `.rhosts` trusts your IP:

```bash
rlogin <TARGET> -l <USER>
```

List active sessions on local network (UDP/513):

```bash
rwho
rusers -al <TARGET>
```

`.rhosts` entry `+ +` = any user from any host can authenticate without a password = instant access as that user.

> [!warning] R-services transmit everything in cleartext including passwords. If you see TCP/512-514 open on any host, check `.rhosts` and `/etc/hosts.equiv` for `+` wildcards before anything else — it's a free shell.