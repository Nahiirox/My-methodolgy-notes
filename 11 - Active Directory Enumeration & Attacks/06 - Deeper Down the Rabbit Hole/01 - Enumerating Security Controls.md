# Enumerating Security Controls

Check Defender status — `RealTimeProtectionEnabled: True` means PowerView and most offensive PS will get flagged:

```powershell
Get-MpComputerStatus
```

Enumerate AppLocker rules — look for what's blocked and what paths are missed:

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

> [!tip] AppLocker commonly blocks `%SYSTEM32%\WindowsPowerShell\v1.0\powershell.exe` but misses `%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe` and `PowerShell_ISE.exe`.

Check PS language mode — `ConstrainedLanguage` breaks most PowerView functionality:

```powershell
$ExecutionContext.SessionState.LanguageMode
```

---

## LAPS Enumeration (LAPSToolkit)

Who can read LAPS passwords — delegated groups per OU:

```powershell
Find-LAPSDelegatedGroups
```

Users with All Extended Rights on LAPS hosts — often less scrutinized than delegated groups:

```powershell
Find-AdmPwdExtendedRights
```

Read cleartext LAPS passwords if your current user has access:

```powershell
Get-LAPSComputers
```

> [!tip] If your compromised account shows up in `Find-AdmPwdExtendedRights` output, you can pull local admin creds for that host directly — no cracking needed.