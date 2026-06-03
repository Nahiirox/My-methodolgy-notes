# Anatomy of a Shell

native terminal emulators: Windows Terminal, PuTTY, kitty, Alacritty, xterm, GNOME, MATE, Konsole, iTerm2

MITRE ATT&CK: Execution

`$` prompt → Bash, Ksh, POSIX, others

ID shell via process list

```
ps
```

ID shell via env vars

```
env
```

ID shell via command errors → shell identifies unrecognized input

interp knowledge → exploit success → sess gain