# Value Fuzzing

No wordlist for numeric IDs — generate one:

```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

FUZZ goes on the value side, not the param name:

```bash
ffuf -w ids.txt:FUZZ -u http://<DOMAIN>:<PORT>/<path> -X POST -d '<PARAM>=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs <BASELINE_SIZE>
```

Confirm hit and retrieve content with curl:

```bash
curl http://<DOMAIN>:<PORT>/<path> -X POST -d '<PARAM>=<VALUE>' -H 'Content-Type: application/x-www-form-urlencoded'
```

> [!tip] Unknown value format — check what the param name implies. `id` → sequential int. `token` → pull from source/JS. `username` → use a names wordlist. Don't default to numeric if the type doesn't fit.