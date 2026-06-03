# Spawning Interactive Shells

No Python → try these in order, use whatever interpreter is present:

```bash
/bin/sh -i
```

```bash
perl -e 'exec "/bin/sh";'
```

```bash
ruby -e 'exec "/bin/sh"'
```

```bash
lua -e 'os.execute("/bin/sh")'
```

```bash
awk 'BEGIN {system("/bin/sh")}'
```

```bash
find . -exec /bin/sh \; -quit
```

VIM escape (if you're stuck in an editor session):

```bash
vim -c ':!/bin/sh'
# or from inside vim:
# :set shell=/bin/sh
# :shell
```

After landing any shell, check sudo rights immediately — `NOPASSWD: ALL` on a web service account is an instant privesc:

```bash
sudo -l
```

> [!tip] `sudo -l` requires a stable TTY to return output. If it hangs or errors, your shell isn't fully interactive yet — upgrade first with Python/pty, then retry.

> [!warning] `find -exec /bin/sh` only spawns a shell if `find` actually locates the specified file. `find . -exec /bin/sh \; -quit` sidesteps this by running on the current directory immediately.