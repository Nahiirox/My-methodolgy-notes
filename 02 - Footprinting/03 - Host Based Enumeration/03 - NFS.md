# NFS — Footprinting

Ports: TCP/UDP 111 (RPC/portmapper) + TCP/UDP 2049 (NFS). NFSv4 only needs 2049.

Nmap — `rpcinfo` script fires on 111, lists all RPC services and their dynamic ports:

```bash
sudo nmap <TARGET> -p111,2049 -sV -sC
```

NFS NSE scripts — lists share contents, permissions, UIDs, and stats without mounting:

```bash
sudo nmap --script nfs* <TARGET> -sV -p111,2049
```

List exported shares:

```bash
showmount -e <TARGET>
```

---

### Mount and Inspect

```bash
mkdir target-NFS
sudo mount -t nfs <TARGET>:/ ./target-NFS/ -o nolock
ls -n ./target-NFS/mnt/nfs/
```

`ls -n` shows raw UIDs/GIDs — use these to create matching local users and read/write files the current user can't access.

Unmount when done:

```bash
sudo umount ./target-NFS
```

---

### Dangerous Config Flags

`no_root_squash` = files created by root keep UID 0 → if writable, create a SUID shell on the share as local root → execute it via SSH as a low-priv user → LPE.

`rw` + `insecure` + `no_root_squash` together = full write as root with no port restriction = critical finding.

`root_squash` (default/safe) = root on the client maps to anonymous UID on the share → can't own root-owned files even if you mount as root.

> [!tip] NFS LPE chain: SSH foothold as low-priv user → NFS share mounted with `no_root_squash` + `rw` → on attack host create local user matching UID from `ls -n` output → copy `/bin/bash` to share, `chmod u+s` → SSH back and execute the SUID binary → shell as that UID.

> [!warning] `ls -n` output is the key — `ls -l` shows usernames from your local `/etc/passwd` which may not match the server's. Always use `-n` to see the actual numeric UIDs before deciding what to impersonate.