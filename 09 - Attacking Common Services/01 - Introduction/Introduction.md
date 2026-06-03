# Interacting with Common Services — SMB, Email, Databases

---

## SMB — Windows

Mount a share:

```cmd
net use n: \\<TARGET>\<SHARE> /user:<USER> <PASS>
```

Hunt files by name pattern:

```cmd
dir n:\*cred* /s /b
dir n:\*secret* /s /b
```

Hunt content inside files:

```cmd
findstr /s /i cred n:\*.*
```

PowerShell equivalents:

```powershell
New-PSDrive -Name "N" -Root "\\<TARGET>\<SHARE>" -PSProvider "FileSystem" -Credential $cred
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List
```

---

## SMB — Linux

Mount with cifs:

```bash
sudo mkdir /mnt/<SHARE>
sudo mount -t cifs -o username=<USER>,password=<PASS>,domain=. //<TARGET>/<SHARE> /mnt/<SHARE>
```

Hunt files / content:

```bash
find /mnt/<SHARE>/ -name *cred*
grep -rn /mnt/<SHARE>/ -ie cred
```

---

## Databases

MSSQL from Linux:

```bash
sqsh -S <TARGET> -U <USER> -P <PASS>
mssqlclient.py <USER>@<TARGET> -windows-auth
```

MSSQL from Windows:

```cmd
sqlcmd -S <TARGET> -U <USER> -P <PASS>
```

MySQL from Linux/Windows:

```bash
mysql -u <USER> -p<PASS> -h <TARGET>
```

GUI options: dbeaver (cross-platform), MySQL Workbench (MySQL), SSMS (Windows/MSSQL only).

---

## Key Strings to Hunt on Any Share

`cred`, `password`, `secret`, `key`, `users` — plus extensions: `.config`, `.xml`, `.ini`, `.ps1`, `.bat`, `.vbs`, `.sql`, `.conf`