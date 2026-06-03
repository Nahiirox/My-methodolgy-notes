# Antak Webshell

ASPX-based PS shell from the Nishang project. Executes each command as a new process, runs scripts in memory, can encode/execute commands. Windows IIS targets only.

Located at `/usr/share/nishang/Antak-WebShell/antak.aspx`.

Copy and edit before uploading — two required changes, both on line 14:

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx /tmp/<OUTFILE>.aspx
```

Set a username and password on line 14 — without this anyone who finds the URL has access. Also strip ASCII art and comments before uploading — both are AV/defender signatures.

Upload via the target's file upload function, navigate to `/<upload_dir>/<OUTFILE>.aspx`, authenticate with your set creds.

From the shell: run PS commands directly, upload/download files via the built-in buttons, use `Encode and Execute` to run obfuscated scripts in memory.

> [!tip] Use `Encode and Execute` to load a reverse shell payload — it base64-encodes the PS command before sending, bypassing basic string-match AV detection in the web request.

> [!warning] Same instability caveat as all web shells — use Antak to immediately drop a proper reverse shell callback. Don't rely on it for sustained post-ex work.