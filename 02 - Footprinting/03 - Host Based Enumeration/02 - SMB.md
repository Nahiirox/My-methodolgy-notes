# SMB — Footprinting

Nmap on 139 and 445 — `smb2-security-mode` line is the key output, note signing status immediately:

```bash
sudo nmap <TARGET> -sV -sC -p139,445
```

`signing:False` in CME output or `signing enabled but not required` in Nmap → relay viable.

---

### Null Session Enum

List shares unauthenticated:

```bash
smbclient -N -L //<TARGET>
```

Check share permissions without creds:

```bash
smbmap -H <TARGET>
```

CME null session — also shows signing status and SMBv1:

```bash
crackmapexec smb <TARGET> --shares -u '' -p ''
```

Full automated null enum — OS info, users, groups, shares, password policy in one run:

```bash
./enum4linux-ng.py <TARGET> -A
```

---

### rpcclient — Manual RPC Enum

Null session:

```bash
rpcclient -U "" -N <TARGET>
```

Key commands inside:

```
srvinfo              # OS version, server type
enumdomains          # domains on the host
querydominfo         # domain name, user count, server role
netshareenumall      # all shares + paths
netsharegetinfo <SHARE>   # ACL detail on a specific share
enumdomusers         # user list + RIDs
queryuser <RID>      # full user detail — badpwdcount, last logon, description
querygroup <RID>     # group membership count
```

Brute RIDs 500–1100 to find all users — covers local + domain accounts:

```bash
for i in $(seq 500 1100); do rpcclient -N -U "" <TARGET> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo ""; done
```

Same output cleaner via Impacket:

```bash
samrdump.py <TARGET>
```

---

### Connect and Interact

Connect to a specific share (blank password = anonymous):

```bash
smbclient //<TARGET>/<SHARE>
```

Inside smbclient — recursive list, download, run local commands:

```
ls -R
get <FILE>
!ls
!cat <FILE>
```

> [!tip] `queryuser` on every RID from `enumdomusers` output — Description field leaks passwords in plaintext surprisingly often. `bad_password_count` tells you if an account is close to lockout before you spray.

> [!warning] enum4linux-ng `-A` runs everything including password policy pull. Read the policy output before doing anything else — `lockout_threshold: None` means no lockout, spray freely. `lockout_threshold: 3` means one spray attempt only.