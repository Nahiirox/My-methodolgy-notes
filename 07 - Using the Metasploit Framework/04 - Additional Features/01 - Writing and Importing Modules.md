# MSF — Importing & Installing Modules

Find MSF-compatible exploits on ExploitDB without a browser:

```bash
searchsploit nagios3
searchsploit -t Nagios3 --exclude=".py"
```

`.rb` extension = Ruby, likely MSF-compatible. Not all `.rb` files are valid MSF modules — some are standalone Ruby scripts.

Copy the `.rb` into the matching category subfolder — filename must be snake_case, no dashes:

```bash
cp ~/Downloads/9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
```

Load without restarting msfconsole:

```
msf6 > reload_all
msf6 > use exploit/unix/webapp/nagios3_command_injection
```

Or load a module path at launch:

```bash
msfconsole -m /usr/share/metasploit-framework/modules/
```

Custom modules also go in `~/.msf4/modules/` — same subfolder structure as `/usr/share/metasploit-framework/modules/`. Create the dirs if they don't exist.

> [!warning] Filename convention breaks silently — if `reload_all` runs without errors but `search` can't find the module, check the filename for dashes or camelCase. Rename to snake_case and reload again.