# MSF — Plugins

Load a pre-installed plugin:

```
msf6 > load nessus
msf6 > load pentest
msf6 > nessus_help
```

Install a custom plugin — drop the `.rb` file in the plugins dir:

```bash
sudo cp <PLUGIN>.rb /usr/share/metasploit-framework/plugins/
```

Then `load <PLUGIN>` inside msfconsole. Fails silently if the filename doesn't match — check exact name in the directory first.

Worth knowing exist (pre-installed): `nessus`, `nexpose`, `openvas`, `sqlmap`, `wmap`, `incognito`, `mimikatz` (v1).

`pentest` plugin from DarkOperator adds useful batch commands — `multi_post`, `multi_cmd`, `sys_creds`, `app_creds` for running post modules across multiple sessions simultaneously. High value when you have many sessions open.

> [!tip] `load` adds new commands to the help menu — run `help` after loading a plugin to see what's new.