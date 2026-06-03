# Credential Hunting in Windows

Run LaZagne against all modules — dumps browser, chat, mail, memory, wifi, Credential Manager, LSA secrets:

```cmd
start LaZagne.exe all
```

findstr sweep across common config/script file types:

```cmd
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```

High-value locations to check manually or with findstr:

- SYSVOL share — GPP passwords, scripts with hardcoded creds
- `unattend.xml` — common on dev/build machines
- `web.config` on dev machines and IT shares
- AD user/computer description fields
- Files named `pass.txt`, `passwords.docx`, `passwords.xlsx` on shares and SharePoint
- KeePass `.kdbx` files — crack master password with keepass2john → john

Key terms to grep/search: `password`, `passphrase`, `creds`, `pwd`, `login`, `dbpassword`, `dbcredential`, `configuration`

> [!tip] Consider what the host is used for before hunting — IT admin workstation → WinSCP/RDP saved creds, VPN configs, scripts. Dev machine → web.config, .env files, Git history. Server → scheduled task creds, service account configs.