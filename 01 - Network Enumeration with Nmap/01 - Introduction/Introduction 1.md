# Enumeration — Nmap Fundamentals

Tools don't find attack paths — you do. Tools automate what you already know. If you're stuck, the gap is usually service knowledge, not missing a tool.

Timeout behavior matters: if a port doesn't respond within the scan window, Nmap marks it closed or filtered. Filtered you can work with. Closed gets skipped. Manual interaction with a "closed" service may still yield a path.

---

Nmap syntax:

```bash
nmap <SCAN_TYPES> <OPTIONS> <TARGET>
```

Scan technique flags — know these:

```
-sS   TCP SYN (default, half-open, fast, needs root)
-sT   TCP Connect (full handshake, no root needed, slower)
-sU   UDP
-sA   ACK (firewall rule mapping, not open/closed)
-sN/-sF/-sX   Null/FIN/Xmas (bypass some stateless filters)
-sI   Idle/zombie scan
-sO   IP protocol scan
-b    FTP bounce scan
```

SYN scan response logic: SYN-ACK → open / RST → closed / no response → filtered.

> [!tip] `-sS` requires root. On a box where you can't sudo, fall back to `-sT` — it completes the full handshake but works as an unprivileged user.