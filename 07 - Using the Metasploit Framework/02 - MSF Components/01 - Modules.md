# MSF — Modules & Usage

Module path format: `<type>/<os>/<service>/<name>` — use index number from search to select, not the full path.

Search by CVE, type, platform, rank — combine filters:

```
msf6 > search type:exploit platform:windows cve:2021 rank:excellent microsoft
msf6 > search ms17_010
msf6 > search eternalromance type:exploit
```

Select by index, check options, run:

```
msf6 > use 0
msf6 > options
msf6 > info
msf6 > run
```

Set target/lhost permanently for the session — survives module switches:

```
msf6 > setg RHOSTS <TARGET>
msf6 > setg LHOST <LHOST>
```

`set` = current module only. `setg` = persists until MSF restarts. Use `setg` when you're hammering one target across multiple modules.

Check `Required: Yes` fields in `options` output — those must be set before `run`. Everything else is optional tuning.

> [!tip] `use -1` opens the last search result directly. `check` (if supported) tests vuln without firing the payload — run it first on production targets.