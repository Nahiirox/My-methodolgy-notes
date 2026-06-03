# SSH for Windows: plink.exe

LOTL option when you can't drop tools — if PuTTY is already on the host or reachable on a share, plink.exe gives you SSH tunneling from Windows with no install.

Dynamic port forward (SOCKS) via plink — same concept as `ssh -D`, different binary:

```cmd
plink -ssh -D 9050 <USER>@<PIVOT_IP>
```

This starts a SOCKS listener on `127.0.0.1:9050` on the Windows host, tunneled through SSH to the pivot.

To use the tunnel with GUI tools on Windows, pair with **Proxifier** — configure it to route traffic through `127.0.0.1:9050`, then launch `mstsc.exe` or any other tool normally and it routes through the proxy transparently.

> [!tip] First time connecting via plink will prompt to trust the host key — pass `y` non-interactively by piping: `echo y | plink -ssh -D 9050 <USER>@<PIVOT_IP>`. Useful for automated or non-interactive contexts.

> [!warning] Plink may trigger AV/EDR on modern hardened hosts. Check security posture before dropping it. If PuTTY was already installed by a sysadmin, the binary is less suspicious than one you transfer in.

Windows native SSH client (`ssh.exe`, available since late 2018) supports the same `-D` flag — prefer it over plink if it's present, it's built-in and less likely to flag.