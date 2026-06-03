# Metasploit — Setup & Structure

Update modules before using — apt handles it, not msfupdate:

```bash
sudo apt update && sudo apt install metasploit-framework
```

Launch quiet (no banner, cleaner for tmux):

```bash
msfconsole -q
```

Custom or imported modules go here — drop them in the matching category subfolder:

```
/usr/share/metasploit-framework/modules/
├── auxiliary
├── exploits
├── payloads
├── post
├── encoders
├── evasion
└── nops
```

Plugins (load manually inside msfconsole with `load <plugin>`):

```
/usr/share/metasploit-framework/plugins/
```

> [!warning] Some MSF modules leave artifacts on the target (service installs, temp files, open ports). Know what your module does before running it — read the module info first: `info` inside msfconsole.