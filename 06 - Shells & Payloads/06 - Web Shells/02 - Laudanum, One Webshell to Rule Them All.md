# Laudanum Web Shells

Pre-built web shells for asp, aspx, jsp, php and more. Already on Parrot/Kali at `/usr/share/laudanum/`. Copy before modifying — never edit the originals.

Copy the relevant shell and edit before uploading:

```bash
cp /usr/share/laudanum/<LANG>/shell.<EXT> /tmp/<OUTFILE>.<EXT>
```

Required edits before upload: add your IP to the `allowedIps` variable (ASPX: line 59). Also strip ASCII art and comments — those strings are signatured by AV/defenders.

Upload via whatever upload vector you found, note the path from the success response, then browse to it:

```
http://<TARGET>/files/<OUTFILE>.<EXT>
```

> [!warning] Some apps randomize filenames on upload or don't expose a public files directory — the reported save path in the success message is your only reliable indicator of where it landed. If the path isn't disclosed, brute-force the filename or check common upload dirs (`/uploads/`, `/files/`, `/images/`, `/temp/`).

> [!tip] Web shell is browser-based and unstable — use it immediately to upgrade to a reverse shell. Don't do post-ex through the browser UI.