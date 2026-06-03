# File Transfer Detection — User Agent Signatures

Know what string your method leaves — defenders hunt these in SIEM/proxy logs.

`Invoke-WebRequest` / `Invoke-RestMethod`:

```
Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.14393.0
```

`WinHttp.WinHttpRequest.5.1` (COM object):

```
Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)
```

`Msxml2.XMLHTTP` (COM object):

```
Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; Win64; x64; Trident/7.0; .NET4.0C; .NET4.0E)
```

`certutil`:

```
Microsoft-CryptoAPI/10.0
```

`BITS`:

```
Microsoft BITS/7.8
```

> [!tip] Override the user agent on any PS download to blend in — `Invoke-WebRequest -UserAgent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"`. Pick a string from a real browser for the target OS. Msxml2 and WinHttp COM objects can be spoofed the same way by setting the `UserAgent` property before `.send()`.

> [!warning] Certutil and BITS have hardcoded UA strings you can't override — `Microsoft-CryptoAPI` and `Microsoft BITS` are instant IOCs in any mature SOC. Avoid on evasive engagements.