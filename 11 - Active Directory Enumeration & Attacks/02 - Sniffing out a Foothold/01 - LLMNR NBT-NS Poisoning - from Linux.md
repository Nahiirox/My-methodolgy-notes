# LLMNR/NBT-NS Poisoning — Linux

Prerequisite: SMB signing disabled or not required → relay viable. Otherwise, crack only.

Run in tmux, leave it while doing other enum — more time = more hashes:

```bash
sudo responder -I <IFACE>
```

Common additions: `-wf` — WPAD rogue proxy + OS fingerprinting. `-w` alone catches all HTTP from IE with auto-detect on (high value in large orgs). Avoid `-F`/`-P` unless needed — they can trigger login prompts.

> [!warning] NTLMv2 hashes **cannot** be used for PTH. Crack offline or relay — that's it.

Hashes land in `/usr/share/responder/logs/` — format `SMB-NTLMv2-SSP-<CLIENT_IP>.txt`.

Crack NTLMv2 (mode 5600):

```bash
hashcat -m 5600 <HASH_FILE> <WORDLIST>
```

Unknown hash type → check hashcat example hashes page to ID format and mode.

> [!tip] If ports 137/138/445/80/etc. are already bound on the attack host, Responder will fail silently on those modules. Check `Responder-Session.log` if captures aren't coming in.

Got cleartext → foothold. Still nothing → move to password spraying.