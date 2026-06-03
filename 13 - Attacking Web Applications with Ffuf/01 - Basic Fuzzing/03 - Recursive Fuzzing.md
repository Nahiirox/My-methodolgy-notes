# Recursive Fuzzing (ffuf)

Depth 1 = root dirs + their immediate subdirs only. Pick interesting hits from there and drill manually rather than letting recursion run deep:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://<TARGET>:<PORT>/FUZZ -recursion -recursion-depth 1 -e .php -v
```

`-e .php` doubles the wordlist (each entry tried bare and with extension). `-v` prints full URLs — without it you can't tell which subdir a file lives under.

> [!warning] No depth cap = runtime explodes on deep trees. Always set `-recursion-depth`, start at 1, go deeper manually on targets that matter.